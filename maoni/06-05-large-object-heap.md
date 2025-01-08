<h1>Large Object Heap</h1>
LOH (Large Object Heap) contains objects that are 85,000 bytes or bigger 
(there’s also some objects that are less than 85,000 bytes that are allocated on the LOH by the runtime itself but usually they are very small and we’ll ignore them for this discussion).

 

The way LOH is implemented changed dramatically from 1.0 to 1.1. In 1.0 we used a malloc type of allocator for allocating large objects; 
in 1.1 and beyond we use the same allocator for both large and small objects: we acquire memory from the OS by heap segments and commit on a segment as needed. 
There’s very little difference between 1.1 and 2.0 for LOH.

 

LOH is not compacted – however if you want a large object to be pinned you should make sure to pin it because whether LOH is compacted or not is an implementation detail 
that could be changed in the future. Free blocks between live large objects are threaded into a free list which will be used to satisfy large object allocation requests.
 Note that this is a real advantage for managed heap – we are able to be very efficient to manage fragmentation because we can coalesce adjacent free objects into a big free block. 
 If a free block is too large we call MEM_RESET on it to tell the OS to not bother backing it up. 
 So far I haven’t heard of any fragmentation problems from production code (of course you write a program that specifically fragments the LOH) – I think one reason is we are doing a good job controlling fragmentation; 
 the other reason is people usually don’t churn LOH too much – one typical pattern I’ve seen people use LOH is to allocate some large arrays that hold on to small objects.

 

When I talked about weak references in Using GC Efficiently – Part 3, 
I mentioned a performance implication of using weak refs which is when you use them to refer to really small objects that are comparable in size. 
Make sure that you are fine with the ratio of the space that weak ref objects take up over the size of objects that they refer to. 
There’s no difference between using a weak reference to refer to a large and a small object aside from the fact that by definition this ratio is naturally smaller for large objects because of their sizes.

 

In this comment, it was asked why “explicitly setting a reference to null would prevent the LOH from growing faster than the collection rate”. 
Actually I think what likely happened was the large objects were held live by something therefore GC was not reclaiming the memory. 
By setting these objects to null just lets GC know that those objects are garbage now and can be reclaimed. You can use either CLRProfiler or the !SOS.gcroot command to verify this.

 

https://devblogs.microsoft.com/dotnet/large-object-heap/