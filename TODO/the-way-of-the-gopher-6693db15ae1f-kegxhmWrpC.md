* * *

# The Way of the Gopher

## Making the Switch from Node.js to Golang

I’ve dabbled in JavaScript since college, made a few web pages here and there
and while JS was always an enjoyable break from C or Java, I regarded it as a
fairly limited language, imbued with the special purpose of serving up
animations and pretty little things to make users go “ooh” and “aah”. It was
the first language I taught anyone who wanted to learn how to code because it
was simple enough to pick up and would quickly deliver tangible results to the
developer. Smash it together with some HTML and CSS and you have a web page.
Beginner programmers love that stuff.

Then something happened two years ago. At that time, I was in a researchy
position working mostly on server-side code and app prototypes for Android. It
wasn’t long before Node.js popped up on my radar. Backend JavaScript? Who
would take that seriously? At best, it seemed like a new attempt to make
server-side development easier at the cost of performance, scalability, etc.
Maybe it’s just my ingrained developer skepticism, but there’s always been
that alarm that goes off in my brain when I read about something being fast
and easy and production-level.

![](https://cdn-images-1.medium.com/max/1200/1*sssxLG43Vvq7O0skPXLmcw.gif)

![](https://cdn-images-1.medium.com/max/1200/1*ql1YI9gJRlFawtXywKxPpw.gif)

Then came the research, the testimonials, the tutorials, the side-projects and
6 months later I realized I had been doing nothing _but_ Node since I first
read about it. It was just too easy, especially since I was in the business of
prototyping new ideas every couple months. But Node wasn’t just for prototypes
and pet projects. Even big boy companies like Netflix had parts of their stack
running Node. Suddenly, the world was full of nails and I had found my hammer.

Fast forward another couple months and I’m at my current job as a backend
developer for [Digg](http://www.digg.com). When I joined, back in April of
2015, the stack at Digg was primarily Python with the exception of two
services written in, wait for it, Node. I was even more thrilled to be
assigned the task of reworking one of the services which had been causing
issues in our pipeline.

Our troublesome Node service had a fairly straightforward purpose. Digg uses
Amazon S3 for storage which is peachy, except S3 has no support for batch GET
operations. Rather than putting all the onus on our Python web server to
request up to 100+ keys at a time from S3, the decision was made to take
advantage of Node’s easy async code patterns and great concurrency handling.
And so Octo, the S3 content fetching service, was born.

Node Octo performed well except for when it didn’t. Once a day it needed to
handle a traffic spike where the requests per minute jump from 50 to 200+.
Also keep in mind that for each request, Octo typically fetches somewhere
between 10–100 keys from S3. That’s potentially 20,000 S3 GETs a minute. The
logs showed that our service slowed down substantially during these spikes,
but the trouble was it didn’t always recover. As such, we were stuck bouncing
our EC2 instances every couple weeks after Octo would seize up and fall flat
on its face.

The requests to the service also pass along a strict timeout value. After the
clock hits X number of milliseconds since receiving the request, Octo is
supposed to return to the client whatever it has successfully fetched from S3
and move on. However, even with a max timeout of 1200ms, in Octo’s worst
moments we had request handling times spiking up to 10 seconds.

The code was heavily asynchronous and we were caching S3 key values
aggressively. Octo was also running across 2 medium EC2 instances which we
bumped up to 4.

I reworked the code three times, digging deeper than ever into Node
optimizations, gotchas, and tricks for squeezing every last bit of performance
out of it. I reviewed benchmarks for popular Node webserver frameworks, like
Express or Hapi, vs. Node’s built-in HTTP module. I removed any third party
modules that, while nice to have, slowed down code execution. The result was
three, one-off iterations all suffering from the same issue. No matter how
hard I tried, I couldn’t get Octo to timeout properly and I couldn’t reduce
the slow down during request spikes.

A theory eventually emerged and it had to do with the way Node’s event loop
works. If you don’t know about the event loop, here’s some insight from [Node
Source](https://nodesource.com/blog/understanding-the-nodejs-event-loop/):

> Node’s “event loop” is central to being able to handle high throughput  
> scenarios. It is a magical place filled with unicorns and rainbows, and is
the  
> reason Node can essentially be “single threaded” while still allowing an  
> arbitrary number of operations to be handled in the background.

![](https://cdn-images-1.medium.com/max/1600/1*Txjoy_plOhkm8fiSZeBRlg.png)

Not-So Magic Event Loop Blocking (X-Axis: Time in milliseconds)

You can see when all the unicorns and rainbows went to hell and back again as
we bounced the service.

With event loop blocking as the biggest culprit on my list, it was just a
matter of figuring why it was getting so backed up in the first place.

Most developers have heard about Node’s non-blocking I/O model; it’s great
because it means all requests are handled asynchronously without blocking
execution, or incurring any overhead (like with threads and processes) and as
the developer you can be blissfully unaware what’s happening in the backend.
However, it’s always important to keep in mind that Node is single-threaded
which means none of your code runs in parallel. I/O may not block the server
but your code certainly does. If I call sleep for 5 seconds, my server will be
unresponsive during that time.

![](https://cdn-images-1.medium.com/max/1600/1*XJaACfD5MC98TezMqRJomg.png)

Visualizing the Event Loop —
[StrongLoop](https://strongloop.com/strongblog/node-js-performance-event-loop-
monitoring/)

And the non-blocking code? As requests are processed and events are triggered,
messages are queued along with their respective callback functions. To explain
further, here’s an excerpt from a particularly insightful [blog post from
Carbon Five](http://blog.carbonfive.com/2013/10/27/the-javascript-event-loop-
explained/):

> In a loop, the queue is polled for the next message (each poll referred to
as a “tick”) and when a message is encountered, the callback for that message
is executed. The calling of this callback function serves as the initial frame
in the call stack, and due to JavaScript being single-threaded, further
message polling and processing is halted pending the return of all calls on
the stack. Subsequent (synchronous) function calls add new call frames to the
stack...

Our Node service may have handled incoming requests like champ if all it
needed to do was return immediately available data. But instead it was waiting
on a ton of nested callbacks all dependent on responses from S3 (which can be
god awful slow at times). Consequently, when any request timeouts happened,
the event and its associated callback was put on an already overloaded message
queue. While the timeout event might occur at 1 second, the callback wasn’t
getting processed until all other messages currently on the queue, and their
corresponding callback code, were finished executing (potentially seconds
later). I can only imagine the state of our stack during the request spikes.
In fact, I didn’t need to imagine it. A little bit of CPU profiling gave us a
pretty vivid picture. Sorry for all the scrolling.

![](https://cdn-images-1.medium.com/max/1600/1*ijn5rx734gGfYprTsxDhpQ.png)

The flames of failure

As a quick intro to flame graphs, the y axis represents the number of frames
on the stack, where each function is the parent of the function above it. The
x axis has to do with the sample population more so than the passage of time.
It’s the width of the boxes which show the total time on-CPU; greater width
may indicate slower functions or it may simply mean that the function is
called more often. You can see in Octo’s flame graph the huge spikes in our
stack depth. More detailed info on profiling and flame graphs can be found
[here](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).

In light of these realizations, it was time to entertain the idea that maybe
Node.js wasn’t the perfect candidate for the job. My CTO and I sat down and
had a chat about our options. We certainly didn’t want to continue bouncing
Octo every other week and we were both very interested in a promising case
study that had cropped up on the internet:

[**Handling 1 Million Requests per Minute with Go**  
_Here at Malwarebytes we are experiencing phenomenal growth, and since I have
joined the company over 1 year ago in
the…_marcio.io](http://marcio.io/2015/07/handling-1-million-requests-per-
minute-with-golang/ "http://marcio.io/2015/07/handling-1-million-requests-per-
minute-with-golang/" )[](http://marcio.io/2015/07/handling-1-million-requests-
per-minute-with-golang/)

If the title wasn’t tantalizing enough, the topic was on creating a service
for making PUT requests to S3 (wow, other people have these problems too?). It
wasn’t the first time we had talked about using Golang somewhere in our stack
and now we had a perfect test subject.

Two weeks later, after my initial crash course introduction to Golang, we had
a brand new Octo service up and running. I modeled it closely after the
inspiring solution outlined in [Malwarebyte’s](https://www.malwarebytes.org/)
Golang article; the service has a worker pool and a delegator which passes off
incoming jobs to idle workers. Each worker runs on its own goroutine, and
returns to the pool once the job is done. Simple and effective. The immediate
results were pretty spectacular.

![](https://cdn-images-1.medium.com/max/1600/1*QDECNTMtxdwm92OYBnYvGg.png)

A nice simmer

Our average response time from the service was almost cut in half, our
timeouts (in the scenario that S3 was slow to respond) were happening on time,
and our traffic spikes had minimal effects on the service.

![](https://cdn-images-1.medium.com/max/1600/1*lm3f8I9LBEhmZSDnN72W7g.png)

Blue = Node.js Octo | Green = Golang Octo

With our Golang upgrade, we are easily able to handle 200 requests per minute
and 1.5 million S3 item fetches per day. And those 4 load-balanced instances
we were running Octo on initially? We’re now doing it with 2.

Since our transition to Golang we haven’t looked back. While the majority of
our stack is (and probably will always be) in Python, we’ve begun the process
of modularizing our code base and spinning up microservices to handle specific
roles in our system. Alongside Octo, we now have 3 other Golang services in
production which power our realtime message system and serve up important
metadata for our content. We’re also very proud of the newest addition to our
Golang codebase, [DiggBot](http://digg.com/diggbot).

This is not to say that Golang is a silver bullet for all our problems. We’re
careful to consider the needs of each of our services. As a company, we make
the effort to stay on top of new and emerging technologies and to always ask
ourselves, can we be doing this better? It’s a constantly evolving process and
one that takes careful research and planning.

I’m proud to say that this story has a happy ending as our Octo service has
been up and running for a couple months with great success (a few bug fixes
aside). For now, Digg is going the way of the Gopher.

![](https://cdn-images-1.medium.com/max/1600/1*NlSd8_zML3LljxAypRw64w.png)

<https://github.com/gengo/goship>

