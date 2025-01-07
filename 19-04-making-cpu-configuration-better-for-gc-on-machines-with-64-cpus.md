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

