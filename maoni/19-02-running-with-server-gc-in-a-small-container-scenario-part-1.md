<h1>Running with Server GC in a Small Container Scenario Part 1 – Hard Limit for the GC Heap</h1>

I’ve checked in 2 configs related to specifying a hard limit for the GC heap memory usage so I wanted to describe them and how they are intended to be used. Feedback would be greatly appreciated.

In order to talk about the new configs it’s important to understand what consists of the memory usage in a process. When it comes to memory the terminology can be very confusing so it’s worth pointing out that by “memory” I meant private commit size. A process can have the following memory usage from the runtime’s POV:

1.GC heap usage – this includes both the actual GC heap and the native memory used by the GC itself to do its bookkeeping;

2.native memory used by other things in the runtime such as the jitted code heap;

3.native memory used by something that’s not the runtime, eg, you are allocating native memory in your pinvoke calls;
We cannot control 3) and it could be significant. 2) is in general small compared to 1).

When a process is running inside a container, and if you have specified a memory limit on the container, the process is not supposed to use more than that much memory. However, as we mentioned the GC heap usage is not the only memory contender in the process so by default we specify this as the hard limit for the GC heap usage inside a container

```
max (20mb, 75% of the memory limit on the container)
```

So we leave 25% to other types of memory usage. Of course that may be too little, or too much. So we also provide 2 configs that you can specify if you want a more fine grained control. You could specify this by either an absolute amount or as a percentage of the total memory limit the process is allowed to use. Below is from my PR: GCHeapHardLimit – specifies a hard limit for the GC heap GCHeapHardLimitPercent – specifies a percentage of the physical memory this process is allowed to use

GCHeapHardLimit is checked first and only when it’s not specified would we check GCHeapHardLimitPercent. This means if you specify both we’d take what you specified via GCHeapHardLimit.

If one of the HardLimit configs is specified, and the process is running inside a container with a memory limit, the GC heap usage will not exceed the HardLimit specified but the total memory is still the memory limit on the container so when we calculate the memory load it’s based off the container memory limit.

An example,

process is running inside a container with 200mb limit and user also specified GCHeapHardLimit as 100mb.

if 50mb out of the 100mb is used for GC, and 100mb is used for other things, the memory load is (50 + 100)/200 = 75%.

I also made some changes to the policy to accommodate the container scenario better. For example, if a hard limit is specified (including running inside a container that has a memory limit specified, as indicated above), and you are using Server GC, by default we do not affinitize the Server GC threads like we do when there’s no hard limit specified. Because we mostly target the container scenario with this and we know it’s perfectly common to have multiple containers running on the same machine, it wouldn’t make sense to hard affinitize by default. However, you could certainly use the existing GCHeapAffinitizeMask config to specify which CPUs you want the GC threads to affinitize to.

In order to not run into the situation where your container runs on a machine with many cores, has a very small memory limit specified but no CPU limit specified, we made it so that the default segment size is at least 16mb – segment size is calculated as (hard limit / number of heaps). So you wouldn’t have so many heaps where each heap only handles a tiny amount of memory. For example if you specified 160mb memory limit on your container on a machine with 40 cores, you would get 10 heaps with 16mb segments instead of 40 heaps with 4mb segments. Again, you can change the number of heaps yourself with the GCHeapCount config.

When specifying a hard limit on a GC heap, obviously you still want acceptable perf. You wouldn’t want to be in the situation where the limit is so small that gives no room for your process to run, ie, you do not want to have a limit that’s just barely larger than your live data size. Live data size is defined as the size objects on your heap occupy after a full blocking GC. If your live data is say 100mb and you specify 128mb as your limit, that gives only 28mb for temporary data and let’s say each of your request allocates 5mb of temporary data, that means at the minimum, a GC has to happen after about 5 requests. So specifying a sensible limit means you need to understand how much live data vs temporary data you have and how much time you are willing to let the GC take. We are also doing some more perf testing to see if we should make changes to our default number of heaps when the memory limit is low.

We do understand that you might want even more control over your GC heap usage. For example, if your process also has a native cache and you’d like to have the flexibility to have the GC heap use less during some period of time so the native cache could cache more, and vice versa. We’ll have a proposal on this soon. Stay tuned.

https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-1-hard-limit-for-the-gc-heap/我最近提交了两个与为 GC 堆内存使用指定硬限制相关的配置，因此我想描述一下这些配置以及它们的预期用途。非常欢迎您的反馈。

为了讨论新的配置，了解进程中的内存使用组成非常重要。关于内存的术语可能会非常令人困惑，所以值得指出的是，我所说的“内存”指的是私有提交大小（private commit size）。从运行时的角度来看，一个进程可以有以下几种内存使用：

GC 堆使用 —— 这包括实际的 GC 堆以及 GC 本身用于簿记的本机内存；
运行时中其他部分使用的本机内存，例如 JIT 编译代码堆；
非运行时组件使用的本机内存，例如你在 P/Invoke 调用中分配的本机内存；
我们无法控制第 3 类内存，而且它可能是显著的。通常情况下，第 2 类内存相比第 1 类要小得多。

当一个进程在容器内运行，并且你为该容器指定了内存限制时，该进程不应该使用超过这个限制的内存。然而，正如我们提到的，GC 堆使用并不是进程中唯一的内存竞争者。因此，默认情况下，我们将容器内存限制的以下值指定为 GC 堆使用的硬限制：

深色版本
max(20MB, 容器内存限制的 75%)
这样我们就留出了 25% 的内存供其他类型的内存使用。当然，这可能太少了，也可能太多了。因此，我们还提供了两个配置项，如果你需要更精细的控制，可以使用这些配置。你可以通过绝对值或作为进程允许使用的总内存限制的百分比来指定。以下是来自我的 PR 的相关内容：

GCHeapHardLimit —— 指定 GC 堆的硬限制；
GCHeapHardLimitPercent —— 指定进程允许使用的物理内存的百分比。
首先会检查 GCHeapHardLimit，只有当它未被指定时才会检查 GCHeapHardLimitPercent。这意味着如果你同时指定了两者，我们会采用你通过 GCHeapHardLimit 指定的值。

如果指定了其中一个硬限制配置，并且进程在具有内存限制的容器中运行，则 GC 堆使用不会超过指定的硬限制，但总内存仍然是容器的内存限制。因此，在计算内存负载时，它是基于容器的内存限制进行计算的。

示例：
假设一个进程在一个内存限制为 200MB 的容器中运行，用户还指定了 GCHeapHardLimit 为 100MB。

如果 GC 使用了 100MB 中的 50MB，而其他用途使用了 100MB，则内存负载为：

(
50
+
100
)
/
200
=
75
%
(50+100)/200=75%
我还对策略做了一些更改，以更好地适应容器场景。例如，如果指定了硬限制（包括在具有内存限制的容器中运行，如上所述），并且你正在使用 Server GC，
默认情况下我们不会像没有硬限制时那样将 Server GC 线程绑定到特定 CPU。这是因为我们主要针对容器场景，而在同一台机器上运行多个容器是很常见的，
因此默认情况下硬绑定是没有意义的。不过，你仍然可以使用现有的 GCHeapAffinitizeMask 配置来指定你希望 GC 线程绑定到哪些 CPU。

为了避免出现这样的情况：你的容器运行在多核机器上，设置了非常小的内存限制但没有设置 CPU 限制，我们确保默认的段大小至少为 16MB——段大小是根据（硬限制 / 堆数量）计算的。
因此，你不会得到太多堆，每个堆只处理少量内存。例如，如果你在一台 40 核的机器上的容器中设置了 160MB 的内存限制，你会得到 10 个堆，每个堆的段大小为 16MB，而不是 40 个堆，每个堆的段大小为 4MB。当然，你也可以通过 GCHeapCount 配置自己调整堆的数量。

当你为 GC 堆指定硬限制时，显然你仍然希望获得可接受的性能。你不希望处于这样的情况：限制太小，以至于几乎没有空间让进程运行，也就是说，你不希望限制仅略大于你的活动数据大小。活动数据大小定义为全阻塞 GC 后堆上对象占用的大小。例如，如果你的活动数据是 100MB，并且你将限制设置为 128MB，那么只剩下 28MB 用于临时数据。假设每个请求分配 5MB 的临时数据，这意味着最少每 5 个请求就需要触发一次 GC。因此，指定一个合理的限制意味着你需要了解你的活动数据和临时数据的比例，以及你愿意让 GC 占用多少时间。我们还在进行更多的性能测试，看看是否应该在内存限制较低时调整默认的堆数量。

我们理解你可能希望对 GC 堆使用拥有更多的控制权。例如，如果你的进程还有一个本机缓存，并且希望在某些时间段内让 GC 堆使用更少的内存，以便本机缓存可以缓存更多数据，反之亦然。我们很快会有一个关于这方面的提案，请持续关注。

