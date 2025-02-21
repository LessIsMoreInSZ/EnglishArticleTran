<h1>My application seems to hang. What do I do? – Part 1</h1>

Defining “hang” is a good place to start.

When people say “hang” they could mean all sorts of things. When I say “hang” I mean the process is not making progress – the threads in the process are either blocked (eg. deadlocked, or not scheduled because of threads from other processes) or executing code (madly) but not doing useful work (eg. infinite loop, or busy spinning for a long time without doing useful work). The former uses no CPU while the later using 100% CPU. When a UI developer says “hang” he could mean “the UI is not getting drawn” so essentially they mean the UI threads are not working – other threads in their process could be doing lots of work but since the UI is not getting updated it appears “hang”. So clarifying what you mean when you say “hang”, which requires you to look at your process and its threads, is the first step.

 

If you start Task Manager (taskmgr.exe) it shows you how much CPU each process is using currently. If you don’t see a CPU column you can add it by clicking View\Select Columns and check the “CPU Usage” checkbox.

 

Note that if you have multiple CPUs, the CPU usage is at most 100. Let’s say you have 4 CPUs and your process has one thread that’s running and taking all the CPU it can you will see the CPU column for your process 25 – since your process can only use one CPU (at most to its full) at any given time.

 

The CPU usage for a process is calculated as the CPU usage used by all the threads that belong to the process. Threads are what get to run on the CPUs. They get scheduled by the OS scheduler which decides when to run what thread on which processor. I won’t cover the details here – the Windows Internals book by Russinovich and Solomon covers it.

 

If you see your process is taking 0 CPU, that would explain why it’s hung (for the period of time when the CPU keeps being 0) – no threads are getting to run in your process! The next thing to look at is the CPU usage of other processes. If you see one or multiple other processes that take up all the CPU that means the threads in your process simply don’t get a chance to run – this is because the threads in those other processes are of higher priorities (or temporarily of higher priorities due to priority boosting) than the threads in your process. The possible causes are:

 

1) there are threads that are marked as low priority which acquired locks that other threads in your process need in order to run. And the low priority threads are preempted by other (normal or high) prority threads from those other processes. This happens when people mistakenly use low priority threads to do “unimportant work” or “work that doesn’t need to be done in a timely fashion” without realizing that it’s nearly impossible to avoid taking locks on those threads. I’ve heard of many people say “but I am not taking a lock on my low priority threads” which is not a valid argument because the APIs you call or the OS services you use can take locks in order to run your code – allocating on native NT heap can take locks; even triggering a page fault can take locks (which is not something an application developer can control in his code).

 

2) the threads in your process are of normal priority but those other processes have high priority threads – this should be relatively easy to diagnose (and unless some process is simply bad citizens this rarely happens) – you can take a look at what those processes are doing (again looking at their threads’ callstacks is a good place to start).

 

That’s all for today. Next time I will talk about other hang scenarios and techniques to debug them.

https://devblogs.microsoft.com/dotnet/my-application-seems-to-hang-what-do-i-do-part-1/

当人们说“挂起”时，他们可能指的是各种各样的情况。当我说“挂起”时，我指的是进程没有进展——进程中的线程要么被阻塞（例如，死锁，或者由于其他进程的线程而没有被调度），要么在执行代码（疯狂地）但没有做有用的工作（例如，无限循环，或者长时间忙等待而没有做有用的工作）。前者不使用CPU，而后者使用100%的CPU。当UI开发人员说“挂起”时，他们可能指的是“UI没有绘制”，因此本质上他们指的是UI线程没有工作——他们的进程中的其他线程可能在做大量工作，但由于UI没有更新，看起来像是“挂起”。因此，澄清你所说的“挂起”是什么意思，这需要你查看你的进程及其线程，这是第一步。

使用任务管理器查看CPU使用情况
如果你打开任务管理器（taskmgr.exe），它会显示每个进程当前使用的CPU量。如果你没有看到CPU列，可以通过点击 查看\选择列 并勾选“CPU使用率”来添加它。

需要注意的是，如果你有多个CPU，CPU使用率最多为100。假设你有4个CPU，而你的进程有一个线程正在运行并尽可能占用CPU，你会在进程的CPU列中看到25——因为你的进程在任何给定时间最多只能使用一个CPU（满负荷）。

进程的CPU使用率是计算为该进程中所有线程使用的CPU使用率的总和。线程是在CPU上运行的单位，它们由操作系统调度器决定何时在哪个处理器上运行。我不会在这里详细介绍——Russinovich和Solomon的《Windows Internals》一书对此有详细说明。

如果你的进程占用0% CPU
如果你看到你的进程占用0% CPU，这可以解释为什么它挂起了（在CPU持续为0的时间段内）——你的进程中没有线程在运行！接下来要查看的是其他进程的CPU使用率。如果你看到一个或多个其他进程占用了所有CPU，这意味着你的进程中的线程根本没有机会运行——这是因为那些其他进程中的线程优先级更高（或者由于优先级提升而暂时更高）比你的进程中的线程。可能的原因包括：

低优先级线程持有锁：

有一些被标记为低优先级的线程获取了锁，而你的进程中的其他线程需要这些锁才能运行。这些低优先级线程被其他进程中的正常或高优先级线程抢占。这种情况发生在人们错误地使用低优先级线程来做“不重要的工作”或“不需要及时完成的工作”时，而没有意识到这些线程几乎不可避免地会持有锁。我听过很多人说“但我的低优先级线程没有持有锁”，这是不成立的，因为你调用的API或操作系统服务可能会为了运行你的代码而持有锁——在本地NT堆上分配内存可能会持有锁；甚至触发页面错误也可能会持有锁（这是应用程序开发人员无法在代码中控制的）。

你的进程中的线程是正常优先级，但其他进程有高优先级线程：

这种情况应该相对容易诊断（除非某些进程是“坏公民”，否则这种情况很少发生）——你可以查看这些进程在做什么（查看它们的线程调用栈是一个很好的起点）。

总结
今天就到这里。下次我将讨论其他挂起场景以及调试它们的技术。