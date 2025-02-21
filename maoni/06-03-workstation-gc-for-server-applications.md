<h1>Workstation GC for server applications?</h1>
In Using GC Efficiently – Part 2 I talked about different flavors of GC that exist in the CLR and how you choose 
which flavor is good for your applications, and I said that the Server GC flavor is designed for server applications. 
As with any performance tuning there are always exceptions – there’s no one-rule-fits-all. Recently I worked with an internal group at MS on their 64-bit server scenario 
and it’s one of those situations where you do want to try Workstation GC for server applications so I thought it’d be interesting to talk about it here. 
The scenario is they are running 12 ASP.NET server processes on a 4-proc machine. Since they are using ASP.NET they get Server GC by default.  

 

First of all, let’s review how the server GC works. It creates a GC thread for each CPU you have on the machine and each GC thread collects parts of the GC heap. 
This means when you have 12 processes you get 48 GC threads. Imagine if all of the processes start doing GCs, there are 48 threads to schedule. In a managed process, 
the GC threads need to rendezvous at some points. 
Imagine if in the first process, one of the 4 threads happens to not be scheduled till the GC threads from the other 11 processes are scheduled, 
the first process will need to wait 11*4=44 quatums.

 

Keep in mind that it’s very common to run only one server process on a dedicated machine so you don’t get into this situation.
 But there are legitimate reasons when you do want to run multiple server processes on the same machine. For example, finacial cost for multiple machine, 
 adminitrative cost to manage multiple machines or all the servers need to share the same IP address (if they all server the same web site).

 

So when would you get into the situation where all processes start doing GCs at the same time? 
One of the common attributes of server applications is they try to use as much resource as possible on the machine. 
This is something that many people find hard to accept ‘cause we normally talk about how to make your app use less resource. 
When you dedicate a machine to running a specific application (+ the normal processes that the OS runs), 
you do want that app to use as much resource as possible otherwise the rest of the resource would be wasted. 
So normally server applications try to maximize their memory usage. 
As I mentioned in Using GC Efficiently – Part 1 one of reasons a GC would be triggered is when the machine is in low memory situation. 
So if you run multiple managed server applications they could all start to GC when the memory gets low. 
Note that this is not a specific problem to managed applications – native apps share the same problem 
because when the memory gets low they all start cleaning up processs which means they will all compete for CPU.

 

An absolute cricial thing to developing a server application is throttling. Unfortunately this is not known to everyone who develops server apps. 
Especially when developing managed server apps, some people think that GC should just take care of that aspect not realizing that GC manages memory, not requests. 
If your machine has 2GB of memory and each request takes 1MB of memory, it can’t handle more than 2048 requests concurrently. 
If you keep accepting requests as fast as they come in without any kind of throttling you will run out of memory when the number of requests exceeds what your machine can handle.

 

Some people think Server GC collects faster due to multiple GC threads working together so that would be one reason to choose Server GC over Workstation GC. 
This is true when you are collecting the same size of heap. Without throttling, however, 
you could have a bigger heap to collect because Server GC is more efficient (scalable on MP) so more time is spent accepting new requests 
if you accept much more concurrent requests you could end up using a lot more memory so you may not observe less time in GC simply because now GC has a lot more work to do.

 

Workstation GC with Concurrent GC off, on the other hand, runs on the user thread that triggered the GC and doesn’t create addition threads. 
When you run many processes on the same machine this can dramatically cut down the number of threads. As a matter of fact,
 our internal testing showed with many ASP.NET server processes running on one machine Workstation GC (no Concurrent GC) proved to be a good choice.

 在《高效使用GC – 第二部分》中，我讨论了CLR中存在的不同GC类型，以及如何选择适合你应用程序的GC类型。我提到，Server GC是为服务器应用程序设计的。然而，与任何性能调优一样，总有一些例外——没有一种规则适用于所有情况。最近，我与微软的一个内部团队合作，处理他们的64位服务器场景，这是一个你确实希望在服务器应用程序中尝试Workstation GC的情况，因此我认为在这里讨论一下会很有趣。他们的场景是在一台4核机器上运行12个ASP.NET服务器进程。由于他们使用ASP.NET，默认情况下会使用Server GC。

首先，让我们回顾一下Server GC的工作原理。它会为机器上的每个CPU创建一个GC线程，每个GC线程负责收集部分GC堆。这意味着当你运行12个进程时，会有48个GC线程。想象一下，如果所有这些进程同时开始执行GC，就会有48个线程需要调度。在托管进程中，GC线程需要在某些时刻进行同步。假设在第一个进程中，4个线程中的一个恰好没有被调度，直到其他11个进程的GC线程都被调度，那么第一个进程将需要等待11*4=44个时间片。

需要注意的是，通常在一台专用机器上只运行一个服务器进程，因此你不会遇到这种情况。但也有一些合理的理由让你希望在同一台机器上运行多个服务器进程。例如，多台机器的财务成本、管理多台机器的管理成本，或者所有服务器需要共享相同的IP地址（如果它们都服务于同一个网站）。

那么，什么时候会出现所有进程同时开始执行GC的情况呢？服务器应用程序的一个常见特征是它们会尝试尽可能多地使用机器上的资源。这可能是许多人难以接受的，因为我们通常讨论的是如何让你的应用程序使用更少的资源。当你将一台机器专用于运行特定应用程序（加上操作系统运行的正常进程）时，你确实希望该应用程序尽可能多地使用资源，否则剩余的资源将被浪费。因此，服务器应用程序通常会尝试最大化其内存使用。正如我在《高效使用GC – 第一部分》中提到的，触发GC的原因之一是机器处于低内存状态。因此，如果你运行多个托管的服务器应用程序，它们可能会在内存不足时同时开始执行GC。请注意，这并不是托管应用程序特有的问题——本地应用程序也面临同样的问题，因为当内存不足时，它们都会开始清理进程，这意味着它们都会竞争CPU资源。

开发服务器应用程序时，一个绝对关键的事情是限流（throttling）。不幸的是，并不是所有开发服务器应用程序的人都了解这一点。特别是在开发托管的服务器应用程序时，有些人认为GC应该自动处理这方面的问题，而没有意识到GC管理的是内存，而不是请求。如果你的机器有2GB内存，每个请求占用1MB内存，那么它无法同时处理超过2048个请求。如果你不加限制地接受所有请求，当请求数量超过机器处理能力时，你将耗尽内存。

有些人认为Server GC由于有多个GC线程协同工作，因此收集速度更快，这可能是选择Server GC而不是Workstation GC的一个原因。这在收集相同大小的堆时是正确的。然而，如果没有限流，你可能会有一个更大的堆需要收集，因为Server GC更高效（在多处理器上更具扩展性），因此会花费更多时间接受新的请求。如果你接受更多的并发请求，最终可能会使用更多的内存，因此你可能不会观察到GC时间的减少，仅仅因为现在GC有更多的工作要做。

另一方面，**Workstation GC（关闭并发GC）**在触发GC的用户线程上运行，不会创建额外的线程。当你在同一台机器上运行多个进程时，这可以显著减少线程数量。事实上，我们的内部测试表明，在一台机器上运行多个ASP.NET服务器进程时，**Workstation GC（关闭并发GC）**被证明是一个不错的选择。



