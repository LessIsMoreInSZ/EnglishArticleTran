<h1>No GCs for your allocations?</h1>

Several people mentioned Java’s “no GC” GC proposal and asked if we can add such a thing. So I thought it deserves a blog post.

Short answer – we already have such a feature and that’s called the NoGCRegion. 
The GC.TryStartNoGCRegion API allows you to tell us the amount of allocations you’d like to do and when you stay within it, no GCs will be triggered. 
And you are free to revert back to doing normal GCs when you want to by calling GC.EndNoGCRegion. This is a better model than having a separate GC.

Long answer –

It’s a better model because you have the flexibility to revert back to doing GCs when you need to. 
Would you rather have to spin up a whole new process just so that you can then do work that needs to do GCs (‘cause if you don’t collect, well, 
your memory usage is just going to keep growing and unless you allocate really little you are going to run out pretty soon)?

And yes, there’s currently limitations on how much you can allocate with NoGCRegion. 
You can’t allocate an arbitrary amount on SOH – I limited it to the SOH segment size just because we’ve always had the same size for SOH segments so far. 
There’s no theoretical limit that segments have to be the same size. 
It’s a matter of doing the work to make sure we don’t have places that accidently (artificially) enforce such a limit. There are other design options but I won’t get into the details here.

So this means if you are using Server GC you are able to ask for a lot more memory on SOH because its SOH segment size is a lot larger. 
Currently (and this has been the case for a long time) the default seg size on 64-bit for Server GC is 4GB and we do cut that in half when you have > 4 procs and again if you have > 8 procs. 
So you have > 8 procs it means the SOH segment size is 1GB each.

On Desktop you have ways to make the SOH segment size larger –

use the gcSegmentSize app config or use the ICLRGCManager::SetGCStartupLimits

For LOH as long as we can reserve and commit that much memory this will allow you to allocate ‘cause LOH can already have totally different sized segments.

We choose to make sure we can commit the amount of memory you ask for instead of randomly throwing you OOM because 
when you use such a feature you really should have a good idea how much you want to allocate. 
If you do have a scenario that says “I just want to allocate till I get OOM and I don’t care if I randomly get OOM” please let me know – I’d like to understand what the rationale is.

I did have some bugs when I first checked this into 4.6.x (thanks Matt Warren for reporting). I’ve made fixes in the current release. 
But I wanted to explain how it’s supposed to work so you know what’s a bug and what’s a design limitation.

And we also are still paying the write barrier cost while you are allocating in the NoGCRegion – I kept it that way 
because when you revert back to doing GCs you do need the info that the write barrier recorded if we don’t do anything special. 
However, if we so choose, we can just not have the write barrier for NoGCRegion and when we need to revert back 
to doing GCs we just promote everything that’s still live to gen2 
(would be reasonable to do as at this point you are very likely done with all the temporary stuff and what’s left is justified to be in the old generation) 
and get the write barrier back in the picture again. I didn’t do it this way because write barrier cost generally doesn’t come up – not to say that it can’t ever be a problem. It’s just that we always need to prioritize the work based on how much it’s needed.

https://devblogs.microsoft.com/dotnet/no-gcs-for-your-allocations/