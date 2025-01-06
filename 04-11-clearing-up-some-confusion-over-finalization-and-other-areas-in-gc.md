In the WinDev conference that I just went to, there seems to be some confusion over finalization (such as why it even exists and etc) and other areas. I hope the following will clear up that confusion. If not, let me know.

 

Finalization

 

1)      Why we have finalization

 

Finalization is necessary because you want to make your component robust. You might have no control over your clients of your component in which case if you are seeing an issue (for example, opening a file handle in the exclusive mode before the finalizer closes it) you won’t be able to say “Oh, look this is a bug, go fix it” to whoever wrote the client (heck, the company that wrote the client might already went out of business!).

 

Obviously you want people to call Dispose on your object so the object doesn’t need to be promoted to a higher generation but you are not always in control of people calling Dispose, and very often you are not.

 

2)      What you can do in finalizers

 

Finalizers are only there so you can release your native resources. It’s by design not to do stuff with other managed objects because they would be taken care of by GC. Releasing native resources should be all you do in your objects’ finalize method and that’s a very important thing to do because we are not in a completely managed world – if that were the case, we wouldn’t need finalization at all.

 

3)      When the finalizer thread runs the finalizers

 

Each time we do a GC, we would see if there are any objects whose finalize method needs to run, if so we add it to the freachable queue and set an event to wait up the finalizer thread. Generally if a managed app is doing work, it will trigger GCs so this means whenever you trigger a GC, the finalizer thread, if it’s not already working, will immediately be aware that there are finalizers to run. The finalizer thread doesn’t wake up at “some random point of time in the future”. When it’s waken up is well defined.

 

So what if a finalizer takes a long time to run? Yes it would block the finalizers behind it from running. But remember we said you should only release native resources in your finalize method which should be fast. If it takes days to run a finalizer then there’s a problem in how you implemented the finalize method and you should look into that.

 

A common problem we see is that the finalizer thread would stall if it needs to call into an STA thread to release a COM object because the STA thread is not pumping messages. This is strictly a user error – by design if a thread is STA it should be pumping messages. You can do so via Thread.Join or Application.DoEvents. Sometimes people observe that when they do this sequence:

 

GC.Collect();

GC.WaitForPendingFinalizers();

GC.Collect();

 

The problem would go away – that’s just because WaitForPendingFinalizers happens to pump messages for your thread. This is not recommanded to use to fix this kind of problems because it’s a lot more heavy-weight than calling Thread.Join.

 

Another possible issue is some people claim they observe the finalizers not being run, yet they are not doing anything with COM. If you allocated some finalizable objects, and finish using them, but there’s no GC happening because you are not running out of the Gen0 limit, yes the finalizers won’t get run. But if you are not doing any allocations (that would trigger GCs) it’s unlikely you need the memory anyway so it wouldn’t matter if the finalizers release the native resources or not.

 

When Server GC is turned on

 

As I described in my last blog entry: Using GC Efficiently – Part 2, where I talked about different GC flavors and when Server GC is on, Server GC is not on by default unless you specify it via config or in hosted environment the host sets it for you. You do NOT get it by default on Win2k3 server or anything.

 

Healthy ratio of GCs between different generations

 

When we look at a perfmon log for .NET CLR memory counters, it’s a good indication of healthy GCs when you see a 10:1 ratio between a generation and its higher generation. So if the app does 1000 Gen0 GCs and 100 Gen1 GCs, usually it means it’s very healthy. Of course if the app doesn’t do high generation GCs at all, it’s a great situation to be in! If all the objects you allocate would be gone with a Gen0 collection, you’d be spending a very small amount of time in GC and that’s the ideal situation.

 

Large objects

 

Large objects are collected with every Gen2 GC. If you look at perfmon counters and notice that after a Gen2 GC, the Gen2 heap doesn’t change at all, most likely the large object heap size has decreased ‘cause the Gen2 GC was to get rid of garbage on the large object heap.

 

The definition of large objects is >= 85,000 bytes. Yet sometimes you see less than 85,000 bytes on the large object heap, that is because the runtime itself uses the large object heap as well and it doesn’t necessarily allocate large objects there.

 

Is pinning bad

 

Yes. Pinning could certainly cause very bad fragmentation problems. My suggestion is you should never pin something unless necessary; and when you do pin something, try to either unpin it as quick as possible; or pin buffers that don’t move much – meaning you either pin something on the large object heap where objects don’t move anyway, or you pin buffers that are in Gen2 if you know Gen2 GCs happen very infrequently (this means pinning old buffers hurts less than pinning new allocated buffers ‘cause old buffers tend to be in stable memory that is less likely to move).

 

If you make an async network I/O call, you could be pinning the buffer for quite a while depending on when the async operation finishes. This could possibly create a situation where you have fragmentation scattered around the GC heap because the pinned buffer doesn’t go away fast enough and the space before the pinned buffer can’t be used (too small for the objects being allocated).

 

We’ve done a lot of work (and will keep doing work) to mitigate fragmentation problems caused by pinning.

