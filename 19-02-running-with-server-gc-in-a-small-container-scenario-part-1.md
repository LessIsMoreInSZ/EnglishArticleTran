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

https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-1-hard-limit-for-the-gc-heap/