<h1>gen2-free-list-changes-in-clr-4-6-gc</h1>

I wanted to mention this because I started seeing posts about it. In 4.6 we improved the way we use the gen2 free list to promote gen1 survivors into. 
Unfortunately there was a perf bug that I didn’t notice till it was fairly late so it wasn’t approved to be checked into 4.6 at the time. Now that I am seeing more people hitting it, 
I have checked the fix into the next hotfix rollup for 4.6 which you will be able to download when it’s released in the near future (I will update the exact location when it’s available).

The symptom for this bug is that you are seeing GC taking a lot longer than 4.5 – if you collect ETW events this will show that gen1 GCs are taking a lot longer.

Note that this is NOT due to any deficiency in background GC (I’ve seen some posts saying that background GC is at fault – this is not the case), 
it’s simply because of the bug in the free list management. 
You may see that the bug manifests itself a lot less when you disable Concurrent GC but that’s only because background GC does not compact, 
by design, so it uses the free list a lot more. Some people also observed turning on Server GC makes the problem “go away” but again, 
that’s just the way it manifests itself in Server GC (by turning on Server GC you just made GC throughput a lot higher 
so even though there’s more work to do the total GC pause is lower as work is now done by multiple GC threads).

What makes this bug show up is when the objects that die are small ones that scattered around the heap. 
This is not exactly very easy to debug so unless you want to put in a bunch of effort (to debug and potentially to change your code) I would recommend to wait for the hotfix rollup.

https://devblogs.microsoft.com/dotnet/gen2-free-list-changes-in-clr-4-6-gc/