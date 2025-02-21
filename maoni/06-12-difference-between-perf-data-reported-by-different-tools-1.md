<h1>difference-between-perf-data-reported-by-different-tools-1</h1>

So, there are many perf tools and some of them report either the same or the same type of data. 
I want to talk about various differences between the ones related to managed heap investigation. 
This is not supposed to cover everything..just the ones I think people use frequently.

Managed Heap Size

We have both .NET CLR Memory perf counters and SoS extensions that report manged heap size related data.

Difference 1

The values reported by the .NET CLR Memory perf counters are affected by

1) how often they are collected – meaning however often you tell the tool you use (most people use perfmon, 
internally I know many groups’ test teams collect them as part of the automation). The most frequent interval perfmon gives you is once per second.

2) how often they are updated. Most .NET CLR Memory perf counters are updated only at the end of each GC.

So for example, assuming you are using perfmon to collect at one sample per second and if more than one GC happened in the past second, 
you’ll be missing the intermediate values.

And since these perf counters are updated only every so often, before a value is refreshed it stays the same value as it was updated last time. 
For example, if you are looking at % Time in GC and say the last value was 80% and if no GC happens for 10 seconds, 
this counter will stay at 80% but really during the past 10 seconds no GCs were happening.

On the other hand, SOS data is only gotten when you request, 
which means you get the value at the time you did the SOS command (unless of course the command specifically tells you 
it reflects a value updated at some specific point in the past).

https://devblogs.microsoft.com/dotnet/difference-between-perf-data-reported-by-different-tools-1/

因此，有许多性能工具，其中一些报告相同或相同类型的数据。
我想谈谈与托管堆调查相关的各种不同之处。
这并不是要涵盖所有内容，只是我认为人们经常使用的那些。

托管堆大小

我们有。net CLR内存性能计数器和SoS扩展来报告托管堆大小相关的数据。

差1

.NET CLR内存性能计数器报告的值受以下因素的影响

1)收集它们的频率——意思是你告诉你使用的工具的频率(大多数人使用perfmon，
在内部，我知道许多小组的测试团队收集它们作为自动化的一部分)。最频繁的时间间隔是每秒一次。

2)更新的频率。大多数。net CLR内存性能计数器只在每次GC结束时更新。

例如，假设您正在使用perfmon以每秒一个样本的速度收集，并且如果在过去一秒钟内发生了多个GC，
你会错过中间值。

由于这些性能计数器只是隔一段时间更新一次，所以在刷新值之前，它会保持与上次更新时相同的值。
例如，如果您正在查看GC中的% Time，并说最后一个值为80%，并且如果10秒内没有发生GC，
这个计数器将保持在80%，但实际上在过去的10秒内没有发生gc。

另一方面，SOS数据只有在你请求时才会得到，
这意味着您在执行SOS命令时获得该值（当然，除非命令特别告诉您）
它反映了过去某个特定时间点更新的值)。