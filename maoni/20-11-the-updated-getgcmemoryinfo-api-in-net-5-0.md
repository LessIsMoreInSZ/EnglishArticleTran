<h1>The updated GetGCMemoryInfo API in .NET 5.0 and how it can help you</h1>

A bit of history
In .NET 3.0 we introduced a GC.GetGCMemoryInfo API for library code to get memory load related things (this was used in ArrayPool for example) so it exposed things library folks wanted at the time. In 5.0 I got requests from folks to monitor more things about the GC. Instead of adding a bit of info each time someone asks, I really thought about the kinds of things that would help with monitoring and diagnostics and expanded the info provided by this API significantly. It also has a new overload, documented here. The returned GCMemoryInfo struct has many more properties.

Goals of the updated API
You can view the new GetGCMemoryInfo API as a rich sampling method. Many people enjoyed using perf counters in the old days. The problem with perf counters was

The info was pretty primitive
It's hard to correlate the info for the same GC since you get counters individually.
2) is less of a problem just because most of the time folks didn't care about individual GCs, they just cared about a trend. And counters were really just used for monitoring, not for diagnostics 'cause they simply didn't provide enough info to look at the GCs in any kind of depth. I wanted the new API to be used for both. There was quite a bit of discussion on the API proposal and with help from other people on my team I think I landed at an API that I'm quite happy about. Big kudos to Noah for suggesting to take the kind of GC as a parameter which makes it still very useful for understanding what's going on even when you are monitoring with very long intervals which was a big problem with perf counters – you get whatever values the last GC happened to set which meant if you only sample once in a long time (like once every minute), it's based on luck whether you catch a long GC pause 'cause that GC could easily not be the last GC that interval caught. By specifying the GC kind if you do sample only once in a long time, you could always get the last full blocking GC (GCKind.FullBlocking) which has the longest GC pause because you can specific ask about that kind of GCs.

Some implementation detail
Since I know many people who read my blog are curious about implementation details I wanted to mention some about this API implementation. You might think this should be very straightforward to implement but as with most things in GC it's rarely straightforward 😅. When you call the API, you get back an instance of the GCMemoryInfo type and all fields of this instance are for the last completed GC, or the last completed GC of the GCKind you specify. It's never the case that some values in this instance belong to a GC and others belong to a different GC. And this is easy to get when we do a blocking GC because user code is not running at the same time. At the end of that blocking GC we fill in all fields for that GC and then resume user code so it's easily guaranteed that all fields are for the same GC. But it's more complicated, as usual, for a background GC. What I did was to maintain an array of 2 element of BGC info and when a BGC starts, it flips the index to !index. This is done while the user threads are suspended so there's no synchronization issues. And this is the index the BGC will log its info to. If the user code calls the API asking for the last BGC while a BGC is in progress, it returns the other index which we know is for the last completed BGC (or all 0s, if the current BGC is the very first BGC that's happening). And if no BGC is in progress, it returns what the index points to because we know that means that BGC was complete and is the last BGC.

Another somewhat tricky thing about this API was the interaction between BCL and the runtime (I know that now BCL is also in the runtime repo; I'm still used to using runtime to refer to just the native part which is what's in coreclr.dll). I mostly work only with native code in the runtime and rarely step onto the BCL side since GC has such a small public surface area. And usually the BCL part of a GC API is pretty straightforward but this API was an exception. If you look at the calls that the BCL makes into the runtime, they usually take very simple arguments which means they are easy to pass back and forth between managed and native code. This API was not like that – it passes a fairly complicated structure which has both array fields and non array fields. At first I was doing some complicated things on the native side to fill in the arrays because I'm pretty inept with the managed side of things and I was not happy about that. So I enlisted Jan's help and he found a much more elegant way of doing what I wanted to do which I went with. The implementation of this was split between GCMemoryInfo.cs, GC.cs on the managed side and comutilnative.cpp on the native side. I would imagine very, very few people ever need to do this but if you ever do, this would serve as a very good example.

Most of the properties in the GCMemoryInfo struct are pretty self explanatory. The only one that might cause confusion is the Index. Before any GCs happen, if you call the API it will get back 0 for Index. The first GC's index will be 1, 2nd one will be 2 and so on. You can also correlate this number with what you get back at GC.CollectionCount which is cheaper since we just get back a number whereas GetGCMemoryInfo needs to fill out many fields. If the first GC is a gen0 GC, GC.CollectionCount(0) would return 1, which is the same as GCMemoryInfo.Index you get back when you call GetGCMemoryInfo() or GetGCMemoryInfo(GCKind.Ephemeral). Of course since we are doing all in managed code, it means a GC can happen between your calling CollectionCount and GetGCMemoryInfo because other threads could have triggered a GC or you are under memory pressure but the chance of that actually happening is quite small. Note that an attribute of CollectionCount is when we increase a higher generation's collection count, all its lower generations' counts are also increased. So if a gen1 GC happened at the very beginning, CollectionCount(0) and CollectionCount(1) would both return 1.

What can you use the info for?
The returned GCMemoryInfo provides a wealth of info about that GC such as the generation, whether it was compacting or not, whether it was concurrent or not, how many pinned objects that GC observed, pause duration(s) and etc. Basically you could think of this as the combination of the detailed info you get from GC ETW events. Since folks have a variety of ways to do monitoring and diagnostics this just made it easier if you do in-proc monitoring and want to get GC info (eg, if you already have a thread that does monitoring you can totally beef it up by calling this API for memory monitoring). I can easily think of many things to do with the info, below are just some examples –

1) I can't tell you how many times people have asked "the GC heap size is X, but my process's usage is a lot higher than that! Why is this?". It turned out that they were simply not measuring the peak size. They were either measuring the size right after a GC (which would be the smallest size) or a size at a random point (which could be fresh out of a GC and very close to the smallest size). This is explained the How to look at the GC heap size properly section of mem-doc. ETW, being the richest info we provide, of course gives you this data but I have seen folks misinterpreting what the ETW event tells them. Recently I had a customer who told me that they didn't think hardlimit they specified was working because the GC heap remained very small even after they increase the hardlimit. This was because they were looking at fields of the GCHeapStats event which reports the "right after a GC" value. Granted, we could do a better job at documenting these events. But I think the GetGCMemoryInfo API makes it very explicit what you are getting – GCGenerationInfo explicitly says SizeBeforeBytes which is the size on entry of a GC which denotes the peak size –


public readonly struct GCGenerationInfo
{
    /// <summary>Size in bytes on entry to the reported collection.</summary>
    public long SizeBeforeBytes{ get; }
    
    /// <summary>Fragmentation in bytes on entry to the reported collection.</summary>
    public long FragmentationBeforeBytes{ get; }
    
    /// <summary>Size in bytes on exit from the reported collection.</summary>
    public long SizeAfterBytes{ get; }
    
    /// <summary>Fragmentation in bytes on exit from the reported collection.</summary>
    public long FragmentationAfterBytes{ get; }
}
view rawGenInfo.cs hosted with ❤ by GitHub
The same info is provided as part of the GCPerHeapHistoryGenData in TraceEvent but in more detail – we separate the fragmentation into FreeListSpace and FreeObjSpace – the latter is for free objects that are too small to get on the free list.

2) Another question I get frequently is "I see there's all this free space in GC, why isn't GC using it?". Again, this is very much affected by when you measure. If you measure right after a BGC, since its job is to build up free space you'd likely see a ton of free space in gen2/LOH. So with the API you get the fragmentation (which is the total amount of free space) before and after a GC so you know if GC is using the free space or not. For example if you have a lot of long lived pins that are scattered in gen0 (since we demoted them), the FragmentationAfterBytes could be quite big 'cause there's a lot of free space in-between the pins. But you should see FragmentationBeforeBytesto be much smaller as we'd have used the free space to satisfy your allocation requests.

3) You could monitor quite infrequently by calling this API with GCKind.FullBlocking which tells you if a full blocking GC has happened if you want to make they don't happen since Full blocking GCs could introduce very long pauses.


4) The "% time in GC" perf counter was only for the last GC (and not accurate if a BGC happened). For long term monitoring, it's more useful to know what the % Pause time in GC is so far. So the PauseTimePercentage provided by the API is exactly that so it's very suitable for long term monitoring. And if you want to know individual GC's pause duration, you can always look at the PauseDurations property which provides 2 durations – only the first one is used for blocking GCs since they only have one GC pause. For BGCs there are 2 durations since each BGC will pause twice – one is the initial pause and the 2nd one is at the end of the mark phase (described in the GC event sequence section in mem-doc). And you can get each of these pause durations. This is actually more detailed than what TraceEvent provides which combines these 2 pauses into 1 and reports that and you see this in the Pause MSec column in the GCStats view in PerfView.

5) I talked about just how important it is to be aware of the generational aspect of the GC in mem-doc. Unfortunately so few tools actually expose this generational aspect. Even some seasoned perf engineers are not aware of it at all. This API provides the PromotedBytes for each GC and you can clearly see the promoted bytes for ephemeral GCs vs full GCs. For example, you can correlate the PauseDurations[0] of ephemeral GCs with their PromotedBytes. And if you see that they totally don't correlate, eg, the pause durations vary greatly even though the promoted remained stable, you might be hitting the stolen CPU problem if you are using Server GC.

### 一点历史背景

在 .NET 3.0 中，我们引入了一个名为 `GC.GetGCMemoryInfo` 的 API，用于库代码获取与内存负载相关的信息（例如，这个 API 在 ArrayPool 中被使用），因此它暴露了一些当时库开发者希望获取的内容。到了 .NET 5.0，我收到了来自许多开发者的请求，希望监控更多关于 GC 的信息。与其每次有人提出需求就添加一些信息，我更深入地思考了哪些内容有助于监控和诊断，并显著扩展了该 API 提供的信息。它还新增了一个重载方法，具体文档可以参考这里。返回的 `GCMemoryInfo` 结构体包含了许多新的属性。

### 更新 API 的目标

你可以将新的 `GetGCMemoryInfo` API 视为一种丰富的采样方法。过去很多人喜欢使用性能计数器。然而，性能计数器的问题在于：

1. **信息非常基础**  
2. **很难将同一 GC 的信息关联起来，因为计数器是单独获取的**

第 2 点之所以不是大问题，是因为大多数时候开发者并不关心单个 GC，他们只关心趋势。而计数器通常只用于监控，而不是诊断，因为它们提供的信息不足以深入分析 GC。我希望新的 API 能同时满足监控和诊断的需求。在 API 提案中进行了大量的讨论，在团队其他成员的帮助下，我认为最终设计出了一个令我满意的 API。特别感谢 Noah 建议将 GC 类型作为参数传递，这使得即使在以很长的时间间隔进行监控时，API 仍然非常有用——性能计数器的一个大问题是，你只能获取最后一个 GC 设置的值，这意味着如果你每隔很长时间采样一次（例如每分钟一次），是否能捕获到长时间的 GC 暂停完全取决于运气，因为那个 GC 可能并不是区间内最后一个 GC。通过指定 GC 类型（如 `GCKind.FullBlocking`），即使采样频率很低，你也可以始终获取最后一个完整的阻塞 GC，因为它具有最长的暂停时间。

---

### 实现细节

由于我知道很多读者对实现细节感兴趣，我想提一下这个 API 的实现。你可能会认为实现这个 API 应该很简单，但在 GC 领域，事情很少是简单的 😅。当你调用这个 API 时，你会得到一个 `GCMemoryInfo` 类型的实例，其中的所有字段都属于最后一个完成的 GC，或者属于你指定类型的最后一个完成的 GC。绝不会出现某个字段属于一个 GC 而另一个字段属于另一个 GC 的情况。对于阻塞 GC 来说，这是很容易实现的，因为在阻塞 GC 期间用户代码不会运行。在阻塞 GC 结束时，我们会填充所有字段，然后恢复用户代码，这样可以轻松保证所有字段都属于同一个 GC。但对于后台 GC（BGC），情况则复杂得多。

我的做法是维护一个包含两个元素的 BGC 信息数组，当一个 BGC 开始时，索引会在 `!index` 之间切换。这一操作是在用户线程被挂起时完成的，因此不存在同步问题。这就是 BGC 记录其信息的索引。如果用户代码在 BGC 进行中调用 API 请求最后一个 BGC 的信息，它会返回另一个索引，因为我们知道那个索引对应的是最后一个完成的 BGC（如果当前 BGC 是第一个 BGC，则返回全零）。如果没有 BGC 正在进行，则返回当前索引指向的内容，因为我们知道这意味着该 BGC 已完成且是最后一个 BGC。

另一个稍微棘手的地方是 BCL 和运行时之间的交互（我知道现在 BCL 也在运行时仓库中；我还是习惯用“运行时”来指代核心部分，也就是 `coreclr.dll` 中的内容）。我主要处理运行时中的原生代码，很少涉足 BCL，因为 GC 的公共接口非常小。通常 GC API 的 BCL 部分都很简单，但这次是个例外。如果你查看 BCL 调用运行时的代码，通常它们传递的参数都非常简单，这意味着它们在托管代码和原生代码之间传递起来很容易。但这个 API 不同——它传递了一个相当复杂的结构，包含数组字段和非数组字段。一开始我在原生代码侧做了一些复杂的事情来填充数组，因为我对托管代码不太熟悉，而且我对这种实现方式并不满意。于是我寻求了 Jan 的帮助，他找到了一种更优雅的方式来实现我想要的功能，所以我采用了他的方案。这个实现分布在托管代码侧的 `GCMemoryInfo.cs` 和 `GC.cs` 文件以及原生代码侧的 `comutilnative.cpp` 文件中。我想象只有极少数人需要这样做，但如果需要的话，这是一个非常好的示例。

`GCMemoryInfo` 结构体中的大多数属性都很直观。唯一可能引起混淆的是 `Index`。在任何 GC 发生之前，如果你调用这个 API，它会返回 `Index = 0`。第一个 GC 的索引将是 1，第二个是 2，依此类推。你还可以将这个数字与 `GC.CollectionCount` 返回的结果进行关联，后者成本更低，因为我们只需要返回一个数字，而 `GetGCMemoryInfo` 需要填充许多字段。如果第一个 GC 是 gen0 GC，那么 `GC.CollectionCount(0)` 将返回 1，这与调用 `GetGCMemoryInfo()` 或 `GetGCMemoryInfo(GCKind.Ephemeral)` 返回的 `GCMemoryInfo.Index` 相同。当然，由于我们都是在托管代码中操作，所以在调用 `CollectionCount` 和 `GetGCMemoryInfo` 之间可能会发生 GC，因为其他线程可能触发了 GC 或系统处于内存压力下，但实际上这种情况发生的概率很小。需要注意的是，`CollectionCount` 的一个特性是：当我们增加更高代别的收集计数时，较低代别的计数也会随之增加。因此，如果一开始就发生了 gen1 GC，`CollectionCount(0)` 和 `CollectionCount(1)` 都将返回 1。

---

### 你能用这些信息做什么？

`GCMemoryInfo` 提供了大量关于 GC 的信息，例如代别、是否进行了压缩、是否为并发 GC、观察到的固定对象数量、暂停时长等。基本上可以将其视为从 GC ETW 事件中获取的详细信息的组合。由于开发者有多种方式进行监控和诊断，因此如果你需要进行进程内监控并希望获取 GC 信息（例如，如果你已经有一个执行监控的线程，可以通过调用此 API 加强内存监控），这个 API 让这一切变得更加容易。我可以想到许多利用这些信息的方法，以下是一些例子：

1. **为什么我的进程内存使用量比 GC 堆大小高得多？**  
   很多人问过这个问题。事实证明，他们只是没有测量峰值大小。他们要么在 GC 后立即测量（这是最小的大小），要么在随机时刻测量（这可能是刚刚完成 GC 后的大小，接近最小值）。这在 mem-doc 的“如何正确查看 GC 堆大小”部分中有解释。尽管 ETW 提供了最丰富的信息，但我看到有些人错误解读了 ETW 事件。最近，一位客户告诉我，他们认为设置的硬限制不起作用，因为即使增加了硬限制，GC 堆仍然保持很小。这是因为他们在查看 `GCHeapStats` 事件的字段，这些字段报告的是“GC 后”的值。诚然，我们可以更好地记录这些事件。但我认为 `GetGCMemoryInfo` API 明确说明了你获得的内容——`GCGenerationInfo` 明确指出 `SizeBeforeBytes` 是进入 GC 时的大小，表示峰值大小。

```csharp
public readonly struct GCGenerationInfo
{
    /// <summary>进入收集时的字节大小。</summary>
    public long SizeBeforeBytes { get; }
    
    /// <summary>进入收集时的碎片化字节数。</summary>
    public long FragmentationBeforeBytes { get; }
    
    /// <summary>退出收集时的字节大小。</summary>
    public long SizeAfterBytes { get; }
    
    /// <summary>退出收集时的碎片化字节数。</summary>
    public long FragmentationAfterBytes { get; }
}
```

同样的信息也作为 `GCPerHeapHistoryGenData` 的一部分提供在 TraceEvent 中，但更加详细——我们将碎片化分为 `FreeListSpace` 和 `FreeObjSpace`，后者是指太小而无法进入空闲列表的空闲对象。

2. **为什么 GC 不使用这些空闲空间？**  
   这同样受到测量时间的影响。如果你在 BGC 后立即测量，由于它的任务是构建空闲空间，你可能会看到大量的空闲空间出现在 gen2 或 LOH 中。通过这个 API，你可以获取 GC 前后的碎片化信息（即总的空闲空间），从而判断 GC 是否使用了这些空闲空间。例如，如果你有大量的长生命周期固定对象分散在 gen0（因为我们降低了它们的代别），`FragmentationAfterBytes` 可能会很大，因为固定对象之间有很多空闲空间。但你应该看到 `FragmentationBeforeBytes` 更小，因为我们已经使用这些空闲空间来满足分配请求。

3. **监控不频繁的 GC**  
   你可以通过调用此 API 并指定 `GCKind.FullBlocking` 来非常不频繁地监控，以判断是否发生了完整的阻塞 GC。如果你希望避免这类 GC，因为它们可能导致非常长的暂停，这种方法非常有用。
A usage example for measuring the GC impact of your tail latency
Recently I helped a team to measure the GC impact of their tail latency which I thought would be very useful for other people who care about tail latency. In my mem-doc I talked about measuring the GC impact of tail latency and this team was interested to do that so I helped them to write the code that utilizes this updated API to achieve it. It's important to mention the context here – this was used in their test scenarios where we know there are no full blocking GCs happening. So we only needed to check for ephemeral GCs and BGCs. I'll leave checking for full blocking GCs as an exercise to the readers 🙂

4) **"% time in GC" 性能计数器的问题**  
"% time in GC" 性能计数器仅适用于最后一个 GC（如果发生了后台 GC，这个值可能并不准确）。对于长期监控而言，了解到目前为止 GC 的暂停时间百分比更为有用。因此，API 提供的 `PauseTimePercentage` 就是为此设计的，非常适合用于长期监控。如果你想要知道每个单独 GC 的暂停时长，可以查看 `PauseDurations` 属性，它提供了两个时长——对于阻塞型 GC 来说，由于它们只有一个暂停阶段，所以只使用第一个时长。而对于后台 GC（BGC），有两个时长，因为每次 BGC 会暂停两次：一次是初始暂停，另一次是在标记阶段结束时（在 mem-doc 的“GC 事件序列”部分中有详细描述）。你可以分别获取这些暂停时长。实际上，这比 TraceEvent 提供的信息更详细，TraceEvent 将这两个暂停合并为一个，并报告总时长，你可以在 PerfView 的 GCStats 视图中的 "Pause MSec" 列中看到这一结果。

---

5) **代别对 GC 的重要性**  
我在 mem-doc 中详细讨论了了解 GC 的代别特性的重要性。不幸的是，很少有工具真正暴露了这一特性。甚至一些经验丰富的性能工程师对此也完全没有意识。这个 API 提供了每次 GC 的 `PromotedBytes`，你可以清楚地看到短暂代 GC 和完整 GC 之间的提升字节数差异。例如，你可以将短暂代 GC 的 `PauseDurations[0]` 与其 `PromotedBytes` 关联起来。如果你发现它们之间完全没有关联，比如尽管提升的字节数保持稳定，但暂停时长却有很大波动，那么如果你使用的是 Server GC，可能是遇到了 CPU 被抢占的问题（stolen CPU problem）。

这种情况下，CPU 时间被其他任务占用，导致 GC 线程无法获得足够的 CPU 时间来完成其工作，从而增加了暂停时间。通过结合分析 `PauseDurations` 和 `PromotedBytes`，你可以更好地诊断和理解 GC 的行为，尤其是当系统负载较高或存在资源争用时。

// The requests here are only requests of interest to our analysis, ie the ones that 
// took >=rangeStart and < rangeEnd milliseconds.
ulong global_totalReqsAffectedByGC = 0;
ulong global_totalReqs = 0;
ulong global_totalReqsAffectedByMultipleGCs = 0;
ulong global_unfortunateReqs = 0;
ulong global_totalReqsGCDurationMSec = 0;
ulong global_totalReqsDurationMSec = 0;

// in each request -
var startGollectionCount = GC.CollectionCount(0);
var startGen2CollectionCount = GC.CollectionCount(2);
stopwatch.Start();

// user request happens here
myRequest();

stopwatch.Stop();
ulong reqDurationMSec = (ulong)stopWatch.ElapsedMilliseconds;

int endCollectionCount = GC.CollectionCount(0);

// This is a request of interest.
if ((time >= rangeStart) && (time < rangeEnd))
{
    Interlocked.Add(ref global_totalReqsDurationMSec, reqDurationMSec);
    var endCollectionCount = GC.CollectionCount(0);
    var endGen2CollectionCount = GC.CollectionCount(2);

    GCMemoryInfo lastEphemeralGCInfo = GC.GetGCMemoryInfo(GCKind.Ephemeral);

    if (endCollectionCount > startGollectionCount)
    {
        Interlocked.Increment(ref global_totalReqsAffectedByGC);

        // For global_totalReqsAffectedByMultipleGCs we only count if there were multiple ephemeral GCs
        if (((endCollectionCount - startGCCount) > 1) && (endGen2CollectionCount == startGen2GCCount))
        {
            Interlocked.Increment(ref global_totalReqsAffectedByMultipleGCs);
        }
        // There could be an ephemeral GC happened inbetween getting endCollectionCount and 
        // lastEphemeralGCInfo. This would be very rare but can happen. So we just log it as
        // an unfortunate outlier case.
        if (lastEphemeralGCInfo.Index != endCollectionCount)
        {
            Interlocked.Increment(ref global_unfortunateReqs);
        }
        else
        {
            // Now we can calculate the GC impact on this request.
            // Of course you can also preserve enough accurancy on the GC pause duration by multiplying it with 10
            // so you don't lose the .x ms part. Just need to compensate for that when you do the impact calcuation.
            Interlocked.Add(ref global_totalReqsGCDurationMSec, (ulong)lastEphemeralGCInfo.PauseDurations[0].TotalMilliseconds);

            // You could also record the GC index because multiple requests could be affected by the same GC so you
            // know how many total unique GCs affected these requests.
        }
    }
}
view rawgcimpact.cs hosted with ❤ by GitHub
It's worth explaining line#37. We could have this case –

request begin: gc count is 1 (and gen2 count is 0)
now a BGC happens which increases the gc count to 2; and before we restart the user threads we choose to do a gen0 or gen1 GC which would increase gc count to 3. (we increase the gc count at the beginning of each GC)
user threads are now restarted
request end: gc count is 3 (and gen2 count is 1)
So at request end we would have observed multiple GCs, but most of the pause time would be from the gen0/1 GC. We could detect this by getting the gen2 count at the beginning as well as gen0 count (it would be better if we had an API that gets gen0/1/BGC/full blocking GC counts all at once instead of having to call multiple of them; I'll definitely think about adding that API).

And this would be the kind of output you'd get –

(global_totalReqsAffectedByGC / global_totalReqs) * 100 --> % requests affected by GC

(global_totalReqsGCDurationMSec / global_totalReqsDurationMSec) * 100 --> GC impact in these requests.

global_totalReqsAffectedByMultipleGCs, global_unfortunateReqs --> just to make sure those are indeed rare.

This is very useful because it tells you whether you should focus on finding out if other factors contribute to your tail latency if the GC impact is low. And if the GC impact is high then you know if you concentrated on improve the GC perf it would help reducing your tail latency. And you can give it different ranges to define which latency bucket you are concentrating on so you can know things like "GC seems to impact my Px request latency a lot but not much my Py latency so if my highest priority is to reduce Py latency I should spend time on something else" (where Px and Py is x and y percentile respectively). And this is the kind of things that I see folks mostly do guessing work on today so I definitely hope you take advantage of the new API!

### 请求开始：GC 计数为 1（gen2 计数为 0）
现在发生了一个后台 GC (BGC)，这将 GC 计数增加到 2；在我们重新启动用户线程之前，我们选择进行一次 gen0 或 gen1 GC，这会将 GC 计数进一步增加到 3。（我们在每次 GC 开始时都会增加 GC 计数。）  
用户线程现在重新启动。  
**请求结束：GC 计数为 3（gen2 计数为 1）。**

因此，在请求结束时，我们会观察到多个 GC，但大部分暂停时间实际上来自于 gen0/1 GC。我们可以通过在开始时获取 gen2 计数以及 gen0 计数来检测这一点。（如果有一个 API 能一次性获取 gen0、gen1、BGC 和完整阻塞 GC 的计数，而不是多次调用不同的方法，那就更好了；我一定会考虑添加这样一个 API。）

---

### 这是你可能会得到的输出：

- **(global_totalReqsAffectedByGC / global_totalReqs) * 100** → 受 GC 影响的请求数百分比。
- **(global_totalReqsGCDurationMSec / global_totalReqsDurationMSec) * 100** → GC 对这些请求的影响百分比。
- **global_totalReqsAffectedByMultipleGCs, global_unfortunateReqs** → 仅为了确保这些情况确实很少发生。

这非常有用，因为它可以告诉你是否应该专注于查找其他因素对尾部延迟的影响（如果 GC 的影响较低的话）。而如果 GC 的影响较高，则你可以知道集中精力优化 GC 性能是否会帮助降低尾部延迟。此外，你可以定义不同的延迟范围，以确定你关注的是哪个延迟区间。例如，你可以得出这样的结论：“GC 对我的 Px 请求延迟有很大影响，但对 Py 延迟影响不大，因此如果我的最高优先级是减少 Py 延迟，我应该把时间花在其他事情上。”（其中 Px 和 Py 分别表示 x 和 y 百分位的延迟。）

今天，我看到很多人在这方面主要依赖猜测工作，所以我强烈建议你充分利用这个新的 API！