<h1>So, what’s new in the CLR 4.0 GC?</h1>

PDC 2008 happened not long ago so I get to write another “what’s new in GC” blog entry. For quite a while now I’ve been working on a new concurrent GC that replaces the existing one. 
And this new concurrent GC is called “background GC”.

First of all let me apologize for having not written anything for so long. It’s been quite busy working on the new GC and other things.

Let me refresh your memory on concurrent GC. Concurrent GC has existed since CLR V1.0. For a blocking GC, ie, a non concurrent GC we always suspend managed threads, 
do the GC work then resume managed threads. Concurrent GC, on the other hand, runs concurrently with the managed threads to the following extend:

§  It allows you to allocate while a concurrent GC is in progress.

 

However you can only allocate so much – for small objects you can allocate at most up to end of the ephemeral segment. Remember if we don’t do an ephemeral GC, 
the total space occupied by ephemeral generations can be as big as a full segment allows so as soon as you reached the end of the segment 
you will need to wait for the concurrent GC to finish so managed threads that need to make small object allocations are suspended.

 

§  It still needs to stop managed threads a couple of times during a concurrent GC.

 

During a concurrent GC we need to suspend managed threads twice to do some phases of the GC. These phases could possibly take a while to finish.

 

We only do concurrent GCs for full GCs. A full GC can be either a concurrent GC or a blocking GC. Ephemeral GCs (ie, gen0 or gen1 GCs) are always blocking.

 

Concurrent GC is only available for workstation GC. In server GC we always do blocking GCs for any GCs.

Concurrent GC is done on a dedicated GC thread. This thread times out if no concurrent GC has happened for a while and gets recreated next time we need to do concurrent GC.

When the program activity (including making allocations and modifying references) is not really high and the heap is not very large concurrent GC works 
well – the latency caused by the GC is reasonable. But as people start writing larger applications with larger heaps that handle more stressful situations, the latency can be unacceptable.

Background GC is an evolution to concurrent GC. The significance of background GC is we can do ephemeral GCs while a background GC is in progress if needed. 
As with concurrent GC, background GC is also only applicable to full GCs and ephemeral GCs are always done as blocking GCs, 
and a background GC is also done on its dediated GC thread. The ephemeral GCs done while a background GC is in progress are called foreground GCs.

So when a background GC is in progress and you’ve allocated enough in gen0, 
we will trigger a gen0 GC (which may stay as a gen0 GC or get elevated as a gen1 GC depending on GC’s internal tuning). 
The background GC thread will check at frequent safe points (ie, when we can allow a foreground GC to happen) and see if there’s a request for a foreground GC. 
If so it will suspend itself and a foreground GC can happen. After this foreground GC is finished, the background GC thread and the user threads can resume their work.

Not only does this allow us to get rid of dead objects in young generations, 
it also lifts the restriction of having to stay in the ephemeral segment – if we need to expand the heap while a background GC is going on, we can do so in a gen1 GC.

We also made some performance improvement in background GC which does better at doing more things concurrently so the time we need to suspend managed threads is also shorter.

We are not offering background GC for server GC in V4.0. It’s under consideration – we recognize how important 
it is for server applications (which usually have much larger heaps than client apps) to benefit from smaller latency but the work did not fit in our V4.0 timeframe. 
For now for server applications, I would recommend you to look at the full GC notification feature we added in .NET 3.5 SP1. 
It’s explained here: http://msdn.microsoft.com/en-us/library/cc713687.aspx. Basically you register to get notified when a full GC is approaching and when it’s finished. 
This allows you to do software load balancing between different server instances – when a full GC is about to happen in one of the server instances, 
you can redirect new requests to other instances.

https://devblogs.microsoft.com/dotnet/so-whats-new-in-the-clr-4-0-gc/

PDC 2008 刚刚结束不久，因此我可以写一篇关于“GC（垃圾回收）新特性”的博客文章。最近一段时间，我一直在开发一种新的并发GC，以取代现有的GC。这个新的并发GC被称为“后台GC（Background GC）”。
首先，我要为这么久没有写任何东西而道歉。开发新的GC以及其他工作让我非常忙碌。

让我先回顾一下并发GC的概念。并发GC自CLR 1.0以来就已经存在。对于阻塞式GC（即非并发GC），我们总是挂起托管线程，执行GC工作，然后恢复托管线程。而并发GC则在一定程度上与托管线程并发运行：

允许在并发GC进行时分配内存：
然而，你只能分配一定数量的内存——对于小对象，你最多只能分配到临时段（ephemeral segment）的末尾。记住，如果我们不进行临时代（ephemeral generation）的GC，
临时代占用的总空间可以大到整个段允许的大小。因此，一旦你到达段的末尾，你需要等待并发GC完成，因此需要分配小对象的托管线程会被挂起。

在并发GC期间仍需要挂起托管线程几次：
在并发GC期间，我们需要挂起托管线程两次以执行GC的某些阶段。这些阶段可能需要一些时间才能完成。

我们只对完全GC（Full GC）执行并发GC。完全GC可以是并发GC，也可以是阻塞GC。临时代GC（即gen0或gen1 GC）始终是阻塞的。

并发GC仅适用于工作站GC（Workstation GC）。在服务器GC（Server GC）中，我们对任何GC都执行阻塞GC。

并发GC在一个专用的GC线程上运行。如果一段时间内没有发生并发GC，该线程会超时，并在下次需要执行并发GC时重新创建。

当程序活动（包括分配内存和修改引用）不是特别高，并且堆不是非常大时，并发GC工作得很好——由GC引起的延迟是合理的。但随着人们开始编写处理更高压力情况的更大应用程序，延迟可能会变得不可接受。

后台GC：并发GC的进化
后台GC是并发GC的进化版本。后台GC的重要意义在于，我们可以在后台GC进行时根据需要执行临时代GC。与并发GC一样，后台GC也仅适用于完全GC，而临时代GC始终是阻塞的。后台GC也在其专用的GC线程上运行。
在后台GC进行时执行的临时代GC被称为“前台GC（Foreground GC）”。

因此，当后台GC正在进行时，如果你在gen0中分配了足够的内存，我们将触发gen0 GC（根据GC的内部调整，它可能保持为gen0 GC，也可能升级为gen1 GC）。
后台GC线程会频繁地在安全点（即允许前台GC发生的点）检查是否有前台GC的请求。如果有，它将挂起自己，前台GC就可以发生。当前台GC完成后，后台GC线程和用户线程可以恢复工作。

这不仅允许我们清理年轻代中的死亡对象，还解除了必须停留在临时段中的限制——如果我们需要在后台GC进行时扩展堆，我们可以在gen1 GC中进行。

我们还对后台GC进行了一些性能改进，使其能够更好地并发执行更多任务，因此挂起托管线程的时间也更短。

服务器GC的现状
在V4.0中，我们没有为服务器GC提供后台GC。这正在考虑中——我们认识到对于服务器应用程序（通常比客户端应用程序拥有更大的堆）来说，减少延迟的重要性，但这项工作没有纳入V4.0的时间表。
目前，对于服务器应用程序，我建议你查看我们在.NET 3.5 SP1中添加的完全GC通知功能。这里有一个解释：完全GC通知功能。基本上，你可以注册以在完全GC即将发生和完成时收到通知。
这允许你在不同的服务器实例之间进行软件负载均衡——当一个服务器实例即将发生完全GC时，你可以将新请求重定向到其他实例。

总结
后台GC是并发GC的进化版本，它允许在完全GC进行时执行临时代GC，从而减少延迟并提高性能。虽然目前尚未在服务器GC中实现，但完全GC通知功能为服务器应用程序提供了一种有效的负载均衡机制。

