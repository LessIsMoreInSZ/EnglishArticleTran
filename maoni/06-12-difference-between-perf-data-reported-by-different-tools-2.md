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