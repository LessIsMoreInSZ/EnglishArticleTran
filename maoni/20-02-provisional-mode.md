<h1>Provisional Mode</h1>

A coworker asked me what this “PMFullGC” trigger reason he’s seeing in GCStats means. I thought it’d be useful to share the info here.

PM stands for Provisional Mode which means after a GC starts, it can change its mind about the kind of GC it’s doing. But what does that mean exactly?

So normally when we start a GC, the first things we do are-

determine which generation we collet
if it’s a gen2 we decide if it should be done a background or blocking GC
And after that the collection work will start and go with the decision we made.

When provisional mode is on, while we are already in the middle of a GC, we can say “hmm, it looks like collecting this generation was not a good idea, let’s go with a different generation instead”. This is to handle the cases where our prediction of how the heap would behave is very difficult to get right (or would be expensive to get it more right when we were predicting). Currently there’s only one situation that would trigger this provisional mode. In the future we might add more.

The one situation that triggers the provisional mode is when we detect high memory/high gen2 frag situation during a full blocking GC. And is turned off when we detect neither situation is true in a full blocking GC.

Before I added this provisional mode, the tuning heuristic for this particular situation, ie, high memory load and high fragmentation in gen2, would cause us to do a lot of full compacting GCs because we would think it’d be productive – after all there’s a lot of free space in gen2 and doing a compacting GC would compact it away and get the heap size down which is what we really want when the memory load is high. But if the fragmentation is due to pinning and the pins keep not going away, we could compact but the heap is not shrinking because the pins are still there. And they are in gen2 so it’s harder to use.

We can’t easily predict when the pins will go away. We do know about the pinned handles but we also need to know how much free space would result inbetween them. And it’s hard to know stack pinning unless you actually go walk the stacks. We can operate on the previous knowledge and perhaps stop doing compacting gen2’s for a while and try it again after some number of GCs.

The way I chose to handle this was when we detect this high memory/high fragmentation situation when we do a full compacting GC, we put GC in this provisional mode. And next time when the normal tuning says we are supposed to do a full blocking GC again, we would reduce it to a gen1 compacting GC. We keep doing this and compact as many gen1 survivors into the gen2 free list (so it doesn’t actually increase gen2 size) till a gen1 GC where we can’t fit gen1 survivors into gen2 free list anymore. At this point we change our mind and say we actually want to do a full compacting GC instead. So these GCs are said to be “provisioned” and the trigger reason for this full compacting GC is what you see in GCStats – PMFullGC.

This way I didn’t need to change much of the existing tuning logic. And when we change our mind during the middle of a gen1 GC, we just do a sweeping gen1 so it’ll quickly finish and immediately trigger a full compacting GC right after. We could actually discard what we’ve done so far for this gen1 and “restart” it as a full compacting GC but it doesn’t gain much and would require a much bigger code churn. Since we are discovering this right before we need to decide whether this should be a compacting or sweeping gen1 it’s trivial to just make it a sweeping gen1.

And when we trigger this full compacting GC, if we then detect we are out of the high memory load/high fragmentation situation, most likely because the pins were gone so we were able to compact and reduce the memory load, we could take GC out of the provisional mode.

Of course we hope that normally you don’t have a bunch of pins in gen2 that keep not going away which was why we had our previous tuning logic. And that logic worked well if there wasn’t high fragmentation created by pinning. But we did want to handle this so we could accommodate more customer scenarios. There was a team that hit exactly this situation and before the provisional mode was added they saw many full compacting GCs which made % pause time in GC very high. With this provisional mode they saw the % pause time in GC reduced dramatically because they were doing much fewer full compacting GCs since most of them got converted to gen1s, and still maintained the same heap size.

I also explained provisional mode during my meetup talk in Prague last year. It was made available in 4.7.x on .NET and 3.0 on .NET Core.

https://devblogs.microsoft.com/dotnet/provisional-mode/

一位客户向我询问了关于分析与 GC 句柄相关的性能问题。我觉得除了固定句柄（pinned handles）之外，通常很少有人讨论其他类型的句柄，所以这个话题值得详细解释一下，尤其是因为它是一个面向用户的功能。

背景信息
GC 句柄确实是分代管理的，因此在进行短暂代（ephemeral）垃圾回收时，我们只需要扫描我们正在收集的那几代中的句柄即可。然而，由于句柄的组织方式，句柄的“年龄”存在一些复杂性——我们并不是按单个句柄来组织的，而是将它们分组管理，而每个组中的句柄必须具有相同的年龄。因此，如果我们需要将一个句柄设置为更年轻，就必须将整个组都设置为更年轻，这样它们都会被报告为属于更年轻的那一代。这种行为很难量化，所以我强烈建议使用我们提供的工具进行测量。

句柄的暴露方式
句柄以多种方式暴露出来。对于大多数人来说，最熟悉的方式可能是通过 GCHandle 类型。只有四种类型是通过这种方式暴露的：Normal、Pinned、Weak 和 WeakTrackResurrection。内部称 Weak 和 WeakTrackResurrection 为短弱句柄（short weak handles）和长弱句柄（long weak handles）。但其他类型的句柄则通过 BCL（基础类库）使用，例如依赖句柄（dependent handle），它在 ConditionalWeakTable 中使用（是的，我知道很多人希望直接将这种类型暴露为 GCHandle，我会在下面进一步讨论这个话题）。

历史上，GC 句柄被视为一种“附加功能”，所以我们并不预期会有大量的句柄使用。但正如许多其他东西一样，它也在不断发展。我已经观察到句柄的总体使用量显著增加（部分原因是库越来越多地使用句柄）。目前，我们在 ETW 事件中收集句柄信息，但还不够详细，所以我计划让句柄的诊断信息更加详尽。

当前我们提供的内容
PerfView 的 GCStats 视图

在 PerfView 的 GCStats 视图中，每一行 GC 都有一列显示本次 GC 中提升的固定对象数量。这是 GC 在该次 GC 中观察到的固定对象数量，包括来自固定句柄（异步 IO 使用的固定句柄）或堆栈固定的对象。
Raw Data XML 文件

在 GCStats 视图中的“Raw Data XML 文件（用于调试）”链接下，你可以生成一个包含每次 GC 更详细信息的文件，其中包括句柄信息。你可能会看到类似以下的内容：
xml
深色版本
<GCEvent GCNumber="358" GCGeneration="0" …[many fields omitted] Reason="AllocSmall">
    <HeapStats GenerationSize0="77,228,592"…[many fields omitted] PinnedObjectCount="1" SinkBlockCount="37,946" GCHandleCount="4,552,434"/>
</GCEvent>
其中 PinnedObjectCount 显示的是固定对象的数量，而另一个字段 GCHandleCount 包括的不仅仅是固定句柄，而是所有句柄。需要注意的一个重要区别是，GCHandleCount 是针对所有句柄的总数，而 PinnedObjectCount 只是当前 GC 所能看到的部分（加上堆栈固定的对象）。此外，值得一提的是，我们在 CoreCLR 中对 GCHandleCount 的跟踪是“宽松”的，也就是说，出于性能原因，我们没有使用 Interlocked 增加/减少计数器，而是简单地使用 ++/--，因此这个数字只是一个大致值，但从性能角度来看已经很有参考价值（例如，在示例中，句柄数量非常多，肯定值得进一步调查）。
程序化获取信息

如果你使用 TraceEvent 库，可以通过 TraceGC 类程序化地获取上述所有信息。
注意事项
当然，上面提到的信息主要突出显示了固定对象，而不是其他类型的句柄，例如 WeakReference 使用的弱句柄、ConditionalWeakTable 使用的依赖句柄，或者全框架 ASP.NET 使用的 SizedRef 句柄（这些在 CoreCLR 中已不再存在，因此这里不作讨论）。gc\gcinterface.h 中还列出了其他类型的句柄，但它们是由运行时内部使用的。

在我们提供更易用的信息之前，你可以做的一件相对简单的事情是查看 CPU 分析配置文件，看看是否应该担心这些句柄的使用情况。短弱引用（Short WeakReferences）通过以下方法扫描：

GCScan::GcShortWeakPtrScan （如果内联，则调用 Ref_CheckAlive）
长弱引用（Long WeakReferences）通过以下方法扫描：

GCScan::GcWeakPtrScan （如果内联，则调用 Ref_CheckReachable）
依赖句柄通过以下方法扫描：

在阻塞式 GC 中：gc_heap::scan_dependent_handles
在后台 GC 中：gc_heap::background_scan_dependent_handles
我知道这并不是理想的方法，但它是一种获取信息的方式。值得一提的是，这些操作目前都在后台 GC 的 STW（Stop-the-World）暂停期间完成。此外，依赖句柄的扫描目前效率不高，这也是我没有直接将其作为 GCHandle 类型暴露的主要原因。我已经有一个设计方案来大幅提升其性能，一旦实现，我们将公开这种句柄类型。

收集句柄创建/销毁事件的 PerfView 命令
以下是用于收集句柄创建和销毁事件的 PerfView 命令：

bash
深色版本
perfview /nogui /KernelEvents=default /ClrEvents:GC+Stack+GCHandle /clrEventLevel=Informational collect