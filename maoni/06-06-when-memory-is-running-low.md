<h1>When memory is running low…</h1>
When I say memory I mean physical memory. Let’s assume that you have enough virtual memory space. 
When the physical memory gets low you may start getting OOMs or start paging.

You can experiment and see how you can avoid getting into the low memory situation but sometimes it’s hard to predict and hard to test all scenarios your app can get into. 
For example, if your app handles requests where the amount of memory each request consumes can vary dramatically, 
you may be able to handle 1000 concurrent requests some times before running out of memory but only 800 other times.

First of all, let’s talk about how you know when the memory is low on the machine. 
You can do this by either calling the Win32 API GlobalMemoryStatusEx, or monitor memory performance counters. 
Currently there’s no notification sent by the GC for low memory – one reason being the memory is used not only by the managed heap but also by native heaps/modules loaded/etc.

You can get the process info via either Win32 APIs, perf counters or the managed Process class.

To get the amount of memory that the managed heap is using, the most accurate way is to monitor the “# Total committed bytes” performance counter under .NET CLR Memory.

Something that server throttling code usually does is to monitor the memory usage and when the memory consumption gets high it stops accepting new requests and waits for the current requests to drain. 
When the memory load gets to a reasonably low point it starts accepting new requests again.

But the problem is, if the current requests don’t make any allocations or don’t make enough allocations to trigger a full GC your managed heap doesn’t have a chance to collect dead objects so after the current requests drain the memory consumption does not go down.

What do you do in this case? You can call GC.Collect. This is a legitimate case for calling GC.Collect yourself – but be careful how you call it! There are a few things you should keep in mind when calling GC.Collect:

Don’t call it too often –

you want to set a minimum interval of how often you call it. This would depend on your application. 
Let’s say you have 500MB worth of live objects to collect (meaning the resulting heap size after GC is about 500MB, not counting the garbage), 
perhaps you want to not call GC.Collect more often than every 5 seconds. 
You can determine this by doing a simple test on the type of machines that your app usually runs on and see how long it takes to collect certain number of bytes.
GC happens on its own schedule so you want to account for that. When it’s the time for you to induce a GC a GC may have already happened during the last interval so it’s not necessary for you to induce the GC.
 You can check for this by calling GC.CollectionCount(GC.MaxGeneration) after the interval elapses and if the collection count has already increased from the last time you induced a GC you don’t need to induce another one.
Don’t call it when it’s not productive – if after you call GC.Collect and the memory doesn’t drop by much, you want to call it less often. 
It’s a good measure to check say, if the memory hasn’t dropped by 1% of the total memory you got on the machine. 
If that’s the case perhaps you want to increase the interval by some percentage.

Recently I worked with a team that implemented just that and got good results.

Some people have asked why we don’t just give you a way to limit the amount of memory that a process is allowed to use.
 Imagine if we did, how would you use it? Do you set the amount of memory that your app is allowed to use to 90% of the physical memory? 
 90% sounds good… now, if everyone was to take this suggestion and set their app to use 90% of the memory we are back to square one. 
 If everyone sets it to 50% and if your app is the only active app on the machine you’d be wasting half of the memory.

If your app is running in a very controlled environment, in other words, you are probably using the hosting APIs to gain more control in a managed app, 
there are hosting APIs you can use to limit the amount of memory that managed heap can use. 
You can use IGCHostControl::RequestVirtualMemLimit to deny the request when GC needs to acquire a new segment. 
So you can check for the amount of memory that managed heap uses and if it exceeds the limit you set, you use this API to tell GC to not add new segments. 
The result of this is there will be a GC trigger trying to get back more memory. If after GC is done there’s still no space to satisfy your allocation request you will get an OOM. 
Note that this means you need to be prepared to handle OOMs which can be very difficult or even impossible. 
So to be able to use this API in an appropriate way generally means you’ll need to use other hosting APIs to do things like communicating the memory load to GC (memory load that you defined), 
tracking allocations and terminating tasks or unloading app domains. Refer to the hosting API documentation for details.

BTW, if you really want to limit the memory for a process, you even have a way to do it via Win32 APIs. 
You can create a job object via the CreateJobObject call that you associate with your process, 
and specify the amount of memory the process is allowed to use (or if you have only one process associated you can set the memory limit for the job) via SetInformationJobObject.

Now, if your application simply needs to use a lot more memory than the amount of physical memory, what do you do? 
Currently (meaning, on CLR 2.0 or earlier) if you need to constantly churning through all this data, it would mean you will incur full GCs often. 
If a large portion of the managed heap is stored in the page file and would need to be paged in to do a GC, you will take a pretty serious performance hit. 
It may be worse of the alternative which is storing it as native data and only page in what you need. 
You will still need to go to the page file very often but you don’t need to pay the price of paging in everything to do a GC. 
So right now doing caching in native code (in memory or in a file) could be a better choice, provided that the cache implementation has very low fragmentation. 
Sorry I don’t have a better answer but we are perfectly aware of this problem though and are working hard on it.

当我说内存时，我指的是物理内存。假设你有足够的虚拟内存空间。当物理内存不足时，你可能会开始遇到OOM（内存不足）或分页问题。

你可以通过实验来观察如何避免进入低内存状态，但有时很难预测和测试应用程序可能遇到的所有场景。例如，如果你的应用程序处理请求时，每个请求消耗的内存量差异很大，有时你可能能够处理1000个并发请求，而在其他时候可能只能处理800个。

首先，我们来谈谈如何知道机器上的内存不足。你可以通过调用Win32 API GlobalMemoryStatusEx 或监控内存性能计数器来实现。目前，GC不会发送低内存通知——原因之一是内存不仅由托管堆使用，还由本地堆、加载的模块等使用。

你可以通过Win32 API、性能计数器或托管类 Process 来获取进程信息。

要获取托管堆使用的内存量，最准确的方法是监控 .NET CLR Memory 下的“# Total committed bytes”性能计数器。

服务器限流代码通常做的一件事是监控内存使用情况，当内存消耗较高时，停止接受新请求并等待当前请求处理完毕。当内存负载降到合理低点时，再开始接受新请求。

但问题是，如果当前请求没有进行任何分配，或者没有进行足够的分配来触发完整GC，托管堆就没有机会回收死对象，因此在当前请求处理完毕后，内存消耗不会下降。

在这种情况下，你可以调用 GC.Collect。这是你自己调用 GC.Collect 的合理情况——但要注意如何调用它！在调用 GC.Collect 时，有几件事需要记住：

不要频繁调用：

你需要设置一个最小的时间间隔来决定调用它的频率。这取决于你的应用程序。假设你有500MB的存活对象需要回收（意味着GC后的堆大小约为500MB，不包括垃圾），你可能希望不要超过每5秒调用一次。

你可以通过在应用程序通常运行的机器上进行简单测试来确定这一点，看看回收一定字节数需要多长时间。

GC有自己的调度机制，因此你需要考虑到这一点。当你准备触发GC时，可能在上一个时间间隔内已经发生了GC，因此你不需要再次触发GC。你可以在时间间隔结束后调用 GC.CollectionCount(GC.MaxGeneration) 来检查这一点，如果回收计数已经增加，则不需要再次触发GC。

在无效时不调用：

如果你调用 GC.Collect 后内存没有显著下降，你应该减少调用频率。一个好的衡量标准是，如果内存没有下降超过机器总内存的1%，你可能需要增加调用间隔的百分比。

最近我与一个团队合作，他们实现了这一点并取得了良好的效果。

有些人问，为什么我们不直接提供一种方法来限制进程允许使用的内存量。想象一下，如果我们这样做了，你会如何使用它？你会将应用程序允许使用的内存量设置为物理内存的90%吗？90%听起来不错……但如果每个人都采纳这个建议，并将他们的应用程序设置为使用90%的内存，我们就回到了原点。如果每个人都设置为50%，而如果你的应用程序是机器上唯一活跃的应用程序，你将浪费一半的内存。

如果你的应用程序在一个非常受控的环境中运行，换句话说，你可能正在使用托管API来在托管应用程序中获得更多控制权，那么有一些托管API可以用来限制托管堆可以使用的内存量。你可以使用 IGCHostControl::RequestVirtualMemLimit 来拒绝GC在需要获取新段时的请求。因此，你可以检查托管堆使用的内存量，如果超过了你设置的限制，你可以使用此API告诉GC不要添加新段。这样做的结果是会触发GC以尝试回收更多内存。如果GC完成后仍然没有空间来满足你的分配请求，你将遇到OOM。请注意，这意味着你需要准备好处理OOM，这可能非常困难甚至不可能。因此，能够以适当的方式使用此API通常意味着你需要使用其他托管API来执行诸如将内存负载（你定义的内存负载）传达给GC、跟踪分配以及终止任务或卸载应用程序域等操作。有关详细信息，请参阅托管API文档。

顺便说一下，如果你真的想限制进程的内存，你甚至可以通过Win32 API来实现。你可以通过 CreateJobObject 调用创建一个与你的进程关联的作业对象，并通过 SetInformationJobObject 指定进程允许使用的内存量（或者如果你只关联了一个进程，你可以为作业设置内存限制）。

现在，如果你的应用程序需要使用比物理内存多得多的内存，你该怎么办？目前（即在CLR 2.0或更早版本中），如果你需要不断处理所有这些数据，这意味着你将经常触发完整GC。如果托管堆的大部分存储在页面文件中，并且需要进行分页以执行GC，你将遭受严重的性能损失。这可能比将数据存储为本地数据并仅分页所需内容更糟糕。你仍然需要经常访问页面文件，但你不需要为执行GC而分页所有内容付出代价。因此，目前在本机代码中进行缓存（在内存或文件中）可能是更好的选择，前提是缓存实现的碎片化非常低。抱歉，我没有更好的答案，但我们完全意识到这个问题，并正在努力解决它。