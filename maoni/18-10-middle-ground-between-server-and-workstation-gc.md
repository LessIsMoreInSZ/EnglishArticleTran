<h1>Middle Ground between Server and Workstation GC</h1>

A long time ago I wrote about using Workstation GC on server applications when you have many instances of your server app running on the same machine. By default Server GC will treat the process as owning the machine so it uses all CPUs to do the GC work. More and more folks find themselves in a situation where they might have a few active instances of their server application running on the same machine. You could of course run each instance inside a container (a job object on Windows or cgroups on Linux) to limit resources that way. For example, if you limit the job object to only use 10 out of 40 CPUs on the machine, naturally Server GC will only create 10 GC threads.

To make it easier for folks to find a middle ground between Server and Workstation GC, in 4.6.2 I added a config to tell Server GC to use fewer CPUs than what’s on the machine. The rollup for Server 2012 R2 (KB4014604) describes this config (apparently this particular rollup was removed due to some issue with the installer and the new rollup does not describe the same info; I dunno why this is). In any case this is the text from that link:

————————————————— Fix 5

On a system that has many processors, there might a desire to have some processes use only some processes with server GC. For example if you have a 40-processor system, you might want to have process 0 use only 10 processors and process 1 use only 6 processors, so process 0 gets 10 GC heaps/threads and process 1 gets 6. Below is an example of using these Server GC configurations:

    <configuration>

        <runtime>

            <gcServer enabled=”true”/>

            <GCHeapCount enabled=”6″/>

            <GCNoAffinitize enabled=”true”/>

            <GCHeapAffinitizeMask enabled=”144″/>

        </runtime>

    </configuration>

<GCNoAffinitize enabled=”true”/> specifies to not affinitize the server GC threads with CPUs.

<GCHeapCount enabled=”6″/> 6 specifies that you want 6 heaps. The actual number of heaps is the minimum of the number of heaps you specify and the number of processors your process is allowed to use.

<GCHeapAffinitizeMask enabled=”144″/> This number is in decimal, so this is 0x90, which means using 2 bits as your process mask. The actual number of heaps is the minimum of the number of heaps you specify, the number of processors your process is allowed to use, and the number of set bits in the mask you specify.

—————————————————

By default Server GC threads are hard affinitized with their respective CPUs (we do this because it facilitates better cache usage). And since these configs became available I’ve seen quite a few folks use them but there seems to also be some confusion. So I will attempt to explain by illustrating with some sample scenarios.

Scenario 0

You have 4 processes that use Server GC on your 40-CPU machine and you want each process to use 10 CPUs and of course you want them to not overlap. So process #0 uses CPU 0 through 9; process #1 uses CPU 10 through 19 and so on. You’d want to specify your configs as the following for process #0:

    <gcServer enabled=”true”/>

    <GCHeapCount enabled=”10″/>

    <GCHeapAffinitizeMask enabled=”1023″/>

1023 is 0x3FF which means bits 0 through 9 are set. And specify 1047552 (0xFFC00) for process #1 and so on.

In general you do want to specify this mask; otherwise GC will just affinitize the Server GC threads with the first N CPUs (obviously you could have a scenario where you only have one process using Server GC and are fine with it using CPU 0 through N-1, in which case you don’t need to specify a mask value).

The value you specify for GCHeapCount means exactly that – the # of GC heaps. So this means there will be that many Server GC threads and that many Background Server GC threads (to do Background GCs) created.

Scenario 1

You simply don’t want to hard affinitize the Server GC threads for whatever reason.

    <gcServer enabled=”true”/>

    <GCNoAffinitize enabled=”true”/>

Scenario 2

You don’t want to hard affinitize the Server GC threads for whatever reason and want to limit the # of GC heaps to let’s say 10

    <gcServer enabled=”true”/>

    <GCHeapCount enabled=”10″/>

    <GCNoAffinitize enabled=”true”/>

If you specify GCNoAffinitize as true and a value for GCHeapAffinitizeMask, GCHeapAffinitizeMask will simply not be used.

If you do specify GCNoAffinitize as true, you should make sure your clr.dll is from the May 2018 rollup or newer (KB4103720 for Win2k16 and KB4098972 for Win2k12 R2) ‘cause there was a perf fix for it.

Note that these configs *only* affect the GC behavior – so the difference between specifying GCHeapCount as 10 on a 40-CPU machine and running the process in a job object that’s only allowed to use 10 CPUs is the former will still allow user threads to use all 40 CPUs whereas the latter will limit all threads in your process to only 10 CPUs.

很久以前，我写过一篇关于在服务器应用程序中使用 Workstation GC 的文章，当时讨论的是当同一台机器上运行了多个服务器应用程序实例时的情况。默认情况下，Server GC 会将进程视为独占整台机器的资源，因此它会使用所有 CPU 来完成垃圾回收（GC）工作。然而，越来越多的人发现自己处于这样一种情况：他们可能在同一台机器上运行了几个活跃的服务器应用程序实例。当然，你可以通过将每个实例运行在容器中（Windows 上是作业对象，Linux 上是 cgroups）来限制资源。例如，如果你将作业对象限制为只能使用机器上的 40 个 CPU 中的 10 个，那么 Server GC 自然只会创建 10 个 GC 线程。

为了让人们更容易在 Server GC 和 Workstation GC 之间找到一个中间地带，在 .NET 4.6.2 中，我添加了一个配置项，可以让 Server GC 使用比机器上实际拥有的 CPU 更少的核心数。针对 Windows Server 2012 R2 的更新汇总（KB4014604）描述了这个配置（显然，由于安装程序存在某些问题，该更新汇总被移除，而新的更新汇总没有包含相同的信息；我不知道为什么会这样）。无论如何，以下是从那个链接中提取的文字：

修复 5

在一个拥有大量处理器的系统中，你可能希望某些进程只使用部分处理器进行 Server GC。例如，如果你有一台 40 核的系统，你可能希望进程 0 只使用 10 个核，
而进程 1 只使用 6 个核，这样进程 0 将获得 10 个 GC 堆/线程，而进程 1 获得 6 个。以下是使用这些 Server GC 配置的示例：

xml
深色版本
<configuration>
    <runtime>
        <gcServer enabled="true"/>
        <GCHeapCount enabled="6"/>
        <GCNoAffinitize enabled="true"/>
        <GCHeapAffinitizeMask enabled="144"/>
    </runtime>
</configuration>
<GCNoAffinitize enabled="true"/> 指定不要将 Server GC 线程与特定的 CPU 绑定。
<GCHeapCount enabled="6"/> 指定你想要 6 个堆。实际的堆数量是你指定的堆数量和你的进程允许使用的处理器数量之间的最小值。
<GCHeapAffinitizeMask enabled="144"/> 这个数字是十进制的，所以这里是 0x90，意味着使用 2 位作为进程掩码。实际的堆数量是你指定的堆数量、
你的进程允许使用的处理器数量以及你指定的掩码中设置的位数之间的最小值。
默认情况下，Server GC 线程会被硬绑定到它们各自的 CPU 上（我们这样做是为了更好地利用缓存）。自从这些配置可用以来，我已经看到不少人使用它们，
但似乎也存在一些混淆。因此，我将通过一些示例场景来尝试解释。

场景 0
你在一台 40 核的机器上有 4 个使用 Server GC 的进程，并且希望每个进程使用 10 个 CPU，同时不重叠。例如，进程 #0 使用 CPU 0 到 9，进程 #1 使用 CPU 10 到 19，
依此类推。对于进程 #0，你应该这样指定配置：

xml
深色版本
<gcServer enabled="true"/>
<GCHeapCount enabled="10"/>
<GCHeapAffinitizeMask enabled="1023"/>
1023 是 0x3FF，意味着位 0 到 9 被设置。对于进程 #1，指定 1047552（0xFFC00），依此类推。

通常情况下，你确实需要指定这个掩码；否则，GC 会简单地将 Server GC 线程绑定到前 N 个 CPU 上（显然，你可能会遇到一种情况，
即只有一个进程使用 Server GC 并且你对其使用 CPU 0 到 N-1 没有异议，在这种情况下，你不需要指定掩码值）。

你为 GCHeapCount 指定的值就是 GC 堆的数量。这意味着将创建相应数量的 Server GC 线程和 Background Server GC 线程（用于后台 GC）。

场景 1
无论出于什么原因，你只是不想将 Server GC 线程硬绑定到特定的 CPU 上。

xml
深色版本
<gcServer enabled="true"/>
<GCNoAffinitize enabled="true"/>
场景 2
无论出于什么原因，你不希望将 Server GC 线程硬绑定到特定的 CPU 上，并且希望将 GC 堆的数量限制为例如 10。

xml
深色版本
<gcServer enabled="true"/>
<GCHeapCount enabled="10"/>
<GCNoAffinitize enabled="true"/>
如果你将 GCNoAffinitize 设置为 true 并指定了一个 GCHeapAffinitizeMask 值，那么 GCHeapAffinitizeMask 将不会被使用。

如果你确实将 GCNoAffinitize 设置为 true，则应确保你的 clr.dll 来自 2018 年 5 月的更新汇总或更新版本（Win2k16 的 KB4103720 和 Win2k12 R2 的 KB4098972），
因为其中有一个性能修复。

需要注意的是，这些配置 仅 影响 GC 的行为——因此，在 40 核机器上将 GCHeapCount 设置为 10 和将进程限制在一个只能使用 10 个 CPU 的作业对象中运行的区别在于，
前者仍然允许用户线程使用所有 40 个 CPU，而后者会将进程中的所有线程限制为只能使用 10 个 CPU。