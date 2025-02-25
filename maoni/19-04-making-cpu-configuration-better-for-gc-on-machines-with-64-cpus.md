<h1>Making CPU configuration better for GC on machines with > 64 CPUs</h1>

If you are running Windows on a machine with > 64 CPUs, you’ll need to use this feature called the CPU groups for your process to be able to use more than 64 CPUs. At some point in the far distant past, people thought having more than 64 processors on a machine was inconceivable so they used a 64-bit number for the processor mask. And when 64-proc machines became available, Windows invented this CPU group concept for such machines which says processors are now belong to different CPU groups where each group has no more than 64 procs. Eg, on a machine with 96 procs you will see 2 groups with 48 procs each. When a process starts, it always starts in a single CPU group. The only way to use processors from other groups is for this process to set its thread affinity so it will run on processors from other groups. When machines with > 64 procs became available, we added a config called GCCpuGroup. The default for it is 0 (ie, enabled=false) meaning without this config, if you are using Server GC, the process would create at most N Server GC threads/heaps where N is the # of processors in the single group it started in. When this is set to 1, GC will create Server GC threads that span all active processors in all available CPU groups on the machine. Then our runtime started to run on Linux and with that we introduced an OS layer that GC calls via GCToOSInterface which abstracted away the OS functionalities GC needed, eg, VirtualAlloc and processor affinity. And we had our Linux OS layer simulated the Windows behavior. For the most part this was desired but there’s one thing that became more and more a thorny point – it’s the CPU group concept. Linux does not have this concept – if you have > 64 processors you will get to use 64 procs without doing anything special. And we had to write all this code in our Linux OS layer to group processors into CPU groups. Recently we decided to pull the plug on this and no longer have Linux simulate the Windows behavior ‘cause we like the Linux behavior better for this particular case. I’ve been working with Jan Vorlicek on this (he’s doing all the work). The main pieces of this work are the following – 1) GC will no longer have the concept of CPU groups. Checks like this:

        if (GCToOSInterface::CanEnableGCCPUGroups())
            set_thread_group_affinity_for_heap(heap->heap_number, &affinity);
        else
            set_thread_affinity_mask_for_heap(heap->heap_number, &affinity);

view rawcpugroup.cpp hosted with ❤ by GitHub

will be removed. If the process needs to use > 64 procs it will be handled by the OS layer automatically. In fact, we are even thinking about changing the default of the GCCpuGroup from 0 to 1 so on coreclr on Windows you will no longer need to specify this config to have your process using more than 64 processors. As always, we welcome your feedback on this. 2) Previously, I explained the GCHeapAffinitizeMask config in my blog. Since that’s also a 64-bit number (on 64-bit OSs), it was designed for the single CPU group case. We are adding a new config, GCHeapAffinitizeRanges, that allows you to specify processor ranges instead of a mask and it allows you to specify more than 64 procs if you wish. From Jan’s PR:
```
// Unix:
//  The cpu index ranges is a comma separated list of indices or ranges of indices (e.g. 1-5).
//  Example 1,3,5,7-9,12
// Windows:
//  The cpu index ranges is a comma separated list of group-annotated indices or ranges of indices.
//  The group number always prefixes index or range and is followed by colon.
//  Example 0:1,0:3,0:5,1:7-9,1:12
```
We need different formats for Windows and Linux because Windows simply does not expose global indices of processors – they have to be relative to the group they belong to. Previously on Windows, if you specified an affinity mask it would simply be ignored when you also specified to use CPU groups. With the new

GCHeapAffinitizeRanges config you will be able to specify any processors on the machine, whether it has more than 64 procs or not, and whether you want to have your process use more than 64 procs or not. On Windows, if we do change the default of GCCpuGroup to 1, it means you will automatically using processors in all CPU groups. And if you want the previous behavior you can just set GCCpuGroup to 0. So our proposed new behavior would be – * When GCCpuGroup is 0, we read the GCHeapAffinitizeMask config; * When GCCpuGroup is 1, we read the GCHeapAffinitizeRanges config. * GCCpuGroup will be default to 1 and only be applicable on Windows. Specifically –

On Windows

When GCCpuGroup is 1, GCHeapAffinitizeRanges will be used to pick the processors to run on and GCHeapAffinitizeMask is ignored.
When GCCpuGroup is 0 (default), GCHeapAffinitizeMask will be used to pick the processors within the CPU group the process runs in, and GCHeapAffinitizeRanges is ignored

On Linux

Our Linux OS layer will not have the CPU group concept anymore (all code there will be removed). So the GCCpuGroup config is ignored on Linux, which *also* means the GCHeapAffinitizeMask config is ignored. Note that on 2.2 and prior, our Linux OS layer only supported running on the first 64 procs that you process is allowed to run on. So essentially it was run with always GCCpuGroup as false. The new behavior will always automatically use > 64 procs and you can use the GCHeapAffinitizeRanges config to specify <= 64 procs if you wish.
Curious minds will also notice in the vicinity of the

GCCpuGroup config in src\clrconfigvalues.h there’s a config called GCNumaAware. This was added the same time when we added the GCCpuGroup config. By default we always enabled NUMA. This means we allocate memory on the proper NUMA node and when we do heap balancing we will try to balance allocations to heaps that live on the same NUMA node first before we look at heaps on remote nodes. I’m thinking of just getting rid of this config altogether – we had it for testing when we made GC NUMA aware years ago but I don’t see any reason why anyone would want to be NUMA unware so there’s no need for it anymore.

EDIT on 04/04/2019 – we needed 2 different formats for on Windows and Linux for the GCHeapAffinitizeRanges config. EDIT on 09/23/2019 – on Windows we kept the default as disabled for GCCpuGroup.

如果你在一台具有超过 64 个 CPU 的机器上运行 Windows，你需要使用一个名为 CPU 组 的功能，才能让你的进程使用超过 64 个 CPU。很久以前，人们认为一台机器拥有超过 64 个处理器是不可想象的，因此他们用一个 64 位的数字来表示处理器掩码。当 64 核机器出现时，Windows 引入了 CPU 组 概念，将处理器分配到不同的 CPU 组中，每个组最多包含 64 个处理器。例如，在一台有 96 个处理器的机器上，你会看到两个组，每个组有 48 个处理器。当一个进程启动时，它总是从单个 CPU 组开始运行。要使用其他组中的处理器，这个进程必须设置线程亲和性（affinity），以便它可以运行在其他组的处理器上。

当超过 64 核的机器可用时，我们在 CoreCLR 中添加了一个名为 GCCpuGroup 的配置项。默认情况下，它的值为 0（即 enabled=false），这意味着如果没有启用此配置，即使你使用 Server GC，该进程最多也只能创建与启动时所在的单个组中的处理器数量相等的 Server GC 线程/堆。当 GCCpuGroup 设置为 1 时，GC 将创建覆盖所有可用 CPU 组中所有活动处理器的 Server GC 线程。

后来，我们的运行时开始支持 Linux，并为此引入了一个 OS 层，GC 通过 GCToOSInterface 调用它，抽象了 GC 所需的操作系统功能，例如 VirtualAlloc 和处理器亲和性。我们在 Linux OS 层模拟了 Windows 的行为。总体来说这是符合预期的，但有一个问题变得越来越棘手——那就是 CPU 组 概念。Linux 并没有这个概念——如果你有超过 64 个处理器，你可以直接使用其中的 64 个，而不需要做任何特殊处理。为了模拟 Windows 的行为，我们不得不在 Linux OS 层编写大量代码，将处理器分组为 CPU 组。最近，我们决定停止这种模拟行为，因为在这种特定情况下，我们更喜欢 Linux 的行为。我正在与 Jan Vorlicek 合作完成这项工作（他负责具体实现）。这项工作的主要部分如下：

1. GC 不再需要 CPU 组的概念
像以下这样的检查代码将被移除：

cpp
深色版本
if (GCToOSInterface::CanEnableGCCPUGroups())
    set_thread_group_affinity_for_heap(heap->heap_number, &affinity);
else
    set_thread_affinity_mask_for_heap(heap->heap_number, &affinity);
如果进程需要使用超过 64 个处理器，这将由 OS 层自动处理。事实上，我们甚至在考虑将 GCCpuGroup 的默认值从 0 改为 1，这样在 Windows 上使用 CoreCLR 时，你不再需要显式指定此配置来让进程使用超过 64 个处理器。一如既往，我们欢迎你的反馈。

2. 新的配置项：GCHeapAffinitizeRanges
之前我在博客中解释过 GCHeapAffinitizeMask 配置项。由于这是一个 64 位数字（在 64 位操作系统上），它只适用于单个 CPU 组的情况。我们现在引入了一个新的配置项 GCHeapAffinitizeRanges，允许你指定处理器范围而不是掩码，并且可以指定超过 64 个处理器。以下是 Jan 的 PR 中的格式说明：

Unix/Linux:
深色版本
cpu 索引范围是一个逗号分隔的索引或索引范围列表（例如 1-5）。
示例：1,3,5,7-9,12
Windows:
深色版本
cpu 索引范围是一个逗号分隔的、带有组注释的索引或索引范围列表。
组号始终作为前缀，后面跟着冒号。
示例：0:1,0:3,0:5,1:7-9,1:12
我们需要为 Windows 和 Linux 使用不同的格式，因为 Windows 并不暴露全局处理器索引——它们必须相对于所属的组。

在 Windows 上，之前如果指定了亲和性掩码并且启用了 CPU 组，掩码会被忽略。而使用新的 GCHeapAffinitizeRanges 配置项后，你可以指定机器上的任何处理器，无论是否超过 64 个，无论你是否希望进程使用超过 64 个处理器。

新的行为提案：
当 GCCpuGroup 为 0 时，读取 GCHeapAffinitizeMask 配置；
当 GCCpuGroup 为 1 时，读取 GCHeapAffinitizeRanges 配置；
GCCpuGroup 默认值为 1，仅适用于 Windows。
具体行为如下：

Windows:
当 GCCpuGroup 为 1 时，使用 GCHeapAffinitizeRanges 来选择运行的处理器，忽略 GCHeapAffinitizeMask。
当 GCCpuGroup 为 0（默认值）时，使用 GCHeapAffinitizeMask 来选择进程所在 CPU 组内的处理器，忽略 GCHeapAffinitizeRanges。
Linux:
我们的 Linux OS 层将不再有 CPU 组的概念（所有相关代码将被移除）。因此，GCCpuGroup 配置在 Linux 上会被忽略，这也意味着 GCHeapAffinitizeMask 配置也会被忽略。
注意：在 2.2 及之前的版本中，我们的 Linux OS 层仅支持运行在进程允许使用的前 64 个处理器上，相当于总是以 GCCpuGroup 为 false 运行。新行为将自动支持超过 64 个处理器，如果你想限制为 <= 64 个处理器，可以使用 GCHeapAffinitizeRanges 配置。
3. 关于 GCNumaAware 配置
在 src\clrconfigvalues.h 文件中，靠近 GCCpuGroup 配置的地方还有一个名为 GCNumaAware 的配置项。它是在我们添加 GCCpuGroup 配置项时同时引入的。默认情况下，我们总是启用 NUMA。这意味着我们在正确的 NUMA 节点上分配内存，并且在进行堆平衡时，我们会优先尝试在同一 NUMA 节点上的堆之间平衡分配，然后再考虑远程节点上的堆。

我正在考虑完全移除这个配置项——我们当时为了测试 GC 对 NUMA 的支持才引入了它，但我看不到任何人会希望禁用 NUMA 支持，因此这个配置项已经没有必要了。

编辑更新：
2019 年 4 月 4 日：对于 GCHeapAffinitizeRanges 配置，Windows 和 Linux 需要两种不同的格式。
2019 年 9 月 23 日：在 Windows 上，我们将 GCCpuGroup 的默认值保持为禁用状态（disabled）。