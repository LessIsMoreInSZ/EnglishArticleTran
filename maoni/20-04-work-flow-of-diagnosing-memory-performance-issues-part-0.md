<h1>Work flow of diagnosing memory performance issues – Part 0</h1>

I wanted to describe what I do to diagnose memory perf issues, or rather the common part of various work flows of doing such diagnostics. Diagnosing performance issues can take many forms because there’s no fixed steps you follow. But I’ll try to break it down into basic blocks that get invoked for a variety of diagnostics.

This part is for beginners so if you’ve been doing memory perf analysis for a while you can safely skip it.

First and foremost, before we talk about the actual diagnostics part, it really pays to know a few high level things that can point you in the right directions.

1) Point-in-time vs histogram

Understanding that memory issues are often not point-in-time is very important. Memory issues usually don’t just suddenly come into the picture – it might take a while for one to accumulate to the point that’s noticeable.

Let’s take a simple example, for a very simple non generational GC that only does blocking GCs that compact, this is still the case. If you are freshly out of a GC, of course the heap is at its smallest point. If you happen to measure at that point, you’ll think “great; my heap is small”. But if you happen to measure right before the next GC, the heap might be much bigger and you will have a different perception. And this is just for a simple GC, imagine what happens when you have a generational GC, or a concurrent GC.

This is why it’s extremely important to understand the GC history to see how GC made the decisions and how the decisions led to the current situation.

Unfortunately many memory tools, or many diagnostics approaches, do not take this into consideration. The way they do memory diagnostics is “let me show you what the heap looks like at the point you happened to ask”. This is often not helpful and sometimes to the point that it’s completely misleading and wasting people’s time to chase a problem that doesn’t exist or have a totally wrong approach how to make progress on the problem. This is not to say tools like these are not helpful at all – they can be helpful when the problem is simple. If you have a dramatic memory leak that’s been going on for a while and you used a tool that shows you the heap at that point (either by taking a process dump and using sos, or by another tool that dumps the heap) it’s probably really obvious what the leak is.

2) Generational GC

By design generational GCs don’t collect the whole heap every time a GC is triggered. They try to do young gen GCs much more often than old gen ones. Old gen GCs are often much more costly. With concurrent old gen GCs, the STW pauses may not be long but GC still needs to spend CPU cycles to do its job.

This also makes looking at the heap much more complicated because if you are fresh out of an old gen GC, especially a compacting one, you obviously have a potentially way smaller heap size than if you were right before that GC is triggered; but if you look at young gen GCs, they could be compacting but the difference is heap size isn’t as much and that’s by design.

3) Compacting vs sweeping

Sweeping is not supposed to change the heap size by much. In our implementation we still give up the space at the end of segments so the total heap size can become a bit smaller but as high level you can think of the total heap size as not changing but free spaces get built up in order to accommodate the allocations from a younger gen (or in gen0/LOH case user allocations).

So if you see 2 gen2 GCs, one is compacting and the other is sweeping, it’s expected if the compacting one comes out with a much smaller heap size and the other one with high fragmentation (by design as that’s the free list we built up).

4) Allocation and survival

While many memory tools report allocations, it’s not just allocations that cost. Sure, allocations can trigger GCs, and that’s definitely a cost but when GC is working, the cost is mostly dominated by survivals. Of course you cannot be in a situation that both your allocation rate and survival rate are very high – you’d just run out of memory very quickly.

5) “Mainline GC scenario” vs “not mainline”

If you had a program that just used the stack and created some objects to use, GC has been optimizing that for years and years. Basically “scan stacks to get the roots and handle the objects from there”. This is the mainline GC scenario that many GC papers assume as the only scenario. Of course as a commercial product that has existed for decades and having to accommodate various customer requests, we have a bunch of other things like GC handles and finalizers. The important thing to understand there is while over the years we also optimized for those, we operate based on assumptions that “there aren’t too many of those” which obviously is not true for everyone. So if you do have many of those, it’s worth looking at if you are diagnosing a memory problem. In other words, if you don’t have any memory problem, you don’t need to care; but if you do (eg, high % time in GC), they are good things to suspect.

All this info is expressed in ETW events or the equivalent on Linux – this is why for years we’ve been investing in them and the tooling for analyzing the traces.

Traces to capture to start with

I often ask for 2 traces to start with. The 1st one is to get the accurate GC timing:

perfview /GCCollectOnly /nogui collect

after you are done, press s in the perfview cmd window to stop it

This should be run long enough to capture enough GC activities, eg, if you know problems occur at times, this should cover time that leads up to when problems happen (not only during problematic time).

If you know how long to run it for you can do (this is used much more often actually) –

perfview /GCCollectOnly /nogui /MaxCollectSec:1800 collect

replace 1800 (half an hour) with however many seconds you need.

This collects the informational level of GC events and just enough OS events to decode the process names. This command is very lightweight so it can be on all the time.

Notice I have the /nogui in all the PerfView commandlines I give out. PerfView does have a UI for event collection that allows you to select the events you want to capture. Personally I never use it (after I used it a couple of times when I first started to use PerfView). Some of it is just because I’m much more a commandline person; the other (important) part is because commandlines allow for much more flexibility and are a lot more automation friendly.

After you collect the trace you can open it in PerfView and look at the GCStats view. Some folks tend to just send it to me after they are done collecting but I would really encourage everyone who needs to do memory diagnostics on a regular basis to learn to read this view ’cause it’s very useful. It gives us a wealth of information, even though the trace so lightweight. And if this doesn’t get us to the root cause, it definitely points at the direction we should take to make more progress. I described some of this view in this blog entry and its sequels that are linked in the entry. So I’m not going to show more pictures here. You could easily open that view and see for yourself.

Examples of the type of issues that can be easily spotted with this view –

Very high “% Time paused for garbage collection”. Unless you are doing some microbenchmarking and specifically testing allocation perf (like many GC benchmarks), you should not see this as higher than a few percent. If you do that’s something to investigate. Below are things that can contribute to this percentage significantly.

Individual GCs with unusually long pauses. Is a 60s GC really long? Yes you bet it is! And this is usually largely not due to GC work. From my experience it’s always due to something interfering with the GC threads.

Excessively induced GCs (high ratio of (# of induced GCs / total # of GCs), especially when the induced GCs are gen2s.

Excessive # of gen2 GCs – gen2 are costly especially when you have a large heap. Even though with BGC, most of its work is done concurrently, it’s still CPU cycles spent so if you have every other GC as gen2, that usually immediately points at a problem. One obvious case is most of them are triggered with the AllocLarge trigger reason. Again, there are cases where this is necessarily not a problem, for example if most of your heap is LOH and you are not running inside a container, which means LOH is not compacted by default, in that case doing gen2s just sweeps the LOH and that’s pretty quick.

Long suspension issues – suspension usually should take much less than 1ms, if it takes 10s of ms that’s a problem; if it takes hundreds of ms, that’s definitely a problem.

Excessive # of pinned handles – in general a few pinned handles are ok but if you see hundreds, that’s a cause for concern, especially if they are during ephemeral GCs; if you see thousands, usually it’s telling you to go investigate.

Those are just things you can see at a glance. If you dig a little deeper there are many more things. And we’ll talk about them next time.

https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-0/

我想描述一下我如何诊断内存性能问题，或者更确切地说，诊断这类问题的通用部分。诊断性能问题的形式多种多样，因为没有固定的步骤可以遵循。但我会尝试将其分解为一些基本模块，这些模块适用于各种诊断场景。

这一部分内容是为初学者准备的，如果你已经从事内存性能分析一段时间了，可以跳过这部分。

首先，在我们讨论实际诊断之前，了解一些高层次的概念是非常有帮助的，它们可以指引你朝着正确的方向前进。

---

### 1) 时间点 vs 直方图

理解内存问题通常不是时间点的问题非常重要。内存问题通常不会突然出现——它们可能需要一段时间积累到显著的程度。

举个简单的例子，对于一个非常简单的非代际垃圾回收器（只执行阻塞式且压缩的 GC），情况依然如此。如果你刚刚完成了一次 GC，堆显然处于最小状态。如果你恰好在这个时间点测量，你会认为“太棒了，我的堆很小”。但如果恰好在下一次 GC 前测量，堆可能会大得多，你的看法也会不同。这只是针对简单 GC 的情况，想象一下当你使用代际 GC 或并发 GC 时会发生什么。

这就是为什么理解 GC 历史非常重要，它能帮助你了解 GC 是如何做出决策的，以及这些决策是如何导致当前状况的。

不幸的是，许多内存工具或诊断方法并未考虑这一点。它们进行内存诊断的方式是“让我展示你在某个时间点上堆的样子”。这种方式往往没有帮助，有时甚至会完全误导人，浪费时间去追踪不存在的问题或采取完全错误的解决方法。这并不是说这些工具完全没有用——当问题是简单的时候，它们确实有帮助。如果你有一个持续已久的严重内存泄漏，并且使用了一个工具（例如通过进程转储并使用 SOS，或其他工具转储堆）来查看该时间点的堆，问题可能非常明显。

---

### 2) 代际 GC

按设计，代际 GC 并不会每次触发 GC 时收集整个堆。它们试图更频繁地执行年轻代 GC 而非老年代 GC。老年代 GC 通常成本更高。即使使用并发的老年代 GC，STW（Stop-the-World）暂停时间可能不长，但 GC 仍然需要消耗 CPU 周期来完成其任务。

这也使得观察堆变得更加复杂，因为如果你刚刚完成了一次老年代 GC，尤其是压缩型的 GC，堆大小显然可能比 GC 触发前小得多；但如果你观察年轻代 GC，它们可能是压缩型的，但堆大小的变化并不明显，这是设计使然。

---

### 3) 压缩 vs 清扫

清扫不应该显著改变堆大小。在我们的实现中，我们仍然会在段末释放空间，因此总堆大小可能会稍微变小，但从高层次来看，你可以认为总堆大小不变，而空闲空间被建立起来以适应年轻代（或在 Gen0/LOH 情况下的用户分配）。

因此，如果你看到两次 Gen2 GC，一个是压缩型的，另一个是清扫型的，那么压缩型的堆大小显著较小、而清扫型的堆碎片化较高是预期的行为（这是设计使然，因为那是我们建立的空闲列表）。

---

### 4) 分配与存活

虽然许多内存工具报告分配情况，但分配本身并不是唯一的成本来源。当然，分配可能会触发 GC，这无疑是一种成本，但在 GC 工作时，成本主要由存活对象主导。当然，你不可能处于分配率和存活率都很高的情况——否则你会很快耗尽内存。

---

### 5) “主流 GC 场景” vs “非主流”

如果你的程序只是使用栈并创建了一些对象供使用，GC 多年来一直在优化这种情况。基本上就是“扫描栈以获取根对象，并从那里处理对象”。这是许多 GC 论文假设的唯一场景。然而，作为一个存在了几十年的商业产品，并且需要满足各种客户请求，我们还有许多其他功能，例如 GC 句柄和终结器。重要的是要理解，虽然多年来我们也对这些进行了优化，但我们基于“这些数量不多”的假设进行操作，这显然并非适用于所有人。因此，如果你确实有很多这样的内容，并且正在诊断内存问题，值得检查这些方面。换句话说，如果你没有内存问题，不需要关心这些；但如果你有（例如，GC 占用了很高的时间比例），这些是值得怀疑的地方。

所有这些信息都通过 ETW 事件或 Linux 上的等效事件表达出来——这就是为什么多年来我们一直在投资于这些事件及其分析工具。

---

### 需要捕获的跟踪数据

我通常会要求提供两种跟踪数据作为起点。第一个是为了获取准确的 GC 时间：

```bash
perfview /GCCollectOnly /nogui collect
```

完成后，在 PerfView 命令窗口中按下 `s` 来停止它。

这个跟踪应该运行足够长的时间以捕获足够的 GC 活动，例如，如果你知道问题发生的时间，这个跟踪应该覆盖问题发生前的时间（不仅仅是问题发生期间的时间）。

如果你知道需要运行多长时间，可以这样做（实际上这种方法用得更多）：

```bash
perfview /GCCollectOnly /nogui /MaxCollectSec:1800 collect
```

将 `1800`（半小时）替换为你需要的秒数。

这会收集 GC 事件的信息级别以及解码进程名称所需的最少 OS 事件。这个命令非常轻量级，可以一直运行。

注意，我在所有的 PerfView 命令行中都添加了 `/nogui`。PerfView 确实有一个用于事件收集的 UI，允许你选择要捕获的事件。我个人从未使用过它（除了刚开始使用 PerfView 时尝试过几次）。一部分原因是我是命令行爱好者；另一部分（更重要）的原因是命令行提供了更大的灵活性，并且更适合自动化。

收集完跟踪数据后，你可以在 PerfView 中打开它并查看 `GCStats` 视图。有些人倾向于直接将其发送给我，但我强烈建议任何需要定期进行内存诊断的人都学会阅读这个视图，因为它非常有用。尽管跟踪数据非常轻量级，但它提供了大量信息。即使无法直接找到根本原因，它也能明确指出我们应该采取的方向以进一步推进诊断。我在这篇博客文章及其后续文章中描述了这个视图的一些内容，因此这里不再展示更多图片。你可以轻松打开该视图并自行查看。

通过这个视图可以轻松发现以下类型的问题：

- **极高的“GC 暂停时间百分比”**：除非你在进行微基准测试并专门测试分配性能（如许多 GC 基准测试），否则你不应该看到这个百分比高于几个百分点。如果确实如此，那是一个需要调查的问题。以下是可能导致此百分比显著增加的因素：
  
- **单个 GC 的异常长暂停**：60 秒的 GC 是否很长？当然！而且这通常不是由于 GC 工作本身。根据我的经验，这总是由于某些因素干扰了 GC 线程。

- **过度诱导的 GC**（高比例的“诱导 GC 数量 / 总 GC 数量”，尤其是当诱导的 GC 是 Gen2 时）。

- **过多的 Gen2 GC**：Gen2 成本很高，尤其是在堆很大的情况下。即使使用 BGC（后台 GC），它的大部分工作是并发完成的，但仍然是 CPU 周期的消耗。如果你每两次 GC 中有一次是 Gen2，这通常立即指向一个问题。一个明显的案例是大多数 Gen2 GC 都是由 `AllocLarge` 触发的。当然，也有一些情况这不是问题，例如如果你的堆主要由 LOH 组成，并且你不在容器内运行，则 LOH 默认不会被压缩，在这种情况下执行 Gen2 GC 只是清扫 LOH，速度很快。

- **长时间的挂起问题**：挂起通常应该小于 1 毫秒，如果达到几十毫秒，这是一个问题；如果达到几百毫秒，那肯定是一个问题。

- **过多的固定句柄**：一般来说，少量固定句柄是可以接受的，但如果你看到数百个，这就值得关注，尤其是如果它们出现在短暂代 GC 期间；如果你看到数千个，通常这意味着你需要去调查。

这些都是你一眼就能看到的内容。如果你深入挖掘，还有更多内容。我们下次再详细讨论。