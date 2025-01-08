<h1>GC Handles</h1>

A customer asked me about analyzing perf related to GC handles. I feel like aside from pinned handles in general handles are not talked about much so this topic warrants some explanation, especially since this is a user facing feature.

For some background info, GC handles are indeed generational so when we are doing ephemeral GCs we only need to scan handles in the generations we are collecting. However there are complications associated with handles’ ages because of the way handles are organized – we don’t organize by individual handles, we organize them in groups and each group needs to have the same age so if we need to set one handle to be younger we set all of them to be younger so they will all get reported for that younger generation. But this is hard to quantify so I would definitely recommend measuring with the tools we provide.

Handles are exposed in various ways. The way that’s perhaps the most familiar to most folks is via the GCHandle type. Only 4 types are exposed this way: Normal, Pinned, Weak and WeakTrackResurrection. Weak and WeakTrackResurrection types are internally called short and long weak handles. But other types are used via BCL such as the dependent handle type which is used in ConditionalWeakTable (and yes, I am aware that there’s desire to expose this type directly as GCHandle, I’ll touch more on this topic below).

Historically GC handles were considered as an “add-on” thing so we didn’t expect many of these. But as with many other things, they evolve. I’ve been seeing handle usage in general going up by quite a lot (part of it is due to libraries using handles more and more). And currently, we collect handle info in ETW events but not very detailed so I do plan to make the diagnostics info on handles more detailed.

Right now what we offer is:

# of pinned objects promoted in this GC as a column for each GC in the GCStats view in PerfView. This is the number of pinned objects that GC observed in that GC including from pinned handles (async pinned handles used by IO) or pinned by the stack.
In the “Raw Data XML file (for debugging)” link in the GCStats view you can generate a file with more detailed info per GC including handle info; you’ll see something like this:
<GCEvent GCNumber="358" GCGeneration="0" …[many fields omitted] Reason= "AllocSmall"> <HeapStats GenerationSize0="77,228,592"…[many fields omitted] PinnedObjectCount="1" SinkBlockCount="37,946" GCHandleCount="4,552,434"/>

The PinnedObjectCount is also shown here. Then there’s another field called GCHandleCount. As you would guess, GCHandleCount includes not just pinned handles but all handles. Also there’s another significant difference which is this is for all handles whereas PinnedObjectCount is only for what that GC sees so you could have a lot more pinned handles but since only the ones for that generation are reported PinnedObjectCount only includes those (plus stack pinned objects). Another thing worth mentioning is we track GCHandleCount “loosely” in coreclr, as in, we don’t use Interlocked inc/dec for perf reasons. We just do ++/– so this count is a rough figure but gives you a good idea perf wise (eg, in the example shown, that’s a lot of handles and definitely worth some investigation).

Of course if you use the TraceEvent library you can get all the above info on the TraceGC class programmatically.
As you can see this highlights the pinned objects, not other types of handles like the weak GC handles used by WeakReference, dependent handles used by ConditionalWeakTable uses or the SizedRef handles used by asp.net in full framework (no longer exists on coreclr so I’ll not cover them here). There are other types as shown in gc\gcinterface.h but they are used internally by the runtime.

Before we provide more easily consumable info, one fairly easy thing you could do is look at the CPU profiles to see if you should be worried about the usage of these handles – short WeakReferences are scanned with this method:

GCScan::GcShortWeakPtrScan (in case it’s inlined it calls Ref_CheckAlive)

long WeakReferences are scanned with:

GCScan::GcWeakPtrScan (in case it’s inlined it calls Ref_CheckReachable)

Dependent handles are scanned with:

gc_heap::scan_dependent_handles in blocking GCs gc_heap::background_scan_dependent_handles in BGCs

I know this is not ideal but it’s one way to get the info. One thing worth mentioning is these are currently all done during the STW pause for BGCs. And dependent handle scanning is currently done in a not very efficient way which is the main reason why I haven’t exposed this directly as a GCHandle type. I have a design in place to make the perf much better. When we have it implemented we will make this handle type public.

The PerfView commandline to collect events for creating/destroying handles – perfview /nogui /KernelEvents=default /ClrEvents:GC+Stack+GCHandle /clrEventLevel=Informational collect


https://devblogs.microsoft.com/dotnet/gc-handles/