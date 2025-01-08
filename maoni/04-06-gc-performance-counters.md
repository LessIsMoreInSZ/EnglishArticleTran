<h1>GC Performance Countersh</h1>
There are many .NET Memory Performance Counters and this is meant to give you some guidelines in interpreting the counter data and how to correlate them. This assumes you have a basic understanding of GC.

 

First thing you may want to look at is “% Time in GC”. This is the percentage of the time spent in GC since the end of the last GC. For example, it’s been 1 million cycles since last GC ended and we spent 0.3 million cycles in the current GC, this counter will show 30%.

 

What is a health value for this counter? It’s hard to say. It depends on what your app does. But if you are seeing a really high value (like 50% or more) then it’s a reasonable time to look at what’s going on inside of the managed heap. If this number is 10%, it’s probably better to look elsewhere in your app because even if you could get rid of half of that, you would only be saving 5% – most likely not very worthwhile.

 

If you think you are spending too much time in GC, it’s a good time to look at “Allocated Bytes/sec” which shows you the allocation rate. This counter is not exactly accurate when the allocation rate is too low – meaning that if the sampling frequency is higher than the GC frequency as the counter is only updated at the beginning of each GC.

 

When each GC begins, there could be 2 cases:

 

1)      Gen0 is basically full (meaning that it wasn’t large enough to satisfy the last small object allocation request);

2)      The LOH (Large Object Heap) is basically full (meaning that it wasn’t large enough to satisfy the last large object allocation request); 

 

When the allocation request can’t be satisfied, it triggers a GC. So when this GC begins we update the value this counter uses by adding the number of allocated bytes in gen0 and LOH to it, and since it’s a rate counter the actual number you see is the difference between the last 2 values divided by the time interval.

Let’s say you are sampling every second (which is the default in PerfMon) and during the 1st second a Gen0 GC occured which allocated 250k. So at the end of the 1st second PerfMon samples the counter value which will be (250k-0k) / 1 second = 250k/sec. Then no GCs happened during the 2nd and the 3rd second so we don’t change the value we recorded which will still be 250k, now you get (250k-250k) / 1 second = 0k/sec. Then let’s say a Gen0 GC happened during the 4th second and we recorded that we allocated 505k total, so at the end of the 4th second, PerfMon will show you (505k-250k) / 1 second = 255k/sec.

This means when GC doesn’t happen very frequently, you get these 0k/sec counter values. But if say there’s always at least one GC happening during each second, you will see an accurate value for the allocation rate (when you are sampling every second, that is).


We also have a counter that’s just for large objects – “Large Object Heap size”. This is updated at the same time “Allocated Bytes/sec” is updated and it just counts the bytes allocated in LOH. If you suspect that you are allocating a lot of large objects (in Whidbey this means 85000 bytes or more. But the runtime itself uses the LOH so if you see the LOH size less than 85000 bytes that’s due to the runtime allocation [Editted on 12/31/2004]), you can look at this counter along with “Allocated Bytes/sec” to verify it.

 

Allocating at a high rate definitely is a key factor that causes GC to do a lot of work. But if your objects usually die young, ie, mostly Gen0 GCs, you shouldn’t observe a high percentage of time spend in GC. Ideally if they all die in Gen0 then you could be doing a lot of Gen0 GCs but not much time will be spent in GC as Gen0 GCs take little time.

 

Gen2 GC requires a full collection (Gen0, Gen1, Gen2 and LOH! Large objects are GC’ed at every Gen2 GC even when the GC was not triggered by lack of space in LOH. Note that there isn’t a GC that only collects large objects.) which takes much longer than younger generation collections. A healthy ratio is for every 10 Gen0 GC we do a Gen1 GC; and for every 10 Gen1 GC we do a Gen2 GC. If you are seeing a lot of time spent in GC it could be that you are doing Gen2 GC’s too often. So look at collection counters:

 

“# Gen 0 Collections”

“# Gen 1 Collections”

“# Gen 2 Collections”

 

They show the number of collections for the respective generation since the process started. Note that a Gen1 collection collects both Gen0 and Gen1 in one pass – we don’t do a Gen0 GC, and then determine that no space is available in Gen0, then we go do a Gen1 GC. Which generation to collect is decided at the beginning of each GC.

 

If you are seeing a lot of Gen2 GCs, it means you have many objects that live for too long but not long enough for them to always stay in Gen2. When you are spending a lot of time in GC but the allocation rate is not very high, it might very well be the case that many of the objects you allocated survived garbage collection, ie, they get promoted to the next generation. Looking at the promotion counters should give you some idea, especially the Gen1 counter:

 

“Promoted Memory from Gen 0”

“Promoted Memory from Gen 1”

 

Note that these values don’t include the objects promoted due to finalization for which we have this counter:

 

“Promoted Finalization-Memory from Gen 0”

 

It gets updated at the end of each GC. Note that in Everett there’s also the “Promoted Finalization-Memory from Gen 1”counter which was removed in Whidbey. The reason was it was not useful. The “Promoted Finalization-Memory from Gen 0” counter already has the memory promoted from both Gen 0 and Gen 1. We should really rename it to just “Promoted Finalization-Memory”.

 

One of the worst situations is when you have objects that live long enough to be promoted to Gen2 then die quickly in Gen2 (the “midlife crisis” scenario). When you have this situation you will see high values for “Promoted Memory from Gen 1” and lots of Gen2 GCs. You can then use the CLRProfiler to find out which objects they are.

 

Note that when a finalizable object survives all the objects it refers to also survive. And the values for “Promoted Finalization-Memory from Gen X” include these objects too. Using the CLRProfiler is also a convenient way to find out which objects got finalized.

 

When you observe high values for these promotion counters, you will most likely observe high values for the Gen1 and Gen2 heap sizes as well. These sizes are indicated by these counters:

 

“Gen 1 heap size”

“Gen 2 heap size”

 

We also have a counter for Gen 0 heap size but it means the budget we allocated for the next GC (ie, the number of bytes that would trigger the next GC) and not the actual Gen 0 size which is either 0 or a small number if there’s pinning in Gen 0.

 

The heap size counters are updated at the end of each GC and indicate values for that GC. Gen0 and Gen1 heap sizes are usually fairly small (from ~256k to a few MBs) but Gen2 can get arbitrarily big.

 

If you want to get an idea of how much memory allocated total on the GC heap, you can look at these 2 counters:

 

“# Total committed Bytes”

“# Total reserved Bytes”

 

They are updated at the end of each GC and indicate the total committed/reserved memory on the GC heap at that time. The value of the total committed bytes is a bit bigger than

 

“Gen 0 heap size” + “Gen 1 heap size” + “Gen 2 heap size” + “Large Object Heap Size”

 

Note that this “Gen 0 heap size” is the actual Gen0 size – not the Gen0 budget as shown in the perfmon counter. [Editted on 12/31/2004]

 

When we allocate a new heap segment, the memory is reserved for that segment and will only be committed when needed at an appropriate granularity. So the total reserved bytes could be quite a bit bigger than the total committed bytes.

 

If you see a high value for “# Induced GC” it’s usually a bad sign – someone is calling GC.Collect too often. It’s often a bug in the app when this happens.

 

[Editted on 12/31/2004]

https://devblogs.microsoft.com/dotnet/gc-performance-counters/
