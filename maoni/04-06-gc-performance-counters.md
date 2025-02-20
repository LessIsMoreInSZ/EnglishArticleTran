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

.NET 中有许多内存性能计数器，其目的在于为您提供一些解读计数器数据以及如何对其进行关联的指导原则。此内容假定您对垃圾回收（GC）有基本的了解。




首先，您可能需要关注的是“% GC 时间”。这是自上一次 GC 结束以来在 GC 中所花费的时间的百分比。例如，自上次 GC 结束以来已经过去了 100 万次循环，而本次 GC 中我们花费了 30 万次循环，那么这个计数器将会显示为 30%。




对于这个计数器而言，其健康值是多少呢？这很难说。这取决于您的应用程序的功能。但如果看到的值非常高（比如 50% 或更多），那么此时查看管理堆内部的情况可能是合理的时机。如果这个数字是 10%，那么可能最好在您的应用程序的其他地方寻找问题所在，因为即使能够去除其中的一半，也只能节省 5%——很可能不值得这样做。




如果您觉得在垃圾回收（GC）期间花费的时间过多，那么此时查看“每秒分配字节数”（Allocated Bytes/sec）是一个不错的选择，因为它能显示分配速率。当分配速率过低时，这个计数器的准确性会受到影响——这意味着如果采样频率高于垃圾回收频率（因为计数器仅在每次垃圾回收开始时更新），那么该计数器的准确性就会降低。




每次基因拷贝（GC）开始时，可能会出现两种情况：




1) Gen0 堆基本上已满（这意味着其容量不足以满足上一次小对象分配请求的需求）；

2)  LOH（大型对象堆）基本上已满（这意味着其容量不足以满足上一次对大型对象的分配请求）；




当分配请求无法得到满足时，就会触发一次垃圾回收（GC）。因此，在这个 GC 开始时，我们会更新这个计数器所使用的值，将其与 gen0 和 LOH 中分配的字节数相加，由于这是一个速率计数器，您看到的实际数值是上两个值之间的差值除以时间间隔所得的结果。

假设您每隔一秒进行一次采样（这是 PerfMon 的默认设置），在第 1 秒发生了一次 Gen0 垃圾回收（GC），分配了 250k 的内存。因此，在第 1 秒末，PerfMon 会采样计数器值，其结果将是 (250k - 0k) / 1 秒 = 250k/秒。然后，在第 2 秒和第 3 秒期间没有发生 GC，所以我们记录的值保持不变，仍然是 250k，现在得到 (250k - 250k) / 1 秒 = 0k/秒。假设在第 4 秒发生了一次 Gen0 GC，并且我们记录了总共分配了 505k 的内存，那么在第 4 秒末，PerfMon 会显示给您 (505k - 250k) / 1 秒 = 255k/秒。

这意味着，如果垃圾回收（GC）操作不频繁发生，那么你会得到每秒 0k 的计数值。但如果假设每秒至少有一次垃圾回收操作发生，那么你会看到准确的分配速率值（当你每秒采样一次时，就是这种情况）。



我们还有一个专门针对大型对象的计数器——“大型对象堆大小”。这个计数器与“每秒分配字节数”同时更新，它只是统计 LOH 中分配的字节数。如果您怀疑自己正在分配大量的大型对象（在 Whidbey 中这意味着 85000 字节或更多。但运行时本身会使用 LOH，所以如果您看到 LOH 大小小于 85000 字节，那是因为运行时进行了分配[2004 年 12 月 31 日修订]），您可以同时查看这个计数器和“每秒分配字节数”来验证这一点。




以较高的频率进行垃圾回收（GC）分配操作无疑是一个会导致 GC 处理大量工作的关键因素。但如果您的对象通常都很年轻（即大多属于 Gen0 垃圾回收），那么您不应该看到有很高的时间比例被消耗在 GC 上。理想情况下，如果它们都在 Gen0 中死亡，那么您可能会进行大量的 Gen0 垃圾回收操作，但不会在 GC 上花费太多时间，因为 Gen0 垃圾回收操作所需时间很少。




Gen2 GC（第二代垃圾回收）需要进行一次完整的回收（包括 Gen0、Gen1、Gen2 和 LOH！即使在 LOH 中没有空间可用的情况下，大型对象也会在每次 Gen2 GC 时被回收）——这比年轻代的回收要耗费更多的时间。理想的回收比例是：每执行 10 次 Gen0 GC 就执行一次 Gen1 GC；每执行 10 次 Gen1 GC 就执行一次 Gen2 GC。如果发现 GC 所花费的时间很长，可能是您执行 Gen2 GC 的频率过高了。所以，请查看回收计数器：



“# Gen 0 Collections”

“# Gen 1 Collections”

“# Gen 2 Collections”




它们展示了自进程启动以来每个代的收集次数。请注意，Gen1 收集器在一次遍历中会同时收集 Gen0 和 Gen1 的内存——我们不会先执行 Gen0 的垃圾回收，然后确定 Gen0 中没有可用空间，再执行 Gen1 的垃圾回收。每次垃圾回收开始时，决定要收集哪个代是基于这样的流程：先执行垃圾回收操作，然后根据回收结果来确定要收集哪个代。




如果看到大量的 Gen2 垃圾回收操作（GC），这意味着存在很多对象的存活时间过长，但又不足以使其一直保留在 Gen2 代中。当您花费大量时间进行垃圾回收，但分配率却不高时，很可能的情况是您分配的对象中有许多在垃圾回收后仍然存活下来，也就是说，它们被提升到了下一个代。查看一下您的代码，看看是否有可能存在这样的情况：您分配的对象在垃圾回收后仍然存活下来，即它们被提升到了下一个代。
如果看到大量的 Gen2 垃圾回收操作（GC），这意味着存在很多对象的存活时间过长，但又不足以使其一直保留在 Gen2 代中。当您花费大量时间进行垃圾回收，但分配率却不高时，很可能的情况是您分配的对象中有许多在垃圾回收后仍然存活下来，也就是说，它们被提升到了下一个代。查看晋升计数器应该能给您一些启示，尤其是 Gen1 计数器：
请注意，这里提到的 Gen1 和 Gen2 是指 Java 虚拟机中的两种不同的代，它们分别用于存储年轻代和老年代的对象。




“从 Gen 0 中提升的记忆”

“从《创世纪》第一章中提升的记忆”




请注意，这些数值并未包含因对象终结而被提升的对象（对于这些对象，我们有此计数器）：




“将 Gen 0 中的“完成”内存提升为最终化内存”




每次垃圾回收（GC）结束时都会进行更新。请注意，在 Everett 中还有一个名为“从 Gen 1 转移的终结内存”的计数器，它在 Whidbey 版本中已被移除。其原因是它没有实际用途。而“从 Gen 0 转移的终结内存”计数器已经包含了从 Gen 0 和 Gen 1 中转移过来的内存。我们确实应该将其重命名为“终结内存”。




最糟糕的情况之一是，存在一些对象能够存活足够长的时间从而晋升到 Gen2 分代中，但随后又在 Gen2 中迅速死亡（即所谓的“中年危机”情形）。当出现这种情况时，您会看到“从 Gen1 晋升到 Gen2 的内存”这一项的值很高，并且会有大量的 Gen2 垃圾回收操作。此时，您可以使用 CLRProfiler 来找出这些对象究竟是哪些。




请注意，当一个可终结对象存活下来时，它所引用的所有对象也都存活下来。而且“从 X 代生成器晋升的终结内存”这一项的值还包括这些对象。使用 CLRProfiler 还是一种方便的方法来找出哪些对象被终结了。




当您观察到这些提升计数器的值较高时，您很可能会同时观察到 Gen1 和 Gen2 堆大小的值较高。这些计数器所指示的大小如下：
这些计数器所指示的大小如下：这些计数器所指示的大小如下：这些计数器所指示的大小如下：这些计数器所指示的大小如下：这些




“Gen 1 堆大小”

“Gen 2 堆大小”




我们还有一个针对 Gen 0 堆大小的计数器，但它所代表的含义是我们在下一次垃圾回收（GC）时所预留的预算（即触发下一次 GC 所需要的字节数），而不是实际的 Gen 0 堆大小，如果在 Gen 0 中存在内存锁定，那么实际的 Gen 0 堆大小要么为 0，要么是一个较小的数值。




每次垃圾回收（GC）结束时，堆大小计数器都会被更新，并显示该次 GC 的堆大小值。Gen0 和 Gen1 堆大小通常都比较小（从约 256k 到几 MB 不等），但 Gen2 堆大小可能会变得非常大。




如果您想大致了解 GC 堆中总共分配了多少内存，您可以查看这两个计数器：




“# 总已分配字节数”

“# 总预留字节数”




它们在每次垃圾回收（GC）结束时都会更新，并显示出在该次 GC 期间 GC 堆中所累计的已分配/预留的内存总量。已分配字节总量的数值略大于上述数值。




“新生代堆大小” + “老生代堆大小” + “永久代堆大小” + “大对象堆大小”




请注意，这里的“Gen 0 堆大小”指的是实际的 Gen0 堆大小——而非性能监视器计数器中所显示的 Gen0 预算大小。[于 2004 年 12 月 31 日进行了编辑]




当我们为一个新的堆段分配内存时，所分配的内存将专门用于该段，并且只有在需要时才会以适当的粒度进行提交。因此，所预留的总字节数可能会比实际提交的总字节数大很多。




如果看到“# 引发的 GC”这一项的数值较高，通常就意味着情况不太妙——说明有人频繁地调用 GC.Collect 方法。这种情况发生时，往往意味着应用程序存在缺陷。