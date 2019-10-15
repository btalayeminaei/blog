#  Writing Scalable Go Services

## Problem

We need to quickly load hotel rates from hundreds of hotels from multiple hotel
vendors. We often need to use different strategies to retrieve rates depending
on the vendor. Some vendors allow us to get avaialble rates for multiple hotels
in one call. Others require that we ask for hotel availbility one at a time. We
have a lot of bottleneck in the latter case because even if they respond quickly
to individual requests, the latency increases with every extra hotel requested.
This would not scale well if, in a hypothetical world, searched for rates from
thousands of hotels. Even if we don't scale to that level, we see an opportunity
to improve user experience and confidence by quickly returning rates.


## Go 

This will not be a tutorial on go because I am still learning the language
and would likely not do a thorough job. But I want to provide an explanation for
why we chose go to tackle this problem. If you have not heard, go is a new
language that is designed for modern concurrency.  Its concurrency primitives
(goroutine and channel) make creating concurrent programs simple and fun.
goroutines can be thought of as very lightweight and in-expenssive threads.
channels allow these goroutines to communicate with each other, sort of like a
pipe. So creating goroutines is cheap and we have a first-class primitive to
enable communication between these processes. But how does this enable us to
design a scalable go service? 

`select`: multiway concurrent control switch


## Communicating Sequential Processes

Go concurrency is based on Tony Hoare's CSP paper, a variant of Erlang's actor
model. CSP and Actor Model take a message passing approach to ensure data
integrity rather than locks and mutexes in traditional threading models. Go's
goroutines and channels mediate access to shared data by passing data references
between routines and help structure concurrent software.  I personally find this
mental model to be easier to reason about compared to traditional thread and
lock model. It also fits how things work in the real world. We want things to
execute concurrantly and independently and we want to enable them to report to
each other if need be. This decouples concurrency from parallelism. Concurreny
is about the composition of independently executing procesees and their
communication. Parrallelism is the simoultaneous execution of multiple
processes. I think our goal should be to compose programs so that they can
execute one or more cores efficiently wihtout needing it to run on a specific
number of cores. 

```go
SAMPLE CODE
```

> Do not communicate by sharing memory; instead, share memory by communicating.

## First Attempt

As explained earlier, due to certain and potential limitation in the vendor APIs
we use to get hotel rates, we wanted to come up with a solution that would lend
itself to parallel processing. Specifically, we wanted our vendor service to
take a list of hotel ids (potentially hundreds) and quickly gather the rates. We
were also aware of certain limitations in our vendor APIs, the most limiting
being the fact we could only request rates one hotel at a time. We decided that
we would get a lot of benefits from proper parallelizing this step.

We began by designing a solution that would use go's concurrent primitives to
parallelize rates gathering. For each request to gather rates for n number of
hotels, our program would break it up into n goroutines, each of which would
contact our vendor API to get rates. We use a buffered channel to gather
all results and then aggregate. This solution actually worked fine and it was a
great proof of concept. However, there were some uncerntainties around
scalability, maintainability, and configurability.

```go
SAMPLE CODE
```

Since goroutines are cheap, go is able to spawn thousands of goroutines during
its execution. But even then, we did not like that at any given moment we could
be spawning 100s of routines. More concerning was that the request would have
such concequences for the internal resources of the application. Even if we were
fine with that, we could easily spawn thousands of goroutines under heavy load.
100 users searching for 100 hotels each would mean 10,000 goroutines! Something
else that is lacking in this approach is that it does not allow us to configure
how parallel we want to be. Finally, unless you are familiar with go, we felt
that this approach did not clearly communicate the model we wanted to share for
solving the problem in the future.

## The Worker Model

With the proof of concept in the bag, we decided to design a more robust
concurrent model to tackle some of our doubts regarding scalability,
configuratbility, and communication. We went to the white board and started
identifying the various components of the program that could conceptually run
independently of the other components of the system. The white board gave us the
freedom to begin composing these independetly executing modules and facilitate
proper communication. 

#### PICTURE

The three components required, from receiving the request to load rates to
responding with the aggregated rates, are the route handler for the endpoint,
the rate loader, and a token manager. It was not immidiately obvious how these
components could run independently. The rate loader receives requests from the
route handler and needs a token before making a request to the vendor. However,
multiple rate loaders can execute independently once they have all the necessary
info. We realized that once we facilitate proper communication between the
components, we can start regarding them as worker that receive necessary info
from channels, perform their task, and communicate result through channels.

#### PICTURE

Once we arrived at this composition, it became clear that each individual
worker component can be created multiple times and grouped into pool of workers
that all perform the same task. We can scale our ability to load rates by simply
creating multiple rate loaders.

#### PICTURE

How many rate loaders? Depends on the application but that's now a configurable
paramter. It does not have to be an exact number either. One might configure a
lower and an upper bound for how many rate loaders they want in their app and
can add logic to dynamically scale to thow limits based on the current load. It
also simplifies maintenance because ...

## Importance of Communication

As our vendor services have grown, we found that we would return to the
white board time and again whenever we added new features and functionality. The
discussion usually revolved around whether we needed to add another worker to
implement some trivial functionality. For example, we wanted to log all raw
communication to and from the vendor APIs but since they were typically large,
we wanted to store them in S3. Since we already have a servie that abstracts the
intricases of using AWS S3 APIs, all we had to do was send a request to this
service along with what we wanted to get logged. This could be and it was
initially implemented as a simple function call. Everywhere we wanted to log, we
would call this function and it was relatively simple.

But this was not consistent with the rest of the system. When designing the
concurrant structure for loadig rates, we were communicating some more about how
the program should behave. We decided that the program is composed of
independently excutable components that are explicilty connected with
channels. It then makes sense that any other independetly executable component
should be implemented as another worker, connected to other workers if needed
through a channel. The clarity and consistency in the implementation not only
helps new engineers learn the code base quickly, but it also allows the team to
discuss whether the design makes sense as the program grows. The worker model
might make sense now but if we see that we have to wrestle with a lot of issues
as the program grows, it becomes clear that we need to reevaluate the design.


## Helpful Links 

<https://blog.golang.org/share-memory-by-communicating>
<https://medium.com/@1_00794/parallelism-models-actors-vs-csp-vs-multithreading-f1bab1a2ed6b>
<https://www.youtube.com/watch?v=cN_DpYBzKso>
<http://www.usingcsp.com/cspbook.pdf>


## REMOVE THESE IDEAS

scalabilty communication configuration maintenance
