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

