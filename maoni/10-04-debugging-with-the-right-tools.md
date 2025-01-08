<h1>Debugging with the Right Tools</h1>

Wow, it’s been almost a year since I last blogged J We just shipped CLR V4.0. Yay!

 

An internal email thread prompted me to write this blog entry – one very powerful tool I wanted to point out when you need to debug/investigate issues is your debugger
 (if your debugger is also windbg/cdb that is J since that’s what I use and that’s what I will talk about in this article).

 

For those of you who are interested in investigating memory related issues at all, whether it’s because you don’t like your current app’s memory usage or you simply want to improve, 
learning to use the debugger is invaluable. If you haven’t started using windbg/cdb, I would strongly encourage you to – I promise you won’t regret it.

 

I’ve talked about using SoS before. We’ve added some more SoS commands in V4.0 and some of those are related to GC like !AnalyzeOOM, 
!GCWhere and !FindRoots. You can read about them on the MSDN page. But I wanted to talk about some techniques that may not be obvious from reading reference material.

 

What do you want to know about GC when your program is? 2 of the most common things are

 

1) Why are GCs being triggered?

2) Why do GCs take this much time?

 

And the answers are, respectively:

 

1) The difference of the heap before and after a GC

2) The survivors from that GC

 

Let me explain those in detail. Each generation has its own allocation budget which I explained here. 
It’s a value that we set and when exceeded we’d want to trigger a GC. 
If we do a collection and discovered that there’s a lot of memory survived we would set this allocation budget very big which means 
you’d need to allocate more into that generation for us to want to trigger a GC. 
The rationale is that then we’d have a chance to reclaim some sizeable space back next time we do a GC. Otherwise we’d do all this work for a GC and not be able to find much dead memory.

 

So if you do a !dumpheap right before a GC you see what objects you have; then you do a !dumpheap right after that GC you see some of those objects disappeared.
 It’s those objects that triggered this GC. Why? ‘cause if we didn’t have any objects disappearing, we wouldn’t be doing GC (‘cause we’d get no dead space back). 
 When you hear people say “you are doing gen2 GCs because you are churning gen2” this is preciously the explanation for that.

 

As of now, the CLR GC doesn’t compact the LOH. So if you want to look at the LOH you can see exactly which parts of the memory disappeared. 
Here’s a sample output of a heap range before and after a gen2 GC. Before the gen2 GC (I formatted the !dumpheap with the methodtable pointer replaced with their readable names):

 

Address              MethodTable          Size

————————————————–

00000007f7ffe1e0     System.Byte[]        2,080,000

00000007f81f9ee0     System.String        100,000

00000007f8212580     System.Byte[]        140,000

 

After this gen2 GC for the same heap range:

 

Address              MethodTable          Size

————————————————–

00000007f7ffe1e0     Free                 2,320,000

 

So those 3 objects were collected. As the application author those objects may very well ring a bell for you why they were allocated 
so you can go from there see if/how you can do some optimizations.

 

The survivors from the collection are what you see in the output of !dumpheap at the end of that GC.

 

In this MSDN article I talked about how to set a breakpoint at the end of a gen2 GC. To set a bp at the beginning of a gen2 GC simply replace RestartEE with SuspendEE.
 If you want to do this for gen0 GCs, replace 2 with 0. 
 Note that this applies to only non concurrent/background GCs though since concurrent/background GCs would not have the EE suspended for the most part.

 

Often I see people quickly go search for some other tools when they really could whip out the debugger and set a few breakpoints to see what things look like. 
Of course there are many useful tools out there – and they tend to aggregate info for you so you don’t have to do that yourself – but sometimes it’s so convenient to just use the most direct and handy tool which is your debugger.

 

Another classic example is when people ask me for help with debugging memory leaks and the easiest and quickest solution is to set a conditional bp on kernel32!VirtualAlloc for
native memory leaks. I’ve seen this over and over again and this is how the conversation would usually go:

 

someone, “Maoni, I have a managed app that’s leaking memory. Can you help me with it?”

me, “Is it a managed memory leak?”

someone, “I don’t know.”

me, “How big is your managed heap? Can you do the SoS !dumpheap command?”

someone, “it’s only X MBs” [but his whole process uses much more than that]

me, “Does it grow? If you run your app for a while does the managed heap size change?”

someone, “No” (or “Not much, not nearly as much as the whole memory usage increases”)

me, “Can you take a look at your virtual memory address space by doing !address?” [!address gives you info on each virutal memory chunk. 
It’s also a windbg/cdb debugger extension but it’s in ext.dll  which comes with the debugger package.]

someone, “It looks like there’re a lot of chunks of size S…”

me, “Can you set a conditional bp on kernel32!VirtualAlloc when the allocate size is S?” [Which means: bp kernel32!virtualalloc “.if (@edx==S) {kp} .else {g}”]

[some time elapses]

someone, “It looks like they all have the same callstack!!!”

 

At this point it’s clear what causes their memory leak.

 

If you need to see continuous output of some commands without disturbing the process too much you can always log it to a file (.logopen c:\mydebugsession.txt) and analysis the log 
after the process runs for a while.

 

As I mentioned there are tools that will aggregate this info for you (eg, the DebugDiag tool you can find on microsoft.com 
which shows you the aggregated callstack from all the VirtualAlloc calls) but if you’ve never used those tools it may take a while to set them up and learn how to use them. 
The debugger is something you have right there and use all the time so it’s something you can fairly conveniently solve with a debugger I tend to use that.

https://devblogs.microsoft.com/dotnet/debugging-with-the-right-tools/