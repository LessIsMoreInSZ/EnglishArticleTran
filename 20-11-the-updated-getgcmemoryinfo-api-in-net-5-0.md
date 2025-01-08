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

A usage example for measuring the GC impact of your tail latency
Recently I helped a team to measure the GC impact of their tail latency which I thought would be very useful for other people who care about tail latency. In my mem-doc I talked about measuring the GC impact of tail latency and this team was interested to do that so I helped them to write the code that utilizes this updated API to achieve it. It's important to mention the context here – this was used in their test scenarios where we know there are no full blocking GCs happening. So we only needed to check for ephemeral GCs and BGCs. I'll leave checking for full blocking GCs as an exercise to the readers 🙂


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