<h1>Correlating the output of !eeheap -gc and !address</h1>
!address is a very powerful debugger command. It shows you exactly what your VM space looks like. 
If you already got the output from the !sos.eeheap -gc command (refer to this article for usage on sos), for example:

address是一个非常强大的调试命令。它会准确地显示VM空间的样子。
如果已经得到了！sos的输出。Eeheap -gc命令（在sos上的使用请参考本文），例如：

0:003> !eeheap -gc
Number of GC Heaps: 1
generation 0 starts at 0x01245078
generation 1 starts at 0x0124100c
generation 2 starts at 0x01241000
ephemeral segment allocation context: (0x0125a900, 0x0125b39c)
 segment    begin allocated     size
001908c0 793fe120  7941d8a8 0x0001f788(128904)
01240000 01241000  0125b39c 0x0001a39c(107420)
Large object heap starts at 0x02241000
 segment    begin allocated     size
02240000 02241000  02243250 0x00002250(8784)
Total Size   0x3bd74(245108)
——————————
GC Heap Size   0x3bd74(245108)

you can correlate the segments with the output of !address to get a better view of them. For this specific case here’s the excerpt of the output from the !address command:
您可以将这些段与！address的输出关联起来，以便更好地查看它们。对于这个特定的例子，下面是！address命令输出的摘录：
0:003> !address
[omitted]
    01232000 : 01232000 – 0000e000
                    Type     00000000
                    Protect  00000001 PAGE_NOACCESS
                    State    00010000 MEM_FREE
                    Usage    RegionUsageFree
    01240000 : 01240000 – 00052000
                    Type     00020000 MEM_PRIVATE
                    Protect  00000004 PAGE_READWRITE
                    State    00001000 MEM_COMMIT
                    Usage    RegionUsageIsVAD
               01292000 – 00fae000
                    Type     00020000 MEM_PRIVATE
                    Protect  00000000
                    State    00002000 MEM_RESERVE
                    Usage    RegionUsageIsVAD
               02240000 – 00012000
                    Type     00020000 MEM_PRIVATE
                    Protect  00000004 PAGE_READWRITE
                    State    00001000 MEM_COMMIT
                    Usage    RegionUsageIsVAD
               02252000 – 00fee000
                    Type     00020000 MEM_PRIVATE
                    Protect  00000000
                    State    00002000 MEM_RESERVE
                    Usage    RegionUsageIsVAD
    03240000 : 03240000 – 73050000
                    Type     00000000
                    Protect  00000001 PAGE_NOACCESS
                    State    00010000 MEM_FREE
                    Usage    RegionUsageFree
    76290000 : 76290000 – 00001000
                    Type     01000000 MEM_IMAGE
                    Protect  00000002 PAGE_READONLY
                    State    00001000 MEM_COMMIT
                    Usage    RegionUsageImage
                    FullPath C:\WINDOWS\system32\IMM32.DLL
               76291000 – 00015000
                    Type     01000000 MEM_IMAGE
                    Protect  00000020 PAGE_EXECUTE_READ
                    State    00001000 MEM_COMMIT
                    Usage    RegionUsageImage
                    FullPath C:\WINDOWS\system32\IMM32.DLL
[omitted]

——————– Usage SUMMARY ————————–
[omitted]

——————– State SUMMARY ————————–
    TotSize   Pct(Tots) Usage
   0275c000 : 1.92%      : MEM_COMMIT
   7b20a000 : 96.20%      : MEM_FREE
   0268a000 : 1.88%      : MEM_RESERVE

Largest free region: Base 03240000 – Size 73050000

This says that the 2 segments (starting from 01240000 and 02240000) are adjacent to each other – part of them are committed, 
the rest is still reserved memory. Before and after the 2 segments we got some free space there. 
As I mentioned below it’s very unlikely that the managed heap is fragmenting the VM because we are good about requesting large chunks at a time and usually the OS is not bad at giving us addresses that are pretty contiguous if possible. One of the very few cases where you would see managed heap fragmenting VM is if you have temporary large object segments and GC needs to frequently acquire and release VM chunks. Those chunks could be scattered in the VM space especially considering there are other things that consume VM as well at the same time.

这表示两个段（从01240000和02240000开始）彼此相邻-其中一部分已提交，
其余的仍然是预留内存。在这两段之前和之后都有一些空闲空间。
正如我在下面提到的，托管堆不太可能使VM分片，因为我们很擅长一次请求大块，而且通常操作系统在给我们提供可能的相当连续的地址方面并不差。
您将看到托管堆碎片VM的极少数情况之一是，如果您有临时的大型对象段，并且GC需要频繁获取和释放VM块。这些块可以分散在VM空间中，特别是考虑到还有其他事情同时消耗VM。
https://devblogs.microsoft.com/dotnet/correlating-the-output-of-eeheap-gc-and-address/

