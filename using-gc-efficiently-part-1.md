Using GC Efficiently – Part 1
So the goal here is to explain the cost of things so you can make good decisions in your managed memory usage – it’s not to explain GC itself – it’s to explain how to use it. I assume most of you are more interested in using a garbage collector than implementing one yourself.  It assumes basic understanding of GC. Jeff Richter wrote 2 excellent MSDN articles on GC that I would recommand if you need some background: 1 and 2.

 

First I’ll focus on Wks GC (so all the numbers are for Wks GC). Then I’ll talk about stuff that’s different for Svr GC and when you should use which (sometimes you don’t necessary have the choice and I’ll explain why you don’t).

 

Generations

 

The reason behind having 3 generations is that we expect for a well tuned app, most objects die in Gen0. For example, in a server app, the allocations associated each request should die after the request is finished. And the in flight allocation requests will make into Gen1 and die there. Essentially Gen1 acts as a buffer between yound object areas and long lived object areas. When you look at the number of collections in perfmon, you want to see a low ratio of Gen2 collections over Gen0 collection. The number of Gen1 collections is relatively unimportant. Collecting Gen1 is not much expensive than collecting Gen0.

 

GC segments

 

First let’s see how GC gets memory from the OS. GC reserves memory in segments. Each segment is 16 MB. When the EE (Execution Engine) starts, we reserve the initial GC segments – one for the small object heap and the other for the LOH (large object heap).

 

The memory is committed and decommitted as needed. When we run out of segments we reserve a new one. In each full collection if a segment is not in use it’ll be deleted.

 

The LOH always lives in its own segments – large objects are treated differently from small objects thus they don’t share segments with small objects.

 

Allocation

 

When you allocate on the GC heap, exactly what does it cost? If we don’t need to do a GC, allocation is 1) moving a pointer forward and 2) clearing the memory for the new object. For finalizable objects there’s an extra step of entering them into a list that GC needs to watch.

 

Notice I said “if we don’t need to do a GC” – this means the allocation cost is proportional to the allocation volume. The less you allocate, the less work GC needs to do. If you need 15 bytes, ask for 15 bytes; don’t round it up to 32 bytes or some other bigger chunk like you used to do when you used malloc. There’s a threshold that when exceeded, a GC will be triggered. You want to trigger that as infrequently as you can.

 

Another property of the GC heap that distinguishes itself from the NT heap is that objects allocated together stay together on the GC heap thus preserves the locality.

 

Each object allocated on the GC heap has an 8-byte overhead (sync block + method table pointer).

 

As I mentioned, large objects are treated differently so for large objects generally you want to allocate them in a different pattern. I will talk about this in the large object section.

 

Collection

 

First thing first – when exactly does a collection happen (in other words, when is a GC triggered)? A GC occurs if one of the following 3 conditions happens:

 

1)      Allocation exceeds the Gen0 threshold;

2)      System.GC.Collect is called;

3)      System is in low memory situation;

1) is your typical case. When you allocate enough, you will trigger a GC. Allocations only happen in Gen0. After each GC, Gen0 is empty. New allocations will fill up Gen0 and the next GC will happen, and so on.
 

You can avoid 2) by not calling GC.Collect yourself – if you are writing an app, usually you should never call it yourself. BCL is basically the only place that should call this (in very limited places). When you call it in your app the problem is when it’s called more often than you predicted (which could easily happen) the performance goes down the drain because GCs are triggered ahead of their schedule, which is adjusted for best performance.

 

3) is affected by other processes on the system so you can’t exactly control it except doing your part of being a good citizen in your processes/components.

 

Let’s talk about what this all means to you. First of all, the GC heap is part of your working set. And it consumes private pages. In the ideal situation, objects that get allocated always die in Gen0 (meaning, they almost get collected by a Gen0 collection and there’s never full collection happening) so your GC heap will never grow beyond the Gen0 size. In reality of course that’s almost never the case. So you really want to keep your GC heap size under control.

 

Secondly, you want to keep the time you spend in GC under control. This means 1) fewers GCs and 2) fewer high generation GCs. Collecting a higher generation is more expensive than collecting a lower generation because collecting a higher generation includes collecting objects that live in that generation and the lower generation(s). Either you allocate very temporary objects that die really quickly (mostly in Gen0, and Gen0 collections are cheap) or some really long lived objects that stay in Gen2. For the latter case, the usual scenario is the objects you allocate up front when the program starts – for example, in an ordering system, you allocate memory for the whole catalog and it only dies when your app is terminated.

 

CLRProfiler is an awesome tool to use to look at your GC heap see what’s in there and what’s holding objects alive.

 

How to organize your data

 

1) Value type vs. Reference type

 

As you know value types are allocated on the stack unlike reference types which are allocated on the GC heap. So people ask how you decide when to use value types and when to use reference types. Well, with performance the answer is usually “It depends” and this one is no different (did you actually expect something else? J). Value types don’t trigger GCs but if your value type is boxed often, the boxing operation is more expensive than creating an instance of a reference type to begin with; and when value types are passed as parameters they need to be copied. But then if you have a small member, making it a reference type incurs a pointer size overhead (plus the overhead for the reference type). We’ve seen some internal code where making it inline (ie, as a value type) improved perf as it decreased working set. So it really depends on your types’ usage pattern.

 

2) Reference rich objects

 

If an object is reference rich, it puts pressure on both allocation and collection. Each embedded object incurs 8 bytes overhead. And since allocation cost is proportional to allocation volume the allocation cost is now higher. When collecting, it also takes more time to build up the object graph.

 

As far as this goes, I would just recommand that normally you should just organize your classes according to their logical design. You don’t want to hold other objects alive when you don’t have to. For example, you don’t want to store references of young objects in old objects if you can avoid it.

 

3) Finalizable objects

 

I will cover more details about finalization in its own section but for now, one of most important things to keep in mind is when a finalizable object gets finalized, all the objects it holds on to need to be alive and this drives the cost of GC higher. So you want to isolate your finalizable objects from other objects as much as you can.

 

4) Object locality

 

When you allocate the children of an object, if the children need to have similar life time as their parent they should be allocated at the same time so they will stay together on the GC heap.

 

Large Objects

 

When you ask for an object that’s 85000 bytes or more it will be allocated on the LOH. LOH segments are never compacted – only swept (using a free list). But this is an implementation detail that you should NOT depend on – if you allocate a large object that you expect to not move, you should make sure to pin it.

 

Large objects are only collected with every full collection so they are  expensive to collect. Sometimes you see after a full collection the Gen2 heap size doesn’t change much. That could mean the collection was triggered for the LOH (you can judge by looking at the decrease in the LOH size reported by perfmon).

 

A good practice with large objects is to allocate one and keep reusing it so you don’t incur more full GCs. If say you want a large object that can hold either 100k or 120k, allocate one that’s 120k and reuse that. Allocating many very temporary large objects is a very bad idea ‘cause you’ll be doing full collections all the time.

 

 

That’s all for Part 1. In the future entries I’ll cover things like pinning, finalization, GCHandles, Svr GC and etc. If you have questions about the topics I covered in this entry or would like more info on them feel free to post them.

