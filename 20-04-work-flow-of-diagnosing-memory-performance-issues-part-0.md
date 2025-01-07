<h1>Work flow of diagnosing memory performance issues – Part 0</h1>

I wanted to describe what I do to diagnose memory perf issues, or rather the common part of various work flows of doing such diagnostics. Diagnosing performance issues can take many forms because there’s no fixed steps you follow. But I’ll try to break it down into basic blocks that get invoked for a variety of diagnostics.

This part is for beginners so if you’ve been doing memory perf analysis for a while you can safely skip it.

First and foremost, before we talk about the actual diagnostics part, it really pays to know a few high level things that can point you in the right directions.

1) Point-in-time vs histogram

Understanding that memory issues are often not point-in-time is very important. Memory issues usually don’t just suddenly come into the picture – it might take a while for one to accumulate to the point that’s noticeable.

Let’s take a simple example, for a very simple non generational GC that only does blocking GCs that compact, this is still the case. If you are freshly out of a GC, of course the heap is at its smallest point. If you happen to measure at that point, you’ll think “great; my heap is small”. But if you happen to measure right before the next GC, the heap might be much bigger and you will have a different perception. And this is just for a simple GC, imagine what happens when you have a generational GC, or a concurrent GC.

This is why it’s extremely important to understand the GC history to see how GC made the decisions and how the decisions led to the current situation.

Unfortunately many memory tools, or many diagnostics approaches, do not take this into consideration. The way they do memory diagnostics is “let me show you what the heap looks like at the point you happened to ask”. This is often not helpful and sometimes to the point that it’s completely misleading and wasting people’s time to chase a problem that doesn’t exist or have a totally wrong approach how to make progress on the problem. This is not to say tools like these are not helpful at all – they can be helpful when the problem is simple. If you have a dramatic memory leak that’s been going on for a while and you used a tool that shows you the heap at that point (either by taking a process dump and using sos, or by another tool that dumps the heap) it’s probably really obvious what the leak is.

2) Generational GC

By design generational GCs don’t collect the whole heap every time a GC is triggered. They try to do young gen GCs much more often than old gen ones. Old gen GCs are often much more costly. With concurrent old gen GCs, the STW pauses may not be long but GC still needs to spend CPU cycles to do its job.

This also makes looking at the heap much more complicated because if you are fresh out of an old gen GC, especially a compacting one, you obviously have a potentially way smaller heap size than if you were right before that GC is triggered; but if you look at young gen GCs, they could be compacting but the difference is heap size isn’t as much and that’s by design.

3) Compacting vs sweeping

Sweeping is not supposed to change the heap size by much. In our implementation we still give up the space at the end of segments so the total heap size can become a bit smaller but as high level you can think of the total heap size as not changing but free spaces get built up in order to accommodate the allocations from a younger gen (or in gen0/LOH case user allocations).

So if you see 2 gen2 GCs, one is compacting and the other is sweeping, it’s expected if the compacting one comes out with a much smaller heap size and the other one with high fragmentation (by design as that’s the free list we built up).

4) Allocation and survival

While many memory tools report allocations, it’s not just allocations that cost. Sure, allocations can trigger GCs, and that’s definitely a cost but when GC is working, the cost is mostly dominated by survivals. Of course you cannot be in a situation that both your allocation rate and survival rate are very high – you’d just run out of memory very quickly.

5) “Mainline GC scenario” vs “not mainline”

If you had a program that just used the stack and created some objects to use, GC has been optimizing that for years and years. Basically “scan stacks to get the roots and handle the objects from there”. This is the mainline GC scenario that many GC papers assume as the only scenario. Of course as a commercial product that has existed for decades and having to accommodate various customer requests, we have a bunch of other things like GC handles and finalizers. The important thing to understand there is while over the years we also optimized for those, we operate based on assumptions that “there aren’t too many of those” which obviously is not true for everyone. So if you do have many of those, it’s worth looking at if you are diagnosing a memory problem. In other words, if you don’t have any memory problem, you don’t need to care; but if you do (eg, high % time in GC), they are good things to suspect.

All this info is expressed in ETW events or the equivalent on Linux – this is why for years we’ve been investing in them and the tooling for analyzing the traces.

Traces to capture to start with

I often ask for 2 traces to start with. The 1st one is to get the accurate GC timing:

perfview /GCCollectOnly /nogui collect

after you are done, press s in the perfview cmd window to stop it

This should be run long enough to capture enough GC activities, eg, if you know problems occur at times, this should cover time that leads up to when problems happen (not only during problematic time).

If you know how long to run it for you can do (this is used much more often actually) –

perfview /GCCollectOnly /nogui /MaxCollectSec:1800 collect

replace 1800 (half an hour) with however many seconds you need.

This collects the informational level of GC events and just enough OS events to decode the process names. This command is very lightweight so it can be on all the time.

Notice I have the /nogui in all the PerfView commandlines I give out. PerfView does have a UI for event collection that allows you to select the events you want to capture. Personally I never use it (after I used it a couple of times when I first started to use PerfView). Some of it is just because I’m much more a commandline person; the other (important) part is because commandlines allow for much more flexibility and are a lot more automation friendly.

After you collect the trace you can open it in PerfView and look at the GCStats view. Some folks tend to just send it to me after they are done collecting but I would really encourage everyone who needs to do memory diagnostics on a regular basis to learn to read this view ’cause it’s very useful. It gives us a wealth of information, even though the trace so lightweight. And if this doesn’t get us to the root cause, it definitely points at the direction we should take to make more progress. I described some of this view in this blog entry and its sequels that are linked in the entry. So I’m not going to show more pictures here. You could easily open that view and see for yourself.

Examples of the type of issues that can be easily spotted with this view –

Very high “% Time paused for garbage collection”. Unless you are doing some microbenchmarking and specifically testing allocation perf (like many GC benchmarks), you should not see this as higher than a few percent. If you do that’s something to investigate. Below are things that can contribute to this percentage significantly.

Individual GCs with unusually long pauses. Is a 60s GC really long? Yes you bet it is! And this is usually largely not due to GC work. From my experience it’s always due to something interfering with the GC threads.

Excessively induced GCs (high ratio of (# of induced GCs / total # of GCs), especially when the induced GCs are gen2s.

Excessive # of gen2 GCs – gen2 are costly especially when you have a large heap. Even though with BGC, most of its work is done concurrently, it’s still CPU cycles spent so if you have every other GC as gen2, that usually immediately points at a problem. One obvious case is most of them are triggered with the AllocLarge trigger reason. Again, there are cases where this is necessarily not a problem, for example if most of your heap is LOH and you are not running inside a container, which means LOH is not compacted by default, in that case doing gen2s just sweeps the LOH and that’s pretty quick.

Long suspension issues – suspension usually should take much less than 1ms, if it takes 10s of ms that’s a problem; if it takes hundreds of ms, that’s definitely a problem.

Excessive # of pinned handles – in general a few pinned handles are ok but if you see hundreds, that’s a cause for concern, especially if they are during ephemeral GCs; if you see thousands, usually it’s telling you to go investigate.

Those are just things you can see at a glance. If you dig a little deeper there are many more things. And we’ll talk about them next time.

https://devblogs.microsoft.com/dotnet/work-flow-of-diagnosing-memory-performance-issues-part-0/