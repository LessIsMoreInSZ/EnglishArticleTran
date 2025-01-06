<h1>So, what’s new in the CLR 4.0 GC?</h1>

PDC 2008 happened not long ago so I get to write another “what’s new in GC” blog entry. For quite a while now I’ve been working on a new concurrent GC that replaces the existing one. 
And this new concurrent GC is called “background GC”.

First of all let me apologize for having not written anything for so long. It’s been quite busy working on the new GC and other things.

Let me refresh your memory on concurrent GC. Concurrent GC has existed since CLR V1.0. For a blocking GC, ie, a non concurrent GC we always suspend managed threads, 
do the GC work then resume managed threads. Concurrent GC, on the other hand, runs concurrently with the managed threads to the following extend:

§  It allows you to allocate while a concurrent GC is in progress.

 

However you can only allocate so much – for small objects you can allocate at most up to end of the ephemeral segment. Remember if we don’t do an ephemeral GC, 
the total space occupied by ephemeral generations can be as big as a full segment allows so as soon as you reached the end of the segment 
you will need to wait for the concurrent GC to finish so managed threads that need to make small object allocations are suspended.

 

§  It still needs to stop managed threads a couple of times during a concurrent GC.

 

During a concurrent GC we need to suspend managed threads twice to do some phases of the GC. These phases could possibly take a while to finish.

 

We only do concurrent GCs for full GCs. A full GC can be either a concurrent GC or a blocking GC. Ephemeral GCs (ie, gen0 or gen1 GCs) are always blocking.

 

Concurrent GC is only available for workstation GC. In server GC we always do blocking GCs for any GCs.

Concurrent GC is done on a dedicated GC thread. This thread times out if no concurrent GC has happened for a while and gets recreated next time we need to do concurrent GC.

When the program activity (including making allocations and modifying references) is not really high and the heap is not very large concurrent GC works 
well – the latency caused by the GC is reasonable. But as people start writing larger applications with larger heaps that handle more stressful situations, the latency can be unacceptable.

Background GC is an evolution to concurrent GC. The significance of background GC is we can do ephemeral GCs while a background GC is in progress if needed. 
As with concurrent GC, background GC is also only applicable to full GCs and ephemeral GCs are always done as blocking GCs, 
and a background GC is also done on its dediated GC thread. The ephemeral GCs done while a background GC is in progress are called foreground GCs.

So when a background GC is in progress and you’ve allocated enough in gen0, 
we will trigger a gen0 GC (which may stay as a gen0 GC or get elevated as a gen1 GC depending on GC’s internal tuning). 
The background GC thread will check at frequent safe points (ie, when we can allow a foreground GC to happen) and see if there’s a request for a foreground GC. 
If so it will suspend itself and a foreground GC can happen. After this foreground GC is finished, the background GC thread and the user threads can resume their work.

Not only does this allow us to get rid of dead objects in young generations, 
it also lifts the restriction of having to stay in the ephemeral segment – if we need to expand the heap while a background GC is going on, we can do so in a gen1 GC.

We also made some performance improvement in background GC which does better at doing more things concurrently so the time we need to suspend managed threads is also shorter.

We are not offering background GC for server GC in V4.0. It’s under consideration – we recognize how important 
it is for server applications (which usually have much larger heaps than client apps) to benefit from smaller latency but the work did not fit in our V4.0 timeframe. 
For now for server applications, I would recommend you to look at the full GC notification feature we added in .NET 3.5 SP1. 
It’s explained here: http://msdn.microsoft.com/en-us/library/cc713687.aspx. Basically you register to get notified when a full GC is approaching and when it’s finished. 
This allows you to do software load balancing between different server instances – when a full GC is about to happen in one of the server instances, 
you can redirect new requests to other instances.

https://devblogs.microsoft.com/dotnet/so-whats-new-in-the-clr-4-0-gc/