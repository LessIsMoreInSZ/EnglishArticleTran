<h1>using-gc-efficiently-part-4</h1>
In this article I’ll talk about things you want to look for when you look at the managed heap in your applications to determine if you have a healthy heap. I’ll touch on some topics related to large heaps and the implications you want to be aware of when you have an application that maintains or has potential for the need to maintain a managed heap of large sizes. For many people, it’s always a concern when they will get the Out of Memory exception when they have a heap that’s too big.

Lately I’ve been working on Whidbey Beta2 and Beta2+ (which was one reason why I haven’t posted in a long time) and for the past couple of months I’ve looked at many many memory dumps from various internal teams. Many teams needed help on identifying how healthy their heaps were and what factors would affect the heap usage so I thought I’d share some of this info with you.

How GC is concerned with virtual Memory and physical memory

Some of you probably are already perfectly familiar with this so you can just skim through this section.

GC needs to make allocations for its segments. For an explanation on segments please see Using GC Efficiently – Part 1. When we decide that we need to allocate a new segment, we call VirtualAlloc to allocate space for this segment. This means if there isn’t a contiguous free block in the virtual memory in your process’s address space that’s large enough for a segment, we will fail the allocation for the segment therefore fail the allocation request. This is one of the very few legitimate situations for GC to throw OOM at you (to be accurate, GC actually doesn’t throw any exceptions – it’s the execution engine that makes the allocation request on your behalf that throws the exception – GC just returns NULL to the allocation request).

I often get asked this question, “why do I get OOM when my heap is only X MB??” where X is a lot smaller than 2GB. Note this is on x86 and of course by “heap” they mean managed heap (I should be proud that many people are so customized to managed apps these days that they just use “heap” to refer to only managed heap J).

这篇文章是关于如何在应用程序中检查托管堆的健康状况，以及在面对大型堆时需要注意的一些问题。以下是文章的主要内容翻译：

高效使用GC（第4部分）

在这篇文章中，我将讨论在检查应用程序的托管堆时，需要寻找哪些因素来确定堆是否健康。我会涉及到一些与大型堆相关的话题，以及当应用程序需要维护或有可能需要维护大型托管堆时，你需要意识到的一些影响。对于许多人来说，他们总是担心何时会收到内存不足异常，当堆太大时。

最近，我一直在研究Whidbey Beta2和Beta2+（这也是我一段时间没有发帖的原因）。在过去的几个月里，我查看了许多来自不同内部团队的内存转储。许多团队需要帮助识别他们的堆是否健康，以及哪些因素会影响堆的使用，所以我想与你们分享一些这方面的信息。

GC与虚拟内存和物理内存的关系

你可能已经非常熟悉这部分内容，所以可以快速浏览。

GC需要为其段分配空间。关于段的解释，请参见《高效使用GC – 第1部分》。当我们决定需要分配一个新段时，我们会调用VirtualAlloc来为这个段分配空间。这意味着，如果进程中虚拟内存中没有足够大的连续空闲块来容纳一个段，我们将无法为该段分配空间，因此无法满足分配请求。这是GC抛出OOM（内存不足）异常的少数几个合法情况之一（准确地说，GC实际上并不抛出任何异常——它是代表你发出分配请求的执行引擎抛出异常——GC只是对分配请求返回NULL）。

我经常被问到这个问题：“为什么我的堆只有X MB时，我还会收到OOM？”这里的X远小于2GB。请注意，这是在x86平台上，而且他们所说的“堆”是指托管堆（我应该感到骄傲，现在许多人已经习惯于托管应用程序，以至于他们只使用“堆”来指代托管堆）。

记住，除了GC之外，还有其他分配。GC与其他所有东西一样，竞争虚拟内存空间。你进程中加载的模块需要占用虚拟内存；进程中的某些模块可能会进行本机分配，这也会占用虚拟内存（VirtualAlloc、HeapAlloc、new等等）。CLR本身也会为本机代码、CLR需要的数据结构（包括GC执行工作所需的数据结构）等进行本机分配。通常，CLR进行的分配应该相当小。你可以使用SOS调试器扩展dll的!eeheap命令来查看CLR进行的各种类别的分配。

检查托管堆

在此之前，我想提一下关于获取堆性能数据的几个要点：

获取Perfmon日志时，建议使用尽可能短的时间间隔（目前是1秒），并捕获几分钟的日志。通常，这比以5或10秒的间隔捕获2小时的日志能提供更多的洞察力（这是我经常看到人们在做的事情）。如果有什么异常行为，通常在几分钟内就能看到模式。当然，如果问题随机复现，你只能长时间记录，但最好保持间隔短，因为GC通常足够频繁，需要短间隔才能更有意义地解读日志。
如果要为进程获取内存转储，请获取完整内存转储。对于分析托管堆来说，小型转储通常没什么用。
使用CLRProfiler时，可以取消“Profiler Active”复选框，这样在应用程序运行到你想收集数据的地方之前，Profiler不会运行。如果只需分析分配，可以取消“Calls”复选框以收集更少的数据。
那么，在检查托管堆时，你希望寻找哪些类型的事情呢？

GC中的时间占比：如果GC中的时间占比很低，那么寻找分配相关的问题可能没什么用。当GC中的时间占比很高时，通常意味着你进行了太多的Gen2收集，或者每个Gen2收集花费的时间太长。这时候，是时候更详细地查看你的堆，以确定为什么你进行了这么多的Gen2收集，以及如何改变你的分配模式以减少在Gen2收集上的时间。
堆增长模式：如果你有一个需要长时间运行的服务器应用程序，并且观察到堆随时间无限制地增长，这绝对是一个坏兆头。通常，服务器应用程序应该在请求完成后清理所有相关数据。如果不是这样，你有一个内存泄漏，必须解决这个问题，否则你将收到OOM。
CLRProfiler可以提供最全面的信息。如果可以运行应用程序并使用它，那太好了；如果不能，因为它太重，可以采取一些内存转储（如果你不能实时调试应用程序），并比较堆。我建议你在GC之后立即获取转储，以获得更准确的比较。否则，你可能会看到大多数对象尚未清理的堆，以及刚刚进行了完全GC的堆，其中没有死亡对象。
Perfmon也可以非常有助于发现一些明显的问题，例如：如果GC句柄的数量持续增长，这可能表明你有一个句柄泄漏。

Remember that there are always allocations that are made not by GC. GC competes for the VM space just like anything else. Modules you load in your process need to take up VM space; some modules in your process could be making native allocations which also take up VM space (VirtualAlloc, HeapAlloc, new and whatnot). CLR itself makes native allocations as well for jitted code, datastructures the CLR needs (including the ones that GC needs to do its work) and etc. Usually the allocations CLR makes should be pretty small. You can use the !eeheap command from the SOS debugger extension dll to look at various categories of allocations that the CLR makes. Following is a simplified sample output with some comments in []’s:

The app is getting OOM at this point. Let’s look at the free VM blocks. You can achieve this by using the !vadump command or any other tool that analysises the VM space. My favorite is the !address command which nicely prints out the largest free region:

记住，总是有一些分配不是由GC（垃圾回收器）进行的。GC与其他所有东西一样，竞争虚拟内存空间。你的进程加载的模块需要占用虚拟内存空间；进程中的某些模块可能会进行本机分配，这也占用虚拟内存空间（VirtualAlloc、HeapAlloc、new等等）。CLR（公共语言运行时）本身也会为本机代码、CLR需要的数据结构（包括GC工作所需的数据结构）等进行本机分配。通常，CLR进行的分配应该是相当小的。你可以使用SOS调试器扩展dll中的!eeheap命令来查看CLR进行的各种类别的分配。以下是一个简化的示例输出，其中包含一些注释：

应用程序在这个时候收到了OOM（内存不足）异常。让我们来看看空闲的虚拟内存块。你可以使用!vadump命令或其他分析虚拟内存空间的工具来实现这一点。我最喜欢的是!address命令，它可以很好地打印出最大的空闲区域：

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
 
前两个段是冻结字符串的只读段，这就是为什么与其他段相比它们看起来有点奇怪。begin和segment的地址非常不同，通常它们都是很小的段（除非你有大量的冻结字符串）]
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

应用程序此时遇到了内存不足异常（OOM）。为了诊断问题，我们需要检查空闲的虚拟内存块。这可以通过使用!vadump命令或任何其他分析虚拟内存空间的工具来完成。我最喜欢的命令是!address，它清晰地输出了虚拟内存空间中最大的空闲区域：

（请记住，即使有大量的空闲内存，如果应用程序尝试分配的单个大块内存大于任何空闲区域（这种情况称为外部碎片），它仍然可能会遇到OOM。

为了进一步调查OOM的原因，您可能还想要：使用!eeheap -gc命令检查托管堆中的大对象堆（LOH）碎片）
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

最大的空闲区域小于64MB，这解释了为什么我们会遇到OOM异常——我们试图分配另一个GC段，但未能成功（这是在服务器GC上，段的大小更大）。

如果频繁加载和卸载可变大小的模块，虚拟内存空间可能会严重碎片化。一个常见的场景是，如果您有COM DLL，在其中的组件一段时间（我认为是10分钟？）未使用后会被卸载。其他常见场景是大量的互操作程序集，或者对于web服务器来说，大量的微小DLL（如果您有10k个DLL，它将在虚拟内存中消耗64k）。

除了为段分配虚拟内存空间外，GC真的不关心虚拟内存的其他任何事情。另一方面，物理内存完全是另一回事。正如我在《高效使用GC - 第1部分》中提到的，GC可能发生的3种情况之一是如果您的机器物理内存不足。GC会变得更加积极。因此，如果您正在查看GC性能计数器，您可能会注意到，当您的机器内存不足时，gen0、gen1和gen2的比例会有所不同。如果GC无法提交所需的内存来满足您的分配请求，您也会遇到OOM。可用的物理内存可以通过性能监视器计数器Memory\Available Bytes获取。

值得提到的一些事项包括：

现在拥有超过2GB物理内存的机器并不少见。因此，不要过度碎片化虚拟内存真的很重要。然而，如果您在64位系统上运行，虚拟内存空间很大，物理内存成为了限制因素。
您可以使用Whidbey托管API来通知GC，即使您实际上并不处于低内存情况，也想要它认为您处于低内存情况，这是因为您只想将部分物理内存分配给这个特定的应用程序，或者您想要稳妥起见，不担心达到可能开始遇到OOM的点（许多应用程序无法处理OOM）。

Looking at a managed heap 

Before we get into that there are a few things I’d like to mention about taking the perf data for your heap:

Getting Perfmon logs

I recommend when you take GC perfmon counters you use the shortest interval possible (right now it’s 1 second) and capture the perfmon log for a few minutes. Usually it gives you much more insight than capturing a log for 2 hours with 5 or 10 seconds interval (which I often see people do when they send me perfmon logs) because if something is misbehaving it’s usually enough to see the pattern after a few minutes. However if the problem repros randomly of course you have no choice but to log for a possibly long time but it’s best to keep the interval short because GC usually happens frequent enough that you need a short interval to make more sense out of the log.

Taking memory dumps

If you take a memory dump for your process, take a full memory dump. A mini dump for analysing the managed heap is usually useless.

Getting CLRProfiler logs

CLRProfiler is very heavy weight and it can’t attach to a process but you can uncheck the Profiler Active checkbox so it’s not on when the app is not running to the point that you are interested in gathering data. You can also uncheck the Calls checkbox to gather less data if just profiling Allocations is enough for you.

在查看托管堆之前，我想先提几点关于收集堆性能数据的建议：

获取Perfmon日志

我建议在收集GC性能监视器计数器时，使用尽可能短的时间间隔（目前是1秒），并捕获几分钟的性能监视器日志。通常，这种方式比以5或10秒间隔捕获2小时的日志能提供更多的洞察（我经常看到人们这样做，当他们发送给我性能监视器日志时）。因为如果有什么行为不正常，通常在几分钟内就能看到模式。然而，如果问题是随机复现的，那么你只能选择可能长时间记录，但最好保持间隔短，因为GC通常足够频繁，你需要一个短间隔才能从日志中得出更多意义。

获取内存转储

如果你为你的进程获取内存转储，请获取完整的内存转储。对于分析托管堆来说，小型转储通常是无用的。

获取CLRProfiler日志

CLRProfiler非常重量级，它不能附加到进程中，但你可以取消勾选“Profiler Active”复选框，这样在应用程序没有运行到你感兴趣的数据收集点时，它就不会处于活动状态。如果你只是分析分配就足够了，你还可以取消勾选“Calls”复选框来收集更少的数据。
———————————————————–

So what kind of things do you want to look for when you look at a managed heap?

Time in GC

If the % Time in GC is low, it’s not useful to look for allocation related issues. When % Time in GC is high it usually means you are doing too many Gen2 collections or each Gen2 collection takes too long. Then it’s time to start looking at your heap in more details to determine why you are doing so many Gen2 collections and how you can change your allocation pattern to spend less time on Gen2 collections.

The heap growth pattern

If you have a server app that needs to stay running for a long time, and if you observe that the heap is growing unbounded overtime that’s definitely a bad sign. Usually server apps should clean up all data related to a request after the request is finished. If that’s not the case you’ve got a memory leak that you must look at – otherwise you will get OOM. This is usually the 1st thing we ask app developers (especially server app devs) to fix.

CLRProfiler gives you the most extensive info. If you can run your app with it, great; if you can’t, because it’s too heavyweight, you could take some memory dumps (if you don’t have the luxury to debug the app live) and compare the heaps. I would suggest you to take the dumps right after a GC so you get more acccurate comparision. Otherwise you could be looking at one heap where most objects haven’t been cleaned up yet and another one where no dead objects exist because it just did a full GC.

Perfmon can also be really useful to pinpoint some obvious things such as: if the # of GC Handles keeps growing, it indicates you may have a handle leak. For more info on GC perfmon counter please refer to my previous blog entry.

当你查看托管堆时，你想寻找哪些类型的问题？

GC占用的时间

如果GC占用的时间百分比很低，那么寻找与分配相关的问题可能没有太大用处。当GC占用的时间百分比很高时，通常意味着你进行了太多的Gen2垃圾回收，或者每个Gen2垃圾回收耗时过长。这时候，你需要更详细地查看你的堆，以确定为什么你会进行这么多的Gen2垃圾回收，以及你如何改变分配模式以减少在Gen2垃圾回收上的时间。

堆增长模式

如果你有一个需要长时间运行的服务器应用程序，并且你观察到堆随着时间的推移无限制地增长，这绝对是一个不好的迹象。通常，服务器应用程序应该在请求完成后清理所有与请求相关的数据。如果不是这样，那么你有一个内存泄漏，你必须检查它——否则你将遇到OOM。这通常是我们要求应用程序开发人员（尤其是服务器应用程序开发人员）首先要修复的问题。

CLRProfiler提供了最全面的信息。如果你可以用它来运行你的应用程序，那很好；如果你不能，因为它太重，你可以采取一些内存转储（如果你没有实时调试应用程序的便利）并比较堆。我建议你在GC之后立即进行转储，这样你可以得到更准确的比较。否则，你可能会看到一个是大多数对象尚未清理的堆，而另一个则是刚刚进行了完全GC，不存在死亡对象的堆。

Perfmon也可以非常有助于精确地指出一些明显的问题，例如：如果GC句柄的数量持续增长，这可能表明你有一个句柄泄漏。有关GC性能监视器计数器的更多信息，请参考我的上一篇博客文章。

The full GC  ratio

Often we tell people a healthy gen2:gen1 ratio is 1:10. If you are seeing a 1:1 ratio that’s certainly something to look into. When you have a really large heap and doing a gen2 could take some time it’s of course ideal to make the ratio of gen2 as low as possible. As with all perf advices there are always exceptions – for example if you have a heap where most objects are on the LOH and they don’t really move – we’ve seen this in certain server apps where they use some really arrays that live on the LOH to hold references to small objects and these arrays stay pretty much for the process lifetime, it’s not necessarily a bad situation. We do need to trace through the references that these large arrays contain but we don’t need to move memory around for them which would be an expensive operation. On the other hand, if you have an app that creates lots of temporary large objects you are likely to be in a bad situation if you also have a fairly big gen2 heap because the gen2’s that the large object allocations trigger will also need to collect your gen2 heap.

The common causes for doing a relatively high number of gen2’s include a managed cache implementation where the whole cache is in gen2 and it constantly gets churned; or if you have lots of finalizable objects whose finalizers get to run which increases the chance these objects make into gen2.

We’ve seen people who induce GCs, ie, calling GC.Collect, periodically. First of all, if it’s to “help not get OOMs” because you believe that GC is not kicking in early enough (if you are getting OOM because you have too many live objects and you exhausted memory it won’t be helped by inducing GCs anyway), you should tell us about it. Secondly, inducing GCs screws up GC’s tuning which is almost always a bad thing.

Fragmentation

Fragmentation is the free space you have on your managed heap which you can obtain by using the sos command

!dumpheap –type Free –stat

Of course the smaller the fragmentation the better but in the real world scenarios when you use IO and etc you will get pinned objects from the .net framework. Fragmenatation in different generations has different indications to how healthy your app is:

Fragmentation in gen0 is good because gen0 is where you allocate your objects and we will use the free space in gen0 to satisfy your allocation requests. As a hypothetical example, If we are looking a heap where all the Free objects are in gen0 that’s a very very ideal heap picture. You can specify a begin and an end address for !dumpheap to see how much space Free objects occupy in gen0.  Take the example from the dump mentioned at the beginning:

完全GC比例

我们经常告诉人们，一个健康的Gen2:Gen1比例是1:10。如果你看到的是1:1的比例，那肯定是要调查的事情。当你有一个非常大的堆，进行Gen2垃圾回收可能需要一些时间，因此理想情况下是尽可能降低Gen2的比例。就像所有性能建议一样，总是有例外——例如，如果你有一个大部分对象都在大对象堆（LOH）上的堆，并且它们实际上不移动——我们已经在某些服务器应用程序中看到了这种情况，这些应用程序使用一些真正的大型数组来持有对小对象的引用，这些数组几乎在整个进程生命周期中都存在，这不一定是一个糟糕的情况。我们确实需要追踪这些大型数组包含的引用，但我们不需要为它们移动内存，这将是一个昂贵的操作。另一方面，如果你有一个创建大量临时大对象的应用程序，并且你还有一个相当大的Gen2堆，那么你可能会处于一个糟糕的情况，因为大对象分配触发的Gen2垃圾回收也需要收集你的Gen2堆。

进行相对较高数量的Gen2垃圾回收的常见原因包括一个将整个缓存放在Gen2中的托管缓存实现，并且它不断被翻新；或者如果你有大量可终结化的对象，它们的终结器有机会运行，这增加了这些对象进入Gen2的机会。

我们见过一些人定期诱导GC，即调用GC.Collect。首先，如果这样做是为了“帮助避免OOM”，因为你认为GC没有尽早启动（如果你因为活对象太多耗尽了内存而遇到OOM，无论如何诱导GC都不会有帮助），你应该告诉我们这一点。其次，诱导GC会干扰GC的调优，这几乎总是一件坏事。

碎片化

碎片化是你托管堆上的空闲空间，你可以使用sos命令来获取：

!dumpheap –type Free –stat

当然，碎片越小越好，但在现实世界的场景中，当你使用IO等时，你会从.NET框架中得到固定对象。不同代中的碎片化对你应用程序的健康状况有不同的指示：

Gen0中的碎片化是好的，因为Gen0是你分配对象的地方，我们将使用Gen0中的空闲空间来满足你的分配请求。作为一个假设的例子，如果我们正在查看一个所有空闲对象都在Gen0中的堆，那是一个非常非常理想的堆图。你可以为!dumpheap指定开始和结束地址，以查看空闲对象在Gen0中占据多少空间。以下是从前面提到的转储中取得的例子：

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

所以Gen0是9MB，大约有7MB是空闲空间，即碎片化。

大对象堆（LOH）中的碎片化是设计使然，因为我们不压缩LOH，但这并不意味着在LOH上分配等同于使用NT堆管理器的malloc！由于GC的工作方式，相邻的空闲对象会自然地合并成一个大的空闲空间，这个空间可用于满足你的大对象分配请求。

糟糕的碎片化是发生在Gen2和Gen1中的。如果在GC之后，你仍然在Gen2和Gen1中看到大量的空闲空间（特别是Gen2，因为Gen1的大小不能超过一个段的大小），那肯定是要调查的事情。正如我之前提到的，我们一直在努力解决碎片化问题，并且已经大幅改善了这种情况，但你可以通过观察你的应用程序行为并在代码中尽可能减少碎片化来做你的部分。在《高效使用GC – 第3部分》中，我们讨论了在必须固定对象时使用的好技巧。

对于Gen2，如果碎片化低于20%，则被认为是非常好的。因为Gen2可能会变得非常大，所以考虑碎片化比例而不是绝对值是很重要的。

好了，今天的内容就到这里。下次再见！

[编辑于2005年5月16日 – 我发现我实际上把“efficiently”拼错了（你会以为我能拼写它，但是没有…），所以我也就顺便修正了当我第一次将文本从Word粘贴到编辑器进行博客时弄乱的格式。]

https://devblogs.microsoft.com/dotnet/using-gc-efficiently-part-4/


