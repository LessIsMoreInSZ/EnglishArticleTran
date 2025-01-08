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