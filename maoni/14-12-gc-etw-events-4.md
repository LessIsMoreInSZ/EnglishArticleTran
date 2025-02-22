<h1>GC ETW Events – 4</h1>

So you thought it was over, eh? But wait, there is more! My vacation is not over yet! 😀

In the last blog entry I explained the Suspend MSec and the Pause MSec columns in GCStats and how they are calculated. In this entry I’d like to talk about a few other columns.

GenX MB, GenX Surival Rate % and GenX Frag %

First of all, these are calculated at the end of a GC. The GenX MB column includes the fragmentation in that generation. 
Survival Rate % for GenX is (# of bytes survived from that generation / # of bytes taken up by objects in that generation before the collection). 
Say the generation is 4mb, and the fragmentation is 1mb, so # of bytes taken up by objects is 3mb, if out of these 3mb, 1.5mb survived, 
it means the survival rate % is 50%. And if after the collection this generation is 2mb and the fragmentation is 0.5mb, the frag % is 25%.

You will notice sometimes the GenX Surival Rate % column says NaN. 
It’s because we are not collecting that generation therefore doesn’t make sense to talk about that generation’s survival rate. 
So if we are doing a gen1 GC, gen2’s and LOH’s survival rate will be NaN.

If you are pinning (or the libraries you are using pin on your behalf, eg, IO), usually you would see the frag % for gen0 very big,
 sometimes even 100% (100% doesn’t mean we have only fragmentation in gen0, it’s just because we calculate this column as 
 (fragmentation in gen0 in MB / gen0 size in MB) and they differ a small enough amount that fragmentation in gen0 in MB is the same as gen0 size in MB. 
 We try to leave fragmenation in gen0 as much as we can because it can immediately be used to satisfy allocation requests.

Background GC does not compact so it’s not surprising that after a BGC you see the Frag % in gen2 being fairly big (ie, > 50%). 
We will do a compacting gen2 if we detect that there’s a lot of fragmentation in gen2 that we cannot efficiently use.

We do not compact LOH unless explicitly being asked to. So if the frag % for LOH remains big, 
it means we cannot use a lot of the free spaces on LOH. And if LOH takes up a big percentage of the total heap, it’s worth looking into optimzing the LOH usage.

Promoted MB

This is the # of bytes promoted during a GC. So if we are doing a gen1 GC this is the # of bytes promoted from gen0 and gen1. 
So you could imagine that since gen0 and gen1 usually are a small percentage of the heap the promoted bytes for ephemeral GC usually are 
also a small percentage of what you’d see during your gen2 GCs. As I mentioned before, pause time of a GC is proportional to how much this GC survived, 
in other words, proportional to what you see in the Promoted MB column here (of course background GC is an exception to this as it’s specifically designed to minimize pause time).

A good way to know that something unusual is up is by looking at the pause time of a GC vs its promoted memory. 
If the promoted memory is small yet the pause time is big, it’s almost certain something is up that’s not related to GC work. So what’s considered small?
One way to look at it is to compare it with your other GCs. Let’s say all your full blocking GCs promoted ~1GB and took 1 second except a couple of them that took 10s and still promoted ~1GB,
then you know that something abnormal happened during the couple outliers. To know what happened during those GCs, you can use the following perfview commandline:

perfview /ClrEvents=default-stack-GCHeapSurvivalAndMovement /StopOnGCOverMSec:1000 /DelayAfterTriggerSec:0 /CircularMB:2000 /CollectMultiple:3 /NoGUI /NoNGENRundown collect

(the /ClrEvents excludes a couple of things that would make the GC time skewed. This says to stop the trace if it detects a GC that’s >1s, 
and collect 3 such traces. Since this is not meant to be a generic tutorial of perfview, 
I will not go into details for the commandline args – for the explanation of the rest of the commandline args please search for them in Help\Command Line Help in perfview)

Then what you can do is to look at the GCStats view and get the start and end of those unusually long GCs and open up the CPU stack view and look at the call stacks 
between the start and end timestamp of those GCs. This usually will point to the culprit. However that’s not always the case. 
Sometimes you will see the CPU is simply not being used much (by GC or anything else), in this case you will want to do a ThreadTime trace 
(ie, add /ThreadTime to the above commandline if CPU is not the problem – be aware that turning on /ThreadTime will increase the event volume by a huge amount, 
in which case /CircularMB:2000 might not be enough so you’ll want to increase it to 5000 (or even more if needed) and look at the thread time view to see what’s going on.

Occasionally you might see that the Suspend MSec column shows exceptionally large numbers (again, it’s easy to detect the outliers – most of time you’ll see less than 1ms for this column) 
and since GC hasn’t started at this point it can’t be due to the GC work therefore must be something else. Generally I would only suggest to look at it if you see it happens often enough, 
eg, if you occasionally see it less than, say 30ms I wouldn’t worry about it. But if you see it being longer, and happen often enough that it disturbs your latency on a noticeable basis, 
then you would certainly want to try out the above commandline. In one case I worked on, the customer was seeing this column being in hundreds of ms or even >1s for 10+% GCs. 
We tracked it down to a component they were running on their machines (but were unware of it before that) that constantly changed thread priorities. 
After they removed that component this problem went away.

Finalizable Surv MB and Pinned Obj

These are the same value as what you get with the “Promoted Finalization-Memory from Gen 0” and “# of Pinned Objects” .NET CLR Memory perf counter respectively. 
See GC Performance Counters for an explanation if you need to.

Edited on 03/01/2020 to add the indices to GC ETW Event blog entries

https://devblogs.microsoft.com/dotnet/gc-etw-events-4/

在上一篇博客中，我解释了GCStats中的Suspend MSec和Pause MSec列以及它们是如何计算的。在这篇博客中，我想谈谈其他几列。

GenX MB、GenX Survival Rate % 和 GenX Frag %
首先，这些数据是在GC结束时计算的。GenX MB列包括该代的碎片。GenX Survival Rate %（存活率）的计算公式是：（从该代存活下来的字节数 / 该代在回收前对象占用的字节数）。
例如，假设某代大小为4MB，碎片为1MB，那么对象占用的字节数为3MB。如果这3MB中有1.5MB存活下来，那么存活率就是50%。如果在回收后，该代大小为2MB，碎片为0.5MB，
那么碎片率（Frag %）就是25%。

你有时会注意到GenX Survival Rate %列显示为NaN。这是因为我们没有回收该代，因此讨论该代的存活率没有意义。例如，
如果我们正在进行gen1 GC，gen2和LOH（大对象堆）的存活率就会显示为NaN。

如果你正在使用固定（pinning）操作（或者你使用的库代表你进行固定操作，例如IO），你通常会看到gen0的碎片率非常大，有时甚至达到100%（100%并不意味着gen0中只有碎片，
这只是因为我们的计算方式是：gen0中的碎片MB / gen0的大小MB，而它们的差异足够小，以至于gen0中的碎片MB与gen0的大小MB相同）。我们尽量保留gen0中的碎片，
因为它可以立即用于满足分配请求。

后台GC（Background GC，BGC）不会进行压缩，因此在进行BGC后，你看到gen2的碎片率相当大（即>50%）并不奇怪。如果我们检测到gen2中有大量无法有效使用的碎片，
我们会进行一次压缩的gen2 GC。

除非明确要求，否则我们不会压缩LOH。因此，如果LOH的碎片率仍然很大，这意味着我们无法有效利用LOH上的大量空闲空间。如果LOH占用了堆的很大一部分，那么值得优化LOH的使用。

Promoted MB（提升的MB）
这是GC期间提升的字节数。例如，如果我们正在进行gen1 GC，这就是从gen0和gen1提升的字节数。你可以想象，由于gen0和gen1通常只占堆的一小部分，
临时GC的提升字节数通常也比你看到的gen2 GC的提升字节数要小得多。正如我之前提到的，GC的暂停时间与GC存活的对象量成正比，
换句话说，与Promoted MB列中显示的数据成正比（当然，后台GC是个例外，因为它专门设计为最小化暂停时间）。

一个判断是否存在异常的好方法是查看GC的暂停时间与其提升的内存量的关系。如果提升的内存量很小，但暂停时间却很长，那么几乎可以肯定发生了与GC工作无关的事情。
那么什么是“小”呢？一种方法是与其他GC进行比较。假设你所有的完整阻塞GC都提升了约1GB，并花费了1秒，但有几个GC花费了10秒，仍然提升了约1GB，
那么你就知道在这几个异常GC期间发生了不正常的事情。要了解这些GC期间发生了什么，你可以使用以下PerfView命令行：

bash
复制
perfview /ClrEvents=default-stack-GCHeapSurvivalAndMovement /StopOnGCOverMSec:1000 /DelayAfterTriggerSec:0 /CircularMB:2000 /CollectMultiple:3 /NoGUI /NoNGENRundown collect
（/ClrEvents排除了会使GC时间偏差的几个因素。这个命令表示如果检测到GC时间超过1秒，就停止跟踪，并收集3个这样的跟踪。由于这不是一个通用的PerfView教程，
我不会详细解释命令行参数 —— 有关其他命令行参数的解释，请在PerfView的Help\Command Line Help中搜索。）

然后，你可以查看GCStats视图，获取那些异常长的GC的开始和结束时间，并打开CPU堆栈视图，查看这些GC的开始和结束时间戳之间的调用堆栈。这通常会指向问题的根源。
然而，情况并非总是如此。有时你会看到CPU根本没有被大量使用（无论是GC还是其他操作），在这种情况下，你需要进行线程时间跟踪（即在上述命令行中添加/ThreadTime，
如果CPU不是问题的话 —— 请注意，启用/ThreadTime会大幅增加事件量，此时/CircularMB:2000可能不够，你可能需要将其增加到5000甚至更多），
并查看线程时间视图以了解发生了什么。


偶尔，你可能会看到Suspend MSec列显示异常大的数字（同样，检测异常值很容易 —— 大多数时候你会看到该列的值小于1ms），由于此时GC尚未开始，因此这不可能是由于GC工作引起的，
一定是其他原因。一般来说，我建议只有在它频繁发生时才去关注它。例如，如果你偶尔看到它小于30ms，我不会担心。但如果你看到它更长，并且频繁发生以至于明显影响了你的延迟，
那么你肯定需要尝试上述命令行。在我处理的一个案例中，客户看到该列的值在数百毫秒甚至超过1秒，且发生在10%以上的GC中。
我们最终追踪到他们机器上运行的一个组件（之前他们并不知道）不断更改线程优先级。在他们移除该组件后，问题消失了。

Finalizable Surv MB 和 Pinned Obj
这些值与“.NET CLR Memory”性能计数器中的“Promoted Finalization-Memory from Gen 0”和“# of Pinned Objects”相同。如果你需要解释，请参阅GC性能计数器的文档。

2020年3月1日编辑，添加了GC ETW事件博客条目的索引。

