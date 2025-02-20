In the WinDev conference that I just went to, there seems to be some confusion over finalization (such as why it even exists and etc) and other areas. I hope the following will clear up that confusion. If not, let me know.

 

Finalization

 

1)      Why we have finalization

 

Finalization is necessary because you want to make your component robust. You might have no control over your clients of your component in which case if you are seeing an issue (for example, opening a file handle in the exclusive mode before the finalizer closes it) you won’t be able to say “Oh, look this is a bug, go fix it” to whoever wrote the client (heck, the company that wrote the client might already went out of business!).

 

Obviously you want people to call Dispose on your object so the object doesn’t need to be promoted to a higher generation but you are not always in control of people calling Dispose, and very often you are not.

 

2)      What you can do in finalizers

 

Finalizers are only there so you can release your native resources. It’s by design not to do stuff with other managed objects because they would be taken care of by GC. Releasing native resources should be all you do in your objects’ finalize method and that’s a very important thing to do because we are not in a completely managed world – if that were the case, we wouldn’t need finalization at all.

 

3)      When the finalizer thread runs the finalizers

 

Each time we do a GC, we would see if there are any objects whose finalize method needs to run, if so we add it to the freachable queue and set an event to wait up the finalizer thread. Generally if a managed app is doing work, it will trigger GCs so this means whenever you trigger a GC, the finalizer thread, if it’s not already working, will immediately be aware that there are finalizers to run. The finalizer thread doesn’t wake up at “some random point of time in the future”. When it’s waken up is well defined.

 

So what if a finalizer takes a long time to run? Yes it would block the finalizers behind it from running. But remember we said you should only release native resources in your finalize method which should be fast. If it takes days to run a finalizer then there’s a problem in how you implemented the finalize method and you should look into that.

 

A common problem we see is that the finalizer thread would stall if it needs to call into an STA thread to release a COM object because the STA thread is not pumping messages. This is strictly a user error – by design if a thread is STA it should be pumping messages. You can do so via Thread.Join or Application.DoEvents. Sometimes people observe that when they do this sequence:

 

GC.Collect();

GC.WaitForPendingFinalizers();

GC.Collect();

 

The problem would go away – that’s just because WaitForPendingFinalizers happens to pump messages for your thread. This is not recommanded to use to fix this kind of problems because it’s a lot more heavy-weight than calling Thread.Join.

 

Another possible issue is some people claim they observe the finalizers not being run, yet they are not doing anything with COM. If you allocated some finalizable objects, and finish using them, but there’s no GC happening because you are not running out of the Gen0 limit, yes the finalizers won’t get run. But if you are not doing any allocations (that would trigger GCs) it’s unlikely you need the memory anyway so it wouldn’t matter if the finalizers release the native resources or not.

 

When Server GC is turned on

 

As I described in my last blog entry: Using GC Efficiently – Part 2, where I talked about different GC flavors and when Server GC is on, Server GC is not on by default unless you specify it via config or in hosted environment the host sets it for you. You do NOT get it by default on Win2k3 server or anything.

 

Healthy ratio of GCs between different generations

 

When we look at a perfmon log for .NET CLR memory counters, it’s a good indication of healthy GCs when you see a 10:1 ratio between a generation and its higher generation. So if the app does 1000 Gen0 GCs and 100 Gen1 GCs, usually it means it’s very healthy. Of course if the app doesn’t do high generation GCs at all, it’s a great situation to be in! If all the objects you allocate would be gone with a Gen0 collection, you’d be spending a very small amount of time in GC and that’s the ideal situation.

 

Large objects

 

Large objects are collected with every Gen2 GC. If you look at perfmon counters and notice that after a Gen2 GC, the Gen2 heap doesn’t change at all, most likely the large object heap size has decreased ‘cause the Gen2 GC was to get rid of garbage on the large object heap.

 

The definition of large objects is >= 85,000 bytes. Yet sometimes you see less than 85,000 bytes on the large object heap, that is because the runtime itself uses the large object heap as well and it doesn’t necessarily allocate large objects there.

 

Is pinning bad

 

Yes. Pinning could certainly cause very bad fragmentation problems. My suggestion is you should never pin something unless necessary; and when you do pin something, try to either unpin it as quick as possible; or pin buffers that don’t move much – meaning you either pin something on the large object heap where objects don’t move anyway, or you pin buffers that are in Gen2 if you know Gen2 GCs happen very infrequently (this means pinning old buffers hurts less than pinning new allocated buffers ‘cause old buffers tend to be in stable memory that is less likely to move).

 

If you make an async network I/O call, you could be pinning the buffer for quite a while depending on when the async operation finishes. This could possibly create a situation where you have fragmentation scattered around the GC heap because the pinned buffer doesn’t go away fast enough and the space before the pinned buffer can’t be used (too small for the objects being allocated).

 

We’ve done a lot of work (and will keep doing work) to mitigate fragmentation problems caused by pinning.

在WinDev会议上，关于终结（finalization）的混淆似乎有些地方，以下内容希望能澄清这些困惑。如果还有不清楚的地方，请告诉我。

终结（Finalization）

1.为什么我们需要终结
我们需要终结是因为你想让你的组件更加健壮。你可能无法控制使用你组件的客户，在这种情况下，如果你遇到了问题（例如，在终结器关闭之前以独占模式打开文件句柄），
你无法告诉编写客户端的人“哦，看，这是一个错误，去修复它”（搞不好编写客户端的公司已经倒闭了！）。

显然，你希望人们调用Dispose方法来处理对象，这样对象就不需要被提升到更高的代，但你并不总是能控制人们调用Dispose，很多时候你都无法控制。

2.在终结器中你能做什么
终结器只存在于让你能够释放本地资源。这是有意设计成不处理其他托管对象的，因为它们会被垃圾回收器（GC）处理。在你的对象的finalize方法中，你应该只做释放本地资源的事情，
这是非常重要的，因为我们并不是完全处于一个托管的世界中——如果真是那样，我们就根本不需要终结。

3.终结器线程何时运行终结器
每次我们进行垃圾回收时，我们都会检查是否有任何对象的finalize方法需要运行，如果有，我们就将其添加到freachable队列中，
并设置一个事件来唤醒终结器线程。通常，如果一个托管应用程序正在工作，它会触发垃圾回收，这意味着无论何时你触发垃圾回收，如果终结器线程尚未在工作，它将立即意识到有终结器需要运行。
终结器线程不会在“未来的某个随机时间点”醒来。它被唤醒的时间是明确定义的。

那么，如果一个终结器运行很长时间会怎样呢？是的，它会阻止后面的终结器运行。但是记住，你应该只在finalize方法中快速释放本地资源。如果一个终结器需要运行几天，那么你的finalize方法实现就有问题，你应该调查一下。

我们常见的一个问题是，如果终结器线程需要调用STA线程来释放COM对象，而STA线程没有处理消息，那么终结器线程将会停滞。这完全是用户错误——按照设计，如果一个线程是STA，它应该处理消息。
你可以通过Thread.Join或Application.DoEvents来实现。有时人们观察到，当他们执行以下序列时：

GC.Collect(); GC.WaitForPendingFinalizers(); GC.Collect();

问题就会消失——那是因为WaitForPendingFinalizers恰好为你的线程处理了消息。不建议使用这种方法来修复这类问题，因为它比调用Thread.Join要重得多。

另一个可能的问题是，有些人声称他们观察到终结器没有被运行，尽管他们没有使用COM。如果你分配了一些可终结的对象，并且使用完毕，但没有发生垃圾回收，因为你没有达到Gen0的限制，是的，终结器不会被运行。
但是，如果你没有进行任何会触发垃圾回收的分配，你也不太可能需要内存，所以终结器是否释放本地资源也就无关紧要了。

当服务器GC被开启时

正如我在上一篇博客文章《高效使用GC – 第二部分》中描述的，我在那里讨论了不同的GC类型以及何时开启服务器GC，服务器GC默认情况下是不开启的，除非你通过配置指定，或者在托管环境中，宿主为你设置。
在Win2k3服务器或任何其他服务器上，你不会默认获得它。

不同代之间GC的健康比例

当我们查看.NET CLR内存计数器的perfmon日志时，不同代之间10:1的比例是健康GC的一个良好指标。所以，如果一个应用程序进行了1000次Gen0 GC和100次Gen1 GC，通常这意味着它非常健康。
当然，如果一个应用程序根本不进行高代GC，那是一个非常好的情况！如果你分配的所有对象都会在Gen0收集时消失，你将花费非常少的时间在GC上，这是理想的情况。

大对象

大对象在每次Gen2 GC时都会被收集。如果你查看perfmon计数器，并注意到在Gen2 GC之后，Gen2堆的大小没有变化，很可能是因为大对象堆的大小已经减少了，因为Gen2 GC是为了摆脱大对象堆上的垃圾。

大对象的定义是大于或等于85,000字节。然而，有时你会在大对象堆上看到小于85,000字节的对象，那是因为运行时本身也使用大对象堆，而且它不一定在那里分配大对象。

固定（Pinning）是不是不好

是的。固定确实可能导致非常严重的碎片问题。我的建议是，除非必要，否则永远不要固定任何东西；当你固定某物时，要么尽可能快地解除固定；
要么固定不太移动的缓冲区——这意味着你要么固定大对象堆上的对象，因为那里的对象反正不会移动，要么固定你知道Gen2 GC发生频率非常低的Gen2中的缓冲区（这意味着固定旧的缓冲区比固定新分配的缓冲区伤害要小，
因为旧的缓冲区倾向于位于稳定的内存中，不太可能移动）。

如果你进行异步网络I/O调用，你可能需要根据异步操作完成的时间来固定缓冲区一段时间。这可能会创建一种情况，即由于固定缓冲区没有快速

