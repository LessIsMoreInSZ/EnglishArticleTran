<h1>using-gc-efficiently-part-4</h1>
In this article I’ll talk about things you want to look for when you look at the managed heap in your applications to determine if you have a healthy heap. I’ll touch on some topics related to large heaps and the implications you want to be aware of when you have an application that maintains or has potential for the need to maintain a managed heap of large sizes. For many people, it’s always a concern when they will get the Out of Memory exception when they have a heap that’s too big.

Lately I’ve been working on Whidbey Beta2 and Beta2+ (which was one reason why I haven’t posted in a long time) and for the past couple of months I’ve looked at many many memory dumps from various internal teams. Many teams needed help on identifying how healthy their heaps were and what factors would affect the heap usage so I thought I’d share some of this info with you.

How GC is concerned with virtual Memory and physical memory

Some of you probably are already perfectly familiar with this so you can just skim through this section.

GC needs to make allocations for its segments. For an explanation on segments please see Using GC Efficiently – Part 1. When we decide that we need to allocate a new segment, we call VirtualAlloc to allocate space for this segment. This means if there isn’t a contiguous free block in the virtual memory in your process’s address space that’s large enough for a segment, we will fail the allocation for the segment therefore fail the allocation request. This is one of the very few legitimate situations for GC to throw OOM at you (to be accurate, GC actually doesn’t throw any exceptions – it’s the execution engine that makes the allocation request on your behalf that throws the exception – GC just returns NULL to the allocation request).

I often get asked this question, “why do I get OOM when my heap is only X MB??” where X is a lot smaller than 2GB. Note this is on x86 and of course by “heap” they mean managed heap (I should be proud that many people are so customized to managed apps these days that they just use “heap” to refer to only managed heap J).

Remember that there are always allocations that are made not by GC. GC competes for the VM space just like anything else. Modules you load in your process need to take up VM space; some modules in your process could be making native allocations which also take up VM space (VirtualAlloc, HeapAlloc, new and whatnot). CLR itself makes native allocations as well for jitted code, datastructures the CLR needs (including the ones that GC needs to do its work) and etc. Usually the allocations CLR makes should be pretty small. You can use the !eeheap command from the SOS debugger extension dll to look at various categories of allocations that the CLR makes. Following is a simplified sample output with some comments in []’s:

The app is getting OOM at this point. Let’s look at the free VM blocks. You can achieve this by using the !vadump command or any other tool that analysises the VM space. My favorite is the !address command which nicely prints out the largest free region:

0:119> !eeheap
Loader Heap:
————————————–
System Domain: 79bbd970
LowFrequencyHeap: Size: 0x0(0)bytes.
HighFrequencyHeap: 007e0000(10000:2000) Size: 0x2000(8192)bytes.
StubHeap: 007d0000(10000:7000) Size: 0x7000(28672)bytes.
Virtual Call Stub Heap:
  IndcellHeap: Size: 0x0(0)bytes.
  LookupHeap: Size: 0x0(0)bytes.
  ResolveHeap: Size: 0x0(0)bytes.
  DispatchHeap: Size: 0x0(0)bytes.
  CacheEntryHeap: Size: 0x0(0)bytes.
Total size: 0x9000(36864)bytes
————————————–
Shared Domain: 79bbdf18
…
Total size: 0xe000(57344)bytes
————————————–
Domain 1: 151a18
…
Total size: 0x147000(1339392)bytes
————————————–
Jit code heap:
LoaderCodeHeap: 23980000(10000:7000) Size: 0x7000(28672)bytes.
…
Total size: 0x87000(552960)bytes
[jited code takes very little space – this is a fairly large application]
————————————–
Module Thunk heaps:
Module 78c40000: Size: 0x0(0)bytes.
…
Total size: 0x0(0)bytes
————————————–
Module Lookup Table heaps:
Module 78c40000: Size: 0x0(0)bytes.
…
Total size: 0x0(0)bytes
————————————–
Total LoaderHeap size: 0x1e5000(1986560)bytes [total Loader heap takes < 2MB]
=======================================
Number of GC Heaps: 4
——————————
Heap 0 (0015ad08)
generation 0 starts at 0x49521f8c
generation 1 starts at 0x494d7f64
generation 2 starts at 0x007f0038
ephemeral segment allocation context: none
[The first 2 segments are read only segments for frozen strings which is why they look a bit odd compared to other segments. The addresses for begin and segment are very different and usually they are tiny segments (unless you have tons and tons of frozen strings)]
 segment    begin allocated     size
00178250 7a80d84c  7a82f1cc 0x00021980(137600)
00161918 78c50e40  78c7056c 0x0001f72c(128812)
007f0000 007f0038  047eed28 0x03ffecf0(67103984)
3a120000 3a120038  3a3e84f8 0x002c84c0(2917568)
46120000 46120038  49e05d04 0x03ce5ccc(63855820)
Large object heap starts at 0x107f0038
 segment    begin allocated     size
107f0000 107f0038  11ad0008 0x012dffd0(19791824)
20960000 20960038  224f7970 0x01b97938(28932408)
Heap Size  0xae65830(182868016)
——————————
Heap 1 (0015b688)
…
Heap Size  0x7f213bc(133305276)
——————————
Heap 2 (0015c008)
…
Heap Size  0x7ada9ac(128821676)
——————————
Heap 3 (0015cdc8)
…
Heap Size  0x764c214(124043796)
——————————
GC Heap Size  0x21ead7ac(569038764) [total managed heap takes ~540MB]

The app is getting OOM at this point. Let’s look at the free VM blocks. You can achieve this by using the !vadump command or any other tool that analysises the VM space. My favorite is the !address command which nicely prints out the largest free region:

0:119> !address
…
Largest free region: Base 54000000 – Size 03b60000

0:119> ? 03b60000
Evaluate expression: 62259200 = 03b60000

The largest free region is <64MB which explains why we are getting OOM exception – we were trying to allocate another GC segment but failed to do so (this is on Server GC and segment size is bigger).

The VM space can get badly fragmented if you load and unload variable sized modules often. A common scenario is if you have COM DLLs that get unloaded after the components in them are not in use anymore for a certain period of time (10mins I think?). Other common scenarios are tons of interop assemblies or tons of tiny DLLs for a web server (if you have a 10k DLL it’ll consume 64k in VM). 

Besides allocating VM space for segments there’s really nothing else about VM that GC is concerned with. Physical memory, on the other hand, is a totally different story. As I mentioned in Using GC Efficiently – Part 1, one of the 3 situations where a GC can happen is if your machine is low on physical memory. GC becomes more aggressive. So if you are looking at the GC performance counters you might notice that you have a different gen0, gen1 and gen2 ratio when your machine is low on memory. If GC fails to commit memory it needs to satisfy your allocation request you will also get OOM. The available physical memory can be obtained via the Memory\Available Bytes perfmon counter.

A few things worth mentioning are:

1) Nowadays it’s not uncommon to have a machine with more than 2GB of physical memory. So not fragmenting VM too badly is really important. However if you are running on 64-bit where the VM space is huge physical memory becomes the limiting factor.

2) You can use the Whidbey hosting API to communicate to GC when you want it to think you are in a low memory situation even though you really are not because either you want to only allocate part of the physical memory to this particular app, or you want to be on the safe side and not worry about getting to the point where you might start getting OOMs (many apps cannot handle OOMs).

Looking at a managed heap 

Before we get into that there are a few things I’d like to mention about taking the perf data for your heap:

Getting Perfmon logs

I recommend when you take GC perfmon counters you use the shortest interval possible (right now it’s 1 second) and capture the perfmon log for a few minutes. Usually it gives you much more insight than capturing a log for 2 hours with 5 or 10 seconds interval (which I often see people do when they send me perfmon logs) because if something is misbehaving it’s usually enough to see the pattern after a few minutes. However if the problem repros randomly of course you have no choice but to log for a possibly long time but it’s best to keep the interval short because GC usually happens frequent enough that you need a short interval to make more sense out of the log.

Taking memory dumps

If you take a memory dump for your process, take a full memory dump. A mini dump for analysing the managed heap is usually useless.

Getting CLRProfiler logs

CLRProfiler is very heavy weight and it can’t attach to a process but you can uncheck the Profiler Active checkbox so it’s not on when the app is not running to the point that you are interested in gathering data. You can also uncheck the Calls checkbox to gather less data if just profiling Allocations is enough for you.

———————————————————–

So what kind of things do you want to look for when you look at a managed heap?

Time in GC

If the % Time in GC is low, it’s not useful to look for allocation related issues. When % Time in GC is high it usually means you are doing too many Gen2 collections or each Gen2 collection takes too long. Then it’s time to start looking at your heap in more details to determine why you are doing so many Gen2 collections and how you can change your allocation pattern to spend less time on Gen2 collections.

The heap growth pattern

If you have a server app that needs to stay running for a long time, and if you observe that the heap is growing unbounded overtime that’s definitely a bad sign. Usually server apps should clean up all data related to a request after the request is finished. If that’s not the case you’ve got a memory leak that you must look at – otherwise you will get OOM. This is usually the 1st thing we ask app developers (especially server app devs) to fix.

CLRProfiler gives you the most extensive info. If you can run your app with it, great; if you can’t, because it’s too heavyweight, you could take some memory dumps (if you don’t have the luxury to debug the app live) and compare the heaps. I would suggest you to take the dumps right after a GC so you get more acccurate comparision. Otherwise you could be looking at one heap where most objects haven’t been cleaned up yet and another one where no dead objects exist because it just did a full GC.

Perfmon can also be really useful to pinpoint some obvious things such as: if the # of GC Handles keeps growing, it indicates you may have a handle leak. For more info on GC perfmon counter please refer to my previous blog entry.

The full GC  ratio

Often we tell people a healthy gen2:gen1 ratio is 1:10. If you are seeing a 1:1 ratio that’s certainly something to look into. When you have a really large heap and doing a gen2 could take some time it’s of course ideal to make the ratio of gen2 as low as possible. As with all perf advices there are always exceptions – for example if you have a heap where most objects are on the LOH and they don’t really move – we’ve seen this in certain server apps where they use some really arrays that live on the LOH to hold references to small objects and these arrays stay pretty much for the process lifetime, it’s not necessarily a bad situation. We do need to trace through the references that these large arrays contain but we don’t need to move memory around for them which would be an expensive operation. On the other hand, if you have an app that creates lots of temporary large objects you are likely to be in a bad situation if you also have a fairly big gen2 heap because the gen2’s that the large object allocations trigger will also need to collect your gen2 heap.

The common causes for doing a relatively high number of gen2’s include a managed cache implementation where the whole cache is in gen2 and it constantly gets churned; or if you have lots of finalizable objects whose finalizers get to run which increases the chance these objects make into gen2.

We’ve seen people who induce GCs, ie, calling GC.Collect, periodically. First of all, if it’s to “help not get OOMs” because you believe that GC is not kicking in early enough (if you are getting OOM because you have too many live objects and you exhausted memory it won’t be helped by inducing GCs anyway), you should tell us about it. Secondly, inducing GCs screws up GC’s tuning which is almost always a bad thing.

Fragmentation

Fragmentation is the free space you have on your managed heap which you can obtain by using the sos command

!dumpheap –type Free –stat

Of course the smaller the fragmentation the better but in the real world scenarios when you use IO and etc you will get pinned objects from the .net framework. Fragmenatation in different generations has different indications to how healthy your app is:

Fragmentation in gen0 is good because gen0 is where you allocate your objects and we will use the free space in gen0 to satisfy your allocation requests. As a hypothetical example, If we are looking a heap where all the Free objects are in gen0 that’s a very very ideal heap picture. You can specify a begin and an end address for !dumpheap to see how much space Free objects occupy in gen0.  Take the example from the dump mentioned at the beginning:

Heap 0 (0015ad08)
generation 0 starts at 0x49521f8c
generation 1 starts at 0x494d7f64
generation 2 starts at 0x007f0038
ephemeral segment allocation context: none
segment    begin  allocated    size
00178250 7a80d84c  7a82f1cc 0x00021980(137600)
00161918 78c50e40  78c7056c 0x0001f72c(128812)
007f0000 007f0038  047eed28 0x03ffecf0(67103984)
3a120000 3a120038  3a3e84f8 0x002c84c0(2917568)
46120000 46120038  49e05d04 0x03ce5ccc(63855820) [the last one is always the ephemeral segment]

0:119> ? 49e05d04-0x49521f8c
Evaluate expression: 9321848 = 008e3d78 [gen0 is about 9MB]

0:119> !dumpheap -type Free -stat 0x49521f8c 49e05d04
——————————
Heap 0
total 409 objects
——————————
Heap 1
total 0 objects
——————————
Heap 2
total 0 objects
——————————
Heap 3
total 0 objects
——————————
total 409 objects
Statistics:
      MT    Count TotalSize Class Name
0015a498      409   7296540      Free
Total 409 objects

So gen0 is 9MB and about 7MB is free space, ie, fragmentation.

Fragmentation in LOH is by design because we don’t compact LOH, which does NOT mean allocating on LOH is the same as malloc using the NT heap manager! Because of the nature of the way GC works, Free objects that are adjacent to each other are naturally collasped into one big free space which is available to satisfy your large object allocation requests.

The bad fragmentation is fragmenatation in gen2 and gen1. If after a GC you are still seeing lots of free space in gen2 and gen1 (especially gen2 since gen1 cannot exceed the size of a segment) that’s definitely something to look into. As I mentioned before we’ve been doing a lot of work on fragmenation and have improvement the situation by a lot but you can do your part by looking at how your app behaves and limit the fragmentation as much as possible from your code. Using GC Efficiently – Part 3 talks about good techniques to use when you do have to pin.

For gen2 if the fragmentation is lower than 20% it’s considered very good. Because gen2 can get really big it’s important to consider the ratio of fragmentation and not the absolute value.

Well, that’s all for today. See you next time!

[Editted on 05/16/2005 – I discovered that I actually misspelled “efficiently” (you’d think I can spell it but nooo…)…so I also took the opportunity to fix up the formatting that got messed up when I pasted the text from word to the editor for blogging the 1st time around.]

https://devblogs.microsoft.com/dotnet/using-gc-efficiently-part-4/