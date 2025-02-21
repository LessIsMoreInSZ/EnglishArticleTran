<h1>Difference Between Perf Data Reported by Different Tools – 2</h1>

Managed Heap Size

We have both .NET CLR Memory perf counters and SoS extensions that report manged heap size related data.

Difference 2

There are a few .NET CLR Memory counters that are related to the managed heap size:

# Total Committed Bytes

# Total Reserved Bytes

I explained what these counters mean here.

Now, how are they related to the values you see when you do a !SOS.eeheap –gc?

0:003> !eeheap -gc Number of GC Heaps: 1 generation 0 starts at 0x01245078 generation 1 starts at 0x0124100c generation 2 starts 
at 0x01241000 ephemeral segment allocation context: (0x0125a900, 0x0125b39c) segment    
begin allocated     size 001908c0 793fe120  7941d8a8 0x0001f788(128904) 01240000 01241000  0125b39c 0x0001a39c(107420) 
Large object heap starts at 0x02241000 segment       
begin allocated  size 02240000 02241000  02243250 0x00002250(8784) Total Size   0x3bd74(245108) —————————— GC Heap Size   0x3bd74(245108)

The allocated column indicates the end of the last live object on the segment. So for gen0 and LOH it changes as the managed threads allocate, 
unlike the .NET CLR Memory counters which only reflect the end of the last live object on the segment when the last GC’s happened 
(at the end of last GC).

The # Bytes in All Heaps counter  under .NET CLR Memory counter is kind of misleading. 
The explanation says it’s the bytes in “all heaps” but really it’s gen1+gen2+LOH in CLR 2.0, 
so it doesn’t include gen0 ‘cause most of the time gen0 size is 0 right after a GC. 
So if you break into your process under the debugger and use the !sos.eeheap –gc command you will most likely get a value that’s the same as this counter.
But if you break between 2 GCs the value you get from !eeheap –gc will always be larger than the value of this counter.

If you are concerned with the true amount of memory that the managed heap commits (which is usually what you need to worry about) 
you should use the # Total Committed Bytes counter.

https://devblogs.microsoft.com/dotnet/difference-between-perf-data-reported-by-different-tools-2/

托管堆大小

我们有。net CLR内存性能计数器和SoS扩展来报告托管堆大小相关的数据。

差2

有一些。net CLR内存计数器与托管堆大小相关：

#总已提交字节数

#总预留字节数

我在这里解释了这些计数器的含义。

现在，它们与你执行SOS时看到的值有什么关系呢？eeheap gc吗?

[00:03] eeheap -gc GC堆的个数：第0代从0x01245078开始，第1代从0x0124100c开始，第2代开始
在0x01241000临时段分配上下文：（0x0125a900, 0x0125b39c）段
开始分配大小001908c0 793fe120 7941d8a8 0x0001f788(128904) 01240000 01241000 0125b39c 0x0001a39c（107420）
大对象堆从0x02241000段开始
开始分配大小02240000 02241000 02243250 0x00002250（8784）总大小0x3bd74（245108）——————————GC堆大小0x3bd74（245108）

分配的列表示段上最后一个活动对象的结束。所以对于gen0和LOH，它随着托管线程的分配而变化，
不像。net CLR内存计数器，它只反映最后一次GC发生时段上最后一个活动对象的结束
（上次GC结束时）。

. net CLR内存计数器下的# Bytes in All Heaps计数器有点误导人。
解释说它是“所有堆”中的字节数，但实际上是CLR 2.0中的gen1+gen2+LOH，
所以它不包括gen0，因为大多数情况下，gen0的大小在GC之后是0。
因此，如果您在调试器下闯入您的进程并使用！使用Eeheap -gc命令，您很可能会得到与此计数器相同的值。
但是如果你在2个gc之间中断，你从！eeheap -gc得到的值将总是大于这个计数器的值。

如果您关心托管堆提交的真实内存量（这通常是您需要担心的）
你应该使用# Total Committed Bytes计数器。