<h1>GC Handles</h1>

A customer asked me about analyzing perf related to GC handles. I feel like aside from pinned handles in general handles are not talked about much so this topic warrants some explanation, especially since this is a user facing feature.

For some background info, GC handles are indeed generational so when we are doing ephemeral GCs we only need to scan handles in the generations we are collecting. However there are complications associated with handles’ ages because of the way handles are organized – we don’t organize by individual handles, we organize them in groups and each group needs to have the same age so if we need to set one handle to be younger we set all of them to be younger so they will all get reported for that younger generation. But this is hard to quantify so I would definitely recommend measuring with the tools we provide.

Handles are exposed in various ways. The way that’s perhaps the most familiar to most folks is via the GCHandle type. Only 4 types are exposed this way: Normal, Pinned, Weak and WeakTrackResurrection. Weak and WeakTrackResurrection types are internally called short and long weak handles. But other types are used via BCL such as the dependent handle type which is used in ConditionalWeakTable (and yes, I am aware that there’s desire to expose this type directly as GCHandle, I’ll touch more on this topic below).

Historically GC handles were considered as an “add-on” thing so we didn’t expect many of these. But as with many other things, they evolve. I’ve been seeing handle usage in general going up by quite a lot (part of it is due to libraries using handles more and more). And currently, we collect handle info in ETW events but not very detailed so I do plan to make the diagnostics info on handles more detailed.

Right now what we offer is:

# of pinned objects promoted in this GC as a column for each GC in the GCStats view in PerfView. This is the number of pinned objects that GC observed in that GC including from pinned handles (async pinned handles used by IO) or pinned by the stack.
In the “Raw Data XML file (for debugging)” link in the GCStats view you can generate a file with more detailed info per GC including handle info; you’ll see something like this:
<GCEvent GCNumber="358" GCGeneration="0" …[many fields omitted] Reason= "AllocSmall"> <HeapStats GenerationSize0="77,228,592"…[many fields omitted] PinnedObjectCount="1" SinkBlockCount="37,946" GCHandleCount="4,552,434"/>

The PinnedObjectCount is also shown here. Then there’s another field called GCHandleCount. As you would guess, GCHandleCount includes not just pinned handles but all handles. Also there’s another significant difference which is this is for all handles whereas PinnedObjectCount is only for what that GC sees so you could have a lot more pinned handles but since only the ones for that generation are reported PinnedObjectCount only includes those (plus stack pinned objects). Another thing worth mentioning is we track GCHandleCount “loosely” in coreclr, as in, we don’t use Interlocked inc/dec for perf reasons. We just do ++/– so this count is a rough figure but gives you a good idea perf wise (eg, in the example shown, that’s a lot of handles and definitely worth some investigation).

Of course if you use the TraceEvent library you can get all the above info on the TraceGC class programmatically.
As you can see this highlights the pinned objects, not other types of handles like the weak GC handles used by WeakReference, dependent handles used by ConditionalWeakTable uses or the SizedRef handles used by asp.net in full framework (no longer exists on coreclr so I’ll not cover them here). There are other types as shown in gc\gcinterface.h but they are used internally by the runtime.

Before we provide more easily consumable info, one fairly easy thing you could do is look at the CPU profiles to see if you should be worried about the usage of these handles – short WeakReferences are scanned with this method:

GCScan::GcShortWeakPtrScan (in case it’s inlined it calls Ref_CheckAlive)

long WeakReferences are scanned with:

GCScan::GcWeakPtrScan (in case it’s inlined it calls Ref_CheckReachable)

Dependent handles are scanned with:

gc_heap::scan_dependent_handles in blocking GCs gc_heap::background_scan_dependent_handles in BGCs

I know this is not ideal but it’s one way to get the info. One thing worth mentioning is these are currently all done during the STW pause for BGCs. And dependent handle scanning is currently done in a not very efficient way which is the main reason why I haven’t exposed this directly as a GCHandle type. I have a design in place to make the perf much better. When we have it implemented we will make this handle type public.

The PerfView commandline to collect events for creating/destroying handles – perfview /nogui /KernelEvents=default /ClrEvents:GC+Stack+GCHandle /clrEventLevel=Informational collect


https://devblogs.microsoft.com/dotnet/gc-handles/

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