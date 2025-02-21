<h1>My application seems to hang. What do I do? – Part 2</h1>

Last time I talked about the hang scenario where your process is taking 0 CPU and the CPU is taking by other process(es) on the same machine.

 

The next scenario is your process is taking 0 CPU and the CPU is barely used by other processes.

 

As one of the readers correctly pointed out, this is very likely because you have a deadlock. 
Usually debugging deadlocks is relatively straightforward – you look at what the threads are waiting on and figure out which other threads are holding the lock(s). 
And there are plenty of online resources that talk about debugging deadlocks. 
If you use the Windows Debugger package there are built in debugger extension dlls that help you with this like !locks and etc.
 If you are debugging a managed app the SoS debugger extension has commands that will aid you – !SyncBlk shows you managed locks 
 (for CLR 2.0 there’s also !Dumpheap –thinlock for objects locked with ThinLocks instead of SyncBlk’s).

 

Another possibility is your process is not doing any CPU related activities. 
A common activity is IO – for example if the process is heavily paging you will see almost 0 CPU usage but it appears hang 
because the memory it needs is getting loaded from the disk which is really slow.
 A very useful tool that shows you what processes are doing is Process Monitor. Yesterday a program on my machine paused periodically – very annoying. 
 So I used process monitor which showed me that this program periodically checks if I am logged onto my account in another program and since I am not, it would log me on,
  does a little bit stuff then log me off. And the hang was due to waiting on network IO. So to make it happy I logged myself on then the annoying periodic hang disappeared.

 

Now if your process is indeed taking CPU it can also appear to hang – as I mentioned last time this means different things for different people. 
If you have a UI app this can mean the UI is not getting drawn; if you have a server app this can mean your app is not processing requests. 
So you’ll have to define what hang means to you. I will use server app not processing requests as an example. Usually server applications run on dedicated machines. 
So let’s assume that’s the case here – you run a server on a machine and the server could consist of multiple processes. 
You measure the server performance by throughput. One scenario is the CPU usage is high (perhaps even higher than usual) but the throughput is lower than usual.

 

The easiest case, as one of the readers pointed out, is an infinite loop – very easy to debug. 
You break into the debugger a few times and see a thread is taking all the CPU and that thread can not exit some function – so there goes your infinite loop. 
And if your process is pretty much the only process that uses CPU at the time this is super obvious. 
It gets a bit more complicated if you have multiple CPUs and other processes are also using CPUs. 
But still since it’s an infinite loop the nice thing is it will always be executing if you don’t interfere so it’s always available to you to investigate as soon as it happens.

 

It becomes hard if the hang only reproes sporadically and when it reproes it only lasts for a little while. Time to whip out a CPU profiler. 
As another reader pointed out, Process Explorer is a useful tool to get you started.
 It shows you which processes are “active” – meaning it’s using CPUs. Personally I start with collecting appropriate performance counters, 
 partially because pretty much all test teams in the product groups at Microsoft have some sort of automated testing procedure that collects perf counters so requesting them is easy. 
 And because of the low overhead you can collect them for a long period of time so you have a histogram.

 

These are the counters I usually request (comments in []’s):

 

Processor\% Processor Time for _Total and all processors

[This is so I have an idea what kind of CPU usage I am looking at and if there are paticular processors that get used more than the rest]

 

Process\% Processor Time for all processes or less the ones that you already know can not be the problem


 

Thread\% Processor Time for all processes or less the ones that you already know can not be the problem


 

[The above counters will tell you which threads are using the CPU so you know which threads to look at]

 

[Since I usually look at GC related issues I request all counters under .NET CLR Memory]

.NET CLR Memory counters for all managed processes or less the ones that you already know can not be the problem

[If you are looking at other things you should add appropriate counters – for example,  ASP.NET counters for apps that use ASP.NET]

 

[If you know the kind of activities your processes do you can add appropriate counters for them. For me I often request memory related counters like:]

Memory\% Committed Bytes In Use

Memory\Available Bytes

Memory\Pages/sec

Process\Private Bytes for processes I am interested in

…

 

At this point I can look at the results and concentrate on the interesting parts – for example when the CPU is usually high. 
I will have an idea which threads in what processes are consuming the CPU and the aspects of them that interest me (usually GC and other memory activities). 
Then I can request more detailed data on those processes/threads. 
For example I can ask the person to use a sampling profiler so I can see what functions are executing in the part I am interested in
 (along with other info – this depends on what the profiler you are using is capable of).

 

Some people prefer to take memory dumps when the process hangs, sometimes this doesn’t necessarily work (when it works it’s great) because if the hang is related to timing/how threads are scheduled the threads can easily behave differently when you interrupt it in order to take memory dumps so the hang may not repro anymore. If you do have consecutive dumps from one hang then you can use the !runaway command to see which threads have been consuming CPU. One dump is hardly useful for debugging hangs because it only gives you info at one point in time how the process behaves.

https://devblogs.microsoft.com/dotnet/my-application-seems-to-hang-what-do-i-do-part-2/


场景一：进程CPU使用率为0，其他进程占用CPU
上期我们讨论了这种情况，通常是由于低优先级线程被高优先级线程抢占导致的。但还有一种可能：

场景二：进程CPU使用率为0，其他进程CPU使用率也很低
这种情况极有可能是死锁导致的。调试死锁相对直接：

使用Windows调试工具包中的扩展命令（如!locks）

对于托管应用，SoS调试扩展提供了!SyncBlk命令查看托管锁

CLR 2.0还支持!Dumpheap -thinlock检查ThinLocks

另一种可能是进程正在进行IO操作（如大量页面交换），虽然CPU使用率低，但实际在等待磁盘IO。

场景三：进程占用CPU但表现"挂起"
对于不同应用，"挂起"的定义不同：

UI应用：界面无响应

服务器应用：请求处理停滞

以服务器应用为例，假设CPU使用率高于平常但吞吐量下降，可能的原因包括：

无限循环：

调试相对简单：多次中断调试器，观察占用CPU的线程

在多CPU环境下稍复杂，但无限循环的线程始终可被捕获

偶发性挂起：

需要使用CPU分析工具

推荐使用Process Explorer查看活跃进程

建议收集性能计数器，因其开销低且可长期记录

性能计数器收集建议
以下是我常用的计数器（[]内为注释）：

处理器相关

Processor% Processor Time (_Total及所有处理器)
[了解CPU使用概况及是否存在处理器使用不均]

进程/线程相关

Process% Processor Time (所有进程或排除已知无问题的进程)

Thread% Processor Time (所有进程或排除已知无问题的进程)
[定位占用CPU的线程]

.NET相关

.NET CLR Memory (所有托管进程或排除已知无问题的进程)
[如关注GC问题]

内存相关

Memory% Committed Bytes In Use

Memory\Available Bytes

Memory\Pages/sec

Process\Private Bytes (关注的目标进程)
[了解内存使用情况]

深入分析
根据计数器结果，可以：

聚焦CPU使用高峰时段

定位具体进程/线程

使用采样分析器获取函数级执行信息

关于内存转储
虽然内存转储有时很有用，但在以下情况可能失效：

挂起与线程调度时序相关

中断进程获取转储可能改变线程行为

如果确实需要转储，建议获取连续转储并使用!runaway命令查看CPU消耗线程。单一转储对调试挂起问题帮助有限，因为它只反映某一时刻的进程状态。