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
