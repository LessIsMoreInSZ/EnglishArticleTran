<h1>GC ETW Events – 4</h1>

So you thought it was over, eh? But wait, there is more! My vacation is not over yet! 😀

In the last blog entry I explained the Suspend MSec and the Pause MSec columns in GCStats and how they are calculated. In this entry I’d like to talk about a few other columns.

GenX MB, GenX Surival Rate % and GenX Frag %

First of all, these are calculated at the end of a GC. The GenX MB column includes the fragmentation in that generation. 
Survival Rate % for GenX is (# of bytes survived from that generation / # of bytes taken up by objects in that generation before the collection). 
Say the generation is 4mb, and the fragmentation is 1mb, so # of bytes taken up by objects is 3mb, if out of these 3mb, 1.5mb survived, 
it means the survival rate % is 50%. And if after the collection this generation is 2mb and the fragmentation is 0.5mb, the frag % is 25%.

You will notice sometimes the GenX Surival Rate % column says NaN. 
It’s because we are not collecting that generation therefore doesn’t make sense to talk about that generation’s survival rate. 
So if we are doing a gen1 GC, gen2’s and LOH’s survival rate will be NaN.

If you are pinning (or the libraries you are using pin on your behalf, eg, IO), usually you would see the frag % for gen0 very big,
 sometimes even 100% (100% doesn’t mean we have only fragmentation in gen0, it’s just because we calculate this column as 
 (fragmentation in gen0 in MB / gen0 size in MB) and they differ a small enough amount that fragmentation in gen0 in MB is the same as gen0 size in MB. 
 We try to leave fragmenation in gen0 as much as we can because it can immediately be used to satisfy allocation requests.

Background GC does not compact so it’s not surprising that after a BGC you see the Frag % in gen2 being fairly big (ie, > 50%). 
We will do a compacting gen2 if we detect that there’s a lot of fragmentation in gen2 that we cannot efficiently use.

We do not compact LOH unless explicitly being asked to. So if the frag % for LOH remains big, 
it means we cannot use a lot of the free spaces on LOH. And if LOH takes up a big percentage of the total heap, it’s worth looking into optimzing the LOH usage.

Promoted MB

This is the # of bytes promoted during a GC. So if we are doing a gen1 GC this is the # of bytes promoted from gen0 and gen1. 
So you could imagine that since gen0 and gen1 usually are a small percentage of the heap the promoted bytes for ephemeral GC usually are 
also a small percentage of what you’d see during your gen2 GCs. As I mentioned before, pause time of a GC is proportional to how much this GC survived, 
in other words, proportional to what you see in the Promoted MB column here (of course background GC is an exception to this as it’s specifically designed to minimize pause time).

A good way to know that something unusual is up is by looking at the pause time of a GC vs its promoted memory. 
If the promoted memory is small yet the pause time is big, it’s almost certain something is up that’s not related to GC work. So what’s considered small?
One way to look at it is to compare it with your other GCs. Let’s say all your full blocking GCs promoted ~1GB and took 1 second except a couple of them that took 10s and still promoted ~1GB,
then you know that something abnormal happened during the couple outliers. To know what happened during those GCs, you can use the following perfview commandline:

perfview /ClrEvents=default-stack-GCHeapSurvivalAndMovement /StopOnGCOverMSec:1000 /DelayAfterTriggerSec:0 /CircularMB:2000 /CollectMultiple:3 /NoGUI /NoNGENRundown collect

(the /ClrEvents excludes a couple of things that would make the GC time skewed. This says to stop the trace if it detects a GC that’s >1s, 
and collect 3 such traces. Since this is not meant to be a generic tutorial of perfview, 
I will not go into details for the commandline args – for the explanation of the rest of the commandline args please search for them in Help\Command Line Help in perfview)

Then what you can do is to look at the GCStats view and get the start and end of those unusually long GCs and open up the CPU stack view and look at the call stacks 
between the start and end timestamp of those GCs. This usually will point to the culprit. However that’s not always the case. 
Sometimes you will see the CPU is simply not being used much (by GC or anything else), in this case you will want to do a ThreadTime trace 
(ie, add /ThreadTime to the above commandline if CPU is not the problem – be aware that turning on /ThreadTime will increase the event volume by a huge amount, 
in which case /CircularMB:2000 might not be enough so you’ll want to increase it to 5000 (or even more if needed) and look at the thread time view to see what’s going on.

Occasionally you might see that the Suspend MSec column shows exceptionally large numbers (again, it’s easy to detect the outliers – most of time you’ll see less than 1ms for this column) 
and since GC hasn’t started at this point it can’t be due to the GC work therefore must be something else. Generally I would only suggest to look at it if you see it happens often enough, 
eg, if you occasionally see it less than, say 30ms I wouldn’t worry about it. But if you see it being longer, and happen often enough that it disturbs your latency on a noticeable basis, 
then you would certainly want to try out the above commandline. In one case I worked on, the customer was seeing this column being in hundreds of ms or even >1s for 10+% GCs. 
We tracked it down to a component they were running on their machines (but were unware of it before that) that constantly changed thread priorities. 
After they removed that component this problem went away.

Finalizable Surv MB and Pinned Obj

These are the same value as what you get with the “Promoted Finalization-Memory from Gen 0” and “# of Pinned Objects” .NET CLR Memory perf counter respectively. 
See GC Performance Counters for an explanation if you need to.

Edited on 03/01/2020 to add the indices to GC ETW Event blog entries

https://devblogs.microsoft.com/dotnet/gc-etw-events-4/