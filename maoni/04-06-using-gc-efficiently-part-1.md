Using GC Efficiently – Part 1
So the goal here is to explain the cost of things so you can make good decisions in your managed memory usage – it’s not to explain GC itself – it’s to explain how to use it. I assume most of you are more interested in using a garbage collector than implementing one yourself.  It assumes basic understanding of GC. Jeff Richter wrote 2 excellent MSDN articles on GC that I would recommand if you need some background: 1 and 2.

 

First I’ll focus on Wks GC (so all the numbers are for Wks GC). Then I’ll talk about stuff that’s different for Svr GC and when you should use which (sometimes you don’t necessary have the choice and I’ll explain why you don’t).

 

Generations

 

The reason behind having 3 generations is that we expect for a well tuned app, most objects die in Gen0. For example, in a server app, the allocations associated each request should die after the request is finished. And the in flight allocation requests will make into Gen1 and die there. Essentially Gen1 acts as a buffer between yound object areas and long lived object areas. When you look at the number of collections in perfmon, you want to see a low ratio of Gen2 collections over Gen0 collection. The number of Gen1 collections is relatively unimportant. Collecting Gen1 is not much expensive than collecting Gen0.

 

GC segments

 

First let’s see how GC gets memory from the OS. GC reserves memory in segments. Each segment is 16 MB. When the EE (Execution Engine) starts, we reserve the initial GC segments – one for the small object heap and the other for the LOH (large object heap).

 

The memory is committed and decommitted as needed. When we run out of segments we reserve a new one. In each full collection if a segment is not in use it’ll be deleted.

 

The LOH always lives in its own segments – large objects are treated differently from small objects thus they don’t share segments with small objects.

 

Allocation

 

When you allocate on the GC heap, exactly what does it cost? If we don’t need to do a GC, allocation is 1) moving a pointer forward and 2) clearing the memory for the new object. For finalizable objects there’s an extra step of entering them into a list that GC needs to watch.

 

Notice I said “if we don’t need to do a GC” – this means the allocation cost is proportional to the allocation volume. The less you allocate, the less work GC needs to do. If you need 15 bytes, ask for 15 bytes; don’t round it up to 32 bytes or some other bigger chunk like you used to do when you used malloc. There’s a threshold that when exceeded, a GC will be triggered. You want to trigger that as infrequently as you can.

 

Another property of the GC heap that distinguishes itself from the NT heap is that objects allocated together stay together on the GC heap thus preserves the locality.

 

Each object allocated on the GC heap has an 8-byte overhead (sync block + method table pointer).

 

As I mentioned, large objects are treated differently so for large objects generally you want to allocate them in a different pattern. I will talk about this in the large object section.

 

Collection

 

First thing first – when exactly does a collection happen (in other words, when is a GC triggered)? A GC occurs if one of the following 3 conditions happens:

 

1)      Allocation exceeds the Gen0 threshold;

2)      System.GC.Collect is called;

3)      System is in low memory situation;

1) is your typical case. When you allocate enough, you will trigger a GC. Allocations only happen in Gen0. After each GC, Gen0 is empty. New allocations will fill up Gen0 and the next GC will happen, and so on.
 

You can avoid 2) by not calling GC.Collect yourself – if you are writing an app, usually you should never call it yourself. BCL is basically the only place that should call this (in very limited places). When you call it in your app the problem is when it’s called more often than you predicted (which could easily happen) the performance goes down the drain because GCs are triggered ahead of their schedule, which is adjusted for best performance.

 

3) is affected by other processes on the system so you can’t exactly control it except doing your part of being a good citizen in your processes/components.

 

Let’s talk about what this all means to you. First of all, the GC heap is part of your working set. And it consumes private pages. In the ideal situation, objects that get allocated always die in Gen0 (meaning, they almost get collected by a Gen0 collection and there’s never full collection happening) so your GC heap will never grow beyond the Gen0 size. In reality of course that’s almost never the case. So you really want to keep your GC heap size under control.

 

Secondly, you want to keep the time you spend in GC under control. This means 1) fewers GCs and 2) fewer high generation GCs. Collecting a higher generation is more expensive than collecting a lower generation because collecting a higher generation includes collecting objects that live in that generation and the lower generation(s). Either you allocate very temporary objects that die really quickly (mostly in Gen0, and Gen0 collections are cheap) or some really long lived objects that stay in Gen2. For the latter case, the usual scenario is the objects you allocate up front when the program starts – for example, in an ordering system, you allocate memory for the whole catalog and it only dies when your app is terminated.

 

CLRProfiler is an awesome tool to use to look at your GC heap see what’s in there and what’s holding objects alive.

 

How to organize your data

 

1) Value type vs. Reference type

 

As you know value types are allocated on the stack unlike reference types which are allocated on the GC heap. So people ask how you decide when to use value types and when to use reference types. Well, with performance the answer is usually “It depends” and this one is no different (did you actually expect something else? J). Value types don’t trigger GCs but if your value type is boxed often, the boxing operation is more expensive than creating an instance of a reference type to begin with; and when value types are passed as parameters they need to be copied. But then if you have a small member, making it a reference type incurs a pointer size overhead (plus the overhead for the reference type). We’ve seen some internal code where making it inline (ie, as a value type) improved perf as it decreased working set. So it really depends on your types’ usage pattern.

 

2) Reference rich objects

 

If an object is reference rich, it puts pressure on both allocation and collection. Each embedded object incurs 8 bytes overhead. And since allocation cost is proportional to allocation volume the allocation cost is now higher. When collecting, it also takes more time to build up the object graph.

 

As far as this goes, I would just recommand that normally you should just organize your classes according to their logical design. You don’t want to hold other objects alive when you don’t have to. For example, you don’t want to store references of young objects in old objects if you can avoid it.

 

3) Finalizable objects

 

I will cover more details about finalization in its own section but for now, one of most important things to keep in mind is when a finalizable object gets finalized, all the objects it holds on to need to be alive and this drives the cost of GC higher. So you want to isolate your finalizable objects from other objects as much as you can.

 

4) Object locality

 

When you allocate the children of an object, if the children need to have similar life time as their parent they should be allocated at the same time so they will stay together on the GC heap.

 

Large Objects

 

When you ask for an object that’s 85000 bytes or more it will be allocated on the LOH. LOH segments are never compacted – only swept (using a free list). But this is an implementation detail that you should NOT depend on – if you allocate a large object that you expect to not move, you should make sure to pin it.

 

Large objects are only collected with every full collection so they are  expensive to collect. Sometimes you see after a full collection the Gen2 heap size doesn’t change much. That could mean the collection was triggered for the LOH (you can judge by looking at the decrease in the LOH size reported by perfmon).

 

A good practice with large objects is to allocate one and keep reusing it so you don’t incur more full GCs. If say you want a large object that can hold either 100k or 120k, allocate one that’s 120k and reuse that. Allocating many very temporary large objects is a very bad idea ‘cause you’ll be doing full collections all the time.

 

 

That’s all for Part 1. In the future entries I’ll cover things like pinning, finalization, GCHandles, Svr GC and etc. If you have questions about the topics I covered in this entry or would like more info on them feel free to post them.

使用 GC 的高效方法 – 第 1 部分
这里的目标是解释事物的成本，以便您可以在托管内存使用中做出明智的决策——这不是解释 GC 本身——而是解释如何使用它。我假设大多数人对使用垃圾收集器比自己实现一个更感兴趣。假设对 GC 有基本的了解。如果您需要一些背景知识，Jeff Richter 写了两篇关于 GC 的优秀 MSDN 文章，我推荐阅读：1 和 2。

首先，我将重点关注 Wks GC（因此所有数字都针对 Wks GC）。然后我会讨论 Svr GC 的不同之处以及何时应该使用哪种（有时您不一定有选择的余地，我会解释为什么没有）。

代

拥有 3 代的原因是我们期望对于调优良好的应用程序，大多数对象在 Gen0 中死亡。例如，在服务器应用程序中，与每个请求相关的分配应在请求完成后死亡。而正在进行的分配请求将进入 Gen1 并在那里死亡。本质上，Gen1 充当年轻对象区域和长寿命对象区域之间的缓冲区。当您查看 perfmon 中的收集次数时，您希望看到 Gen2 收集与 Gen0 收集的低比率。Gen1 收集的数量相对不重要。收集 Gen1 并不比收集 Gen0 昂贵多少。

GC 段

首先让我们看看 GC 如何从操作系统获取内存。GC 在段中保留内存。每个段为 16 MB。当 EE（执行引擎）启动时，我们保留初始 GC 段——一个用于小对象堆，另一个用于 LOH（大对象堆）。
内存根据需要提交和取消提交。当我们用完段时，我们会保留一个新段。在每次完整收集中，如果一个段未使用，它将被删除。
LOH 始终存在于自己的段中——大对象与小对象的处理方式不同，因此它们不会与小对象共享段。

分配

当你在大对象堆上分配内存时，具体需要付出什么代价？如果我们不需要进行垃圾回收，分配操作包括以下两步：1）向前移动一个指针；2）为新对象清除内存。对于可终结的对象，还需要一个额外的步骤，即将它们加入垃圾回收器需要监视的列表中。

注意我说的是“如果我们不需要进行垃圾回收” —— 这意味着分配的成本与分配的量成比例。你分配得越少，垃圾回收器需要做的工作就越少。如果你需要15字节，就请求15字节；不要像你使用malloc时那样将其四舍五入到32字节或其他更大的块。有一个阈值，当超过这个阈值时，就会触发垃圾回收。你希望尽可能少地触发这个操作。

垃圾回收堆与NT堆的另一个不同之处在于，一起分配的对象在垃圾回收堆上会保持在一起，从而保持了局部性。

在垃圾回收堆上分配的每个对象都有8字节的额外开销（同步块 + 方法表指针）。

正如我提到的，大对象的处理方式不同，因此对于大对象，通常你希望以不同的模式进行分配。我将在大对象部分讨论这个问题。
首先，让我们明确一下垃圾回收（GC）究竟何时发生（换句话说，何时触发GC）？GC会在以下三个条件之一发生时进行：

1.分配超过Gen0阈值；
2.调用System.GC.Collect；
3.系统处于低内存情况。
是典型的情况。当你分配足够多的内存时，将会触发GC。分配只发生在Gen0。每次GC之后，Gen0会被清空。新的分配将填满Gen0，然后下一次GC会发生，如此循环。
你可以通过不自己调用GC.Collect来避免2) —— 如果你正在编写一个应用程序，通常你不应该自己调用它。BCL（基础类库）是基本上唯一应该调用这个方法的地方（在非常有限的情况下）。在你的应用程序中调用它的问题是，当它被调用的频率超过你的预期时（这很容易发生），性能会下降，因为GC会在它们最佳性能调整的计划之前被触发。

受到系统上其他进程的影响，所以你无法完全控制它，除了在你的进程/组件中做好自己的分内之事。
让我们来谈谈这一切对你意味着什么。首先，GC堆是你工作集的一部分。它消耗私有页面。在理想情况下，分配的对象总是在Gen0中死亡（这意味着，它们几乎总是被Gen0收集，并且永远不会发生完全收集），所以你的GC堆永远不会超过Gen0的大小。当然，在现实中这几乎从未发生过。所以你真的想要控制你的GC堆大小。

其次，你想要控制你在GC上花费的时间。这意味着1) 减少GC的次数和2) 减少高代GC的次数。收集更高代的对象比收集更低代的对象要昂贵，因为收集更高代包括收集那个代以及更低代中的对象。你要么分配非常短暂的对象，它们很快就会死亡（主要在Gen0，Gen0的收集相对便宜），要么是一些真正长期存活的对象，它们会留在Gen2。对于后一种情况，通常的场景是在程序启动时分配的对象——例如，在一个订单系统中，你为整个目录分配内存，只有当你的应用程序终止时，这些内存才会被释放。

CLRProfiler是一个很棒的工具，你可以用它来查看你的GC堆，看看里面有什么，以及是什么在保持对象存活。

如何组织你的数据

1.值类型与引用类型
你知道，值类型是在栈上分配的，与在GC堆上分配的引用类型不同。那么，人们经常会问，何时使用值类型，何时使用引用类型。关于性能的问题，答案通常是“视情况而定”，这个也不例外（你难道期待其他的答案吗？:）。值类型不会触发GC，但是如果你的值类型经常被装箱，装箱操作比一开始就创建一个引用类型的实例要昂贵；当值类型作为参数传递时，它们需要被复制。但是，如果你有一个小的成员，将其作为引用类型会带来指针大小的开销（加上引用类型的额外开销）。我们见过一些内部代码，将成员内联（即作为值类型）提高了性能，因为它减少了工作集。所以，这实际上取决于你的类型的使用模式。

2.富引用对象
如果一个对象富含引用，它会增加分配和收集的压力。每个嵌入的对象都会产生8字节的额外开销。由于分配成本与分配量成比例，因此分配成本现在更高。在收集时，构建对象图也需要更多时间。

就这一点而言，我通常会建议你按照逻辑设计来组织你的类。如果你不需要，你不想让其他对象保持存活。例如，如果你能避免，就不要在老对象中存储年轻对象的引用。

3.可终结对象
我将在其自己的部分中详细介绍终结，但现在，你需要记住的最重要的事情之一是，当一个可终结对象被终结时，它所持有的所有对象都需要存活，这将增加GC的成本。因此，你希望尽可能地将可终结对象与其他对象隔离开。

4.对象局部性
当你为一个对象分配子对象时，如果子对象需要与它们父对象有相似的生命周期，它们应该同时分配，这样它们就会在GC堆上一起存活。

大对象

当你请求一个85000字节或更多的对象时，它将在LOH上分配。LOH段永远不会被压缩——只会被清除（使用空闲列表）。但这是一个你不应该依赖的实现细节——如果你分配了一个你期望不移动的大对象，你应该确保将其固定。

大对象只在每次完全收集时被收集，因此收集它们是非常昂贵的。有时你会在完全收集后看到Gen2堆大小没有太大变化。这可能意味着收集是由于LOH触发的（你可以通过查看perfmon报告的LOH大小减少来判断）。

关于大对象的一个好习惯是分配一个并重复使用它，这样你就不会产生更多的完全GC。例如，如果你想要一个可以容纳100k或120k的大对象，就分配一个120k的并重复使用它。分配许多非常短暂的大对象是一个非常糟糕的主意，因为你将一直进行完全收集。

以上是第一部分的内容。在未来的文章中，我将涵盖诸如固定、终结、GCHandles、Svr GC等内容。如果你对我在这篇文章中涵盖的主题有疑问，或者想要了解更多信息，请随时发布。