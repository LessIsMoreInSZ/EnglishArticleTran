<h1>Provisional Mode</h1>

A coworker asked me what this “PMFullGC” trigger reason he’s seeing in GCStats means. I thought it’d be useful to share the info here.

PM stands for Provisional Mode which means after a GC starts, it can change its mind about the kind of GC it’s doing. But what does that mean exactly?

So normally when we start a GC, the first things we do are-

determine which generation we collet
if it’s a gen2 we decide if it should be done a background or blocking GC
And after that the collection work will start and go with the decision we made.

When provisional mode is on, while we are already in the middle of a GC, we can say “hmm, it looks like collecting this generation was not a good idea, let’s go with a different generation instead”. This is to handle the cases where our prediction of how the heap would behave is very difficult to get right (or would be expensive to get it more right when we were predicting). Currently there’s only one situation that would trigger this provisional mode. In the future we might add more.

The one situation that triggers the provisional mode is when we detect high memory/high gen2 frag situation during a full blocking GC. And is turned off when we detect neither situation is true in a full blocking GC.

Before I added this provisional mode, the tuning heuristic for this particular situation, ie, high memory load and high fragmentation in gen2, would cause us to do a lot of full compacting GCs because we would think it’d be productive – after all there’s a lot of free space in gen2 and doing a compacting GC would compact it away and get the heap size down which is what we really want when the memory load is high. But if the fragmentation is due to pinning and the pins keep not going away, we could compact but the heap is not shrinking because the pins are still there. And they are in gen2 so it’s harder to use.

We can’t easily predict when the pins will go away. We do know about the pinned handles but we also need to know how much free space would result inbetween them. And it’s hard to know stack pinning unless you actually go walk the stacks. We can operate on the previous knowledge and perhaps stop doing compacting gen2’s for a while and try it again after some number of GCs.

The way I chose to handle this was when we detect this high memory/high fragmentation situation when we do a full compacting GC, we put GC in this provisional mode. And next time when the normal tuning says we are supposed to do a full blocking GC again, we would reduce it to a gen1 compacting GC. We keep doing this and compact as many gen1 survivors into the gen2 free list (so it doesn’t actually increase gen2 size) till a gen1 GC where we can’t fit gen1 survivors into gen2 free list anymore. At this point we change our mind and say we actually want to do a full compacting GC instead. So these GCs are said to be “provisioned” and the trigger reason for this full compacting GC is what you see in GCStats – PMFullGC.

This way I didn’t need to change much of the existing tuning logic. And when we change our mind during the middle of a gen1 GC, we just do a sweeping gen1 so it’ll quickly finish and immediately trigger a full compacting GC right after. We could actually discard what we’ve done so far for this gen1 and “restart” it as a full compacting GC but it doesn’t gain much and would require a much bigger code churn. Since we are discovering this right before we need to decide whether this should be a compacting or sweeping gen1 it’s trivial to just make it a sweeping gen1.

And when we trigger this full compacting GC, if we then detect we are out of the high memory load/high fragmentation situation, most likely because the pins were gone so we were able to compact and reduce the memory load, we could take GC out of the provisional mode.

Of course we hope that normally you don’t have a bunch of pins in gen2 that keep not going away which was why we had our previous tuning logic. And that logic worked well if there wasn’t high fragmentation created by pinning. But we did want to handle this so we could accommodate more customer scenarios. There was a team that hit exactly this situation and before the provisional mode was added they saw many full compacting GCs which made % pause time in GC very high. With this provisional mode they saw the % pause time in GC reduced dramatically because they were doing much fewer full compacting GCs since most of them got converted to gen1s, and still maintained the same heap size.

I also explained provisional mode during my meetup talk in Prague last year. It was made available in 4.7.x on .NET and 3.0 on .NET Core.

https://devblogs.microsoft.com/dotnet/provisional-mode/