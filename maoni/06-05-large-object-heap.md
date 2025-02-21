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

**LOH（大对象堆）**包含大小在85,000字节或更大的对象（运行时本身也会在LOH上分配一些小于85,000字节的对象，但这些对象通常非常小，我们在此讨论中忽略它们）。

LOH的实现方式从1.0到1.1发生了巨大变化。在1.0中，我们使用类似malloc的分配器来分配大对象；而在1.1及以后的版本中，我们对大对象和小对象使用相同的分配器：我们从操作系统获取堆段的内存，并根据需要提交段。1.1和2.0在LOH的实现上几乎没有区别。

LOH不会被压缩——然而，如果你希望一个大对象被固定（pinned），你应该确保显式地固定它，因为LOH是否被压缩是一个实现细节，未来可能会改变。存活的大对象之间的空闲块会被串联到一个空闲列表中，用于满足大对象的分配请求。需要注意的是，这是托管堆的一个真正优势——我们能够非常高效地管理碎片，因为我们可以将相邻的空闲对象合并成一个大的空闲块。如果一个空闲块太大，我们会对其调用MEM_RESET，告诉操作系统不必为其备份内存。到目前为止，我还没有从生产代码中听到任何关于LOH碎片化的问题（当然，如果你专门编写一个程序来碎片化LOH，那另当别论）——我认为一个原因是我们很好地控制了碎片化；另一个原因是人们通常不会频繁地操作LOH——我看到的典型模式是人们使用LOH来分配一些持有小对象的大数组。

当我在《高效使用GC – 第三部分》中讨论弱引用时，我提到了使用弱引用的一个性能影响，即当你使用弱引用引用非常小的对象时，这些对象的大小与弱引用对象本身的大小相当。确保你能够接受弱引用对象占用的空间与它们引用的对象大小之间的比例。使用弱引用引用大对象和小对象之间没有区别，只是由于大对象的尺寸较大，这个比例自然会更小。

在这条评论中，有人问到为什么“显式地将引用设置为null可以防止LOH的增长速度超过回收速度”。实际上，我认为可能发生的情况是这些大对象被某些东西保持存活，因此GC没有回收这些内存。通过将这些对象设置为null，只是让GC知道这些对象现在是垃圾，可以被回收。你可以使用CLRProfiler或!SOS.gcroot命令来验证这一点。