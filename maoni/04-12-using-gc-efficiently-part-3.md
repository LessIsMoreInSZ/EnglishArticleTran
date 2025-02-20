<h1>using-gc-efficiently-part-3</h1>
In this article I’ll talk about pinning and weak references – stuff related to GC handles.

 

(I was planning on talking about finalization in this part of the “Using GC Efficiently” series but since I already covered it in pretty much detail in one of my previous blog entries I won’t repeat it here. Feel free to ask if you have questions not answered by that entry.)

 

Pinning

 

In a way pinning is like finalization – both exist because we have to deal with native code.

 

When do objects get pinned? In 3 situations:

 

1)      Using of GCHandle of type GCHandleType.Pinned;

2)      Using the “fixed” keyword in C# (and other languages might have similar things, I don’t know);

3)      During Interop, certain types of arguments get pinned by Interop (for example, to marshal LPWSTR as a String object, Interop pins the buffer for the duration of the call).

 

For small object heap,  pinning is the only user scenario that could create fragmentation (by “user scenario” I mean not by the runtime/GC itself but rather from the user code).

 

For large object heap, pinning is no-op as large object heap is never compacted. But you should always pin it if you want it to be pinned. As I mentioned in my previous entry, LOH always being swept is an implementation detail.

 

Creating fragmentation is never a good thing. It makes GC work a lot harder – instead of simply “squeezing” live objects together now it has to keep records of which live objects are pinned and try to fit objects in free spaces between pinned objects. With each release we are doing more work in mitigating issues created by fragmentated heaps.

 

So how do you determine how much fragmentation you have in your application? You can use the !dumpheap command in the SOS debugger extension and look for Free objects – “!dumpheap –type Free –stat” will give you the summary of all Free objects. Generally if there’s 10% or less fragmentation in the heap I wouldn’t worry about it. So when you have a big heap don’t panic if you see the absolute number of bytes of Free objects being high but is less than 10%. Looking at the objects after the Free objects could give you a clue who is pinning.

 

When you do need to pin, here are some things to keep in mind:

 

1)      Pinning for a short time is cheap.

 

How short is “a short time”? Well, if there’s no GC happening, pinning simply sets a bit in the  object header and unpinning simply clears it. But when GC happens, we have to make sure to  not move the pinned objects. So “pinning for a short time” means when GC doesn’t notice this object was pinned. This in turn means when you pinned some objects, and before you unpin it there’re not much, if any, allocations going on.

 

2)      Pinning an older object is not as harmful as pinning a young object.

 

By “an older object” I mean an object that has had the chance to migrate to Gen2 and being compacted into a relatively stable location.

 

3)      Creating pinned buffers that stay together instead of scattered around. This way you create fewer holes.

 

A couple of examples of good techniques:

 

1)      Allocate a pinned buffer in LOH and give out a chuck at a time.

 

The downside is “chucks” are not objects and there are very few APIs that accept non objects.

 

2)      Allocate a pool of small object buffers and hand them out when needed.

 

For example, I have a pool of buffers, and method M takes a byte array which needs to be pinned. If the buffer is already in Gen2 it’s ok to pin it. The idea is hopefully your method doesn’t need to use the buffer for long so the buffer will be free in a Gen0 or Gen1 collection. When the buffer is not a Gen2 buffer you get a buffer from your buffer pool to use – all buffers in the buffer pool by now are most likely all in Gen2 anyway:

 

void M(byte[] b)

{

    if (GC.GetGeneration(b) == GC.MaxGeneration)

    {

        RealM(b);

    }

 

    // GetBuffer will allocate one if no buffers

    // are available in the buffer pool.

    byte[] TempBuffer = BufferPool.GetBuffer();

    RealM(TempBuffer);

    CopyBackToUserBuffer(TempBuffer, b);

    BufferPool.Release(TempBuffer);

}

 

Weak References

 

How are weak references implemented?

 

A weak reference has a managed part and a native part. The managed part is the WeakReference class itself. In its constructor we ask to create a GC handle which is the native part of it – it inserts an entry in that AppDomain’s handle table (note that GCHandle’s are all created this way – they are just inserted as their respective types). The object that the weak reference refers to will die when there are no strong references point to it. Since the weak reference is a managed object itself, it will be freed like any other managed objects.

 

This means if you have a very small object, let’s say one DWORD field, the object will be 12 bytes (size of a mininal object). If you have a WeakReference which has an IntPtr and a bool field, and the GC handle which is a pointer size, you are paying more than the object size to refer to the object with a weak reference. So obviously you don’t want to get into a situation where you are creating many weak references to refer to a small object.

 

What are the uses of weak references?

 

1)      Keeping an object alive temporarily or clean up when an object gets collected.

 

Why would you use weak references to watch objects to clean up, instead of using a finalizer? The advantage is that the object being watched isn’t promoted like it would be if it had a finalizer; the disadvantage is that it’s more expensive in terms of memory consumption and the clean up happens only when the user code checks on the object the weak reference points to being null.

 

Option A):

 

class A

{

    WeakReference _target;   

 

    MyObject Target

    {

        set

        {

            _target = new WeakReference(value);

        }

 

        get

        {

            Object o = _target.Target;

            if (o != null)

            {

                return o;

            }

            else

            {

                // my target has been GC’d – clean up

                Cleanup();   

                return null;

            }

        }

    }

 

    void M()

    {

        // target needs to be alive throughout this method.

        MyObject target = Target;

 

        if (target == null)

            // target has been GC’d, don’t bother

            return;

        else

        {

            // always need target to be alive.

            DoSomeWork();

        }

 

        GC.KeepAlive(target);

    }

}

 

Option B):

 

class A

{

    WeakReference _target;

    MyObject ShortTemp;

 

    MyObject Target

    {

        set

        {

            _target = new WeakReference(value);

        }

 

        get

        {

            Object o = _target.Target;

            if (o != null)

            {

                return o;

            }

            else

            {

                // my target has been GC’d – clean up

                Cleanup();       

                return null;

            }

        }

    }

 

    void M()

    {

        // target needs to be alive throughout this method.

        MyObject target = Target;

        ShortTemp = target;

 

        if (target == null)

        {

            // target has been GC’d, don’t bother

            return;

        }

        else

        {

            // could assert that ShortTemp is not null.

            DoSomeWork();

        }

 

        ShortTemp = null;

    }

}

 

2)      Maintaining a cache.

 

You can have an array of weak references:

 

WeakReferencesToObjects WeakRefs[];

 

Where each item in the array references an object by a weak reference. Periodically we could go through this array and see which objects are dead and release the weak references for those objects.

 

If we always get rid of the weak references when the objects they refer to are dead, the cache will be invalidated on each GC.

 

If that’s not sufficient for you, you can use a 2-level caching mechanism:

 

·          Maintain a strong reference for the cached items for x amount of time;

 

·          After x amount of time is up, convert the strong references to weak references. Weak references will be considered to be kicked out of the cache before strong references are considered.

 

You could have difference policies for the cache such as based on the times the cached items are queried – the ones that are queried less often are maintained by or converted to weak references; or based on the number of items in the cache, maintain weak references to the overflown items when the cache is bigger than a certain size. It all depends on your cache usage. Tuning caches is a whole other topic on its own – perhaps some day I will write about it.

https://devblogs.microsoft.com/dotnet/using-gc-efficiently-part-3/

在这篇文章中，我将讨论固定（pinning）和弱引用（weak references）——与GC句柄相关的内容。

（我原本计划在这部分“高效使用GC”系列中讨论终结，但由于我在之前的博客文章中已经非常详细地覆盖了这个话题，所以这里不再重复。如果你有那篇文章没有回答的问题，欢迎提问。）

固定（Pinning）

在某种程度上，固定就像终结一样——两者存在是因为我们必须处理本机代码。

对象在以下三种情况下会被固定：

1.使用GCHandle类型为GCHandleType.Pinned的GCHandle；
2.在C#中使用“fixed”关键字（其他语言可能也有类似的东西，我不太清楚）；
3.在互操作期间，某些类型的参数会被互操作固定（例如，将LPWSTR作为String对象封送时，互操作会固定缓冲区，直到调用结束）。
对于小对象堆，固定是唯一可能导致碎片化的用户场景（通过“用户场景”我指的是不是由运行时/GC本身，而是由用户代码产生的）。

对于大对象堆，固定是无效操作，因为大对象堆永远不会被压缩。但是，如果你想要固定它，你应该总是去固定它。正如我在之前的文章中提到的，LOH始终被清除是一个实现细节。

创建碎片化绝不是一件好事。它使GC工作变得更加困难——现在它不仅要简单地“挤压”存活对象在一起，还必须记录哪些存活对象被固定，并尝试在固定对象之间的空闲空间中放置对象。
在每次发布中，我们都在做更多的工作来减轻由碎片化堆引起的问题。

那么，你如何确定你的应用程序中的堆碎片化程度呢？你可以使用SOS调试器扩展中的!dumpheap命令，并查找Free对象——“!dumpheap –type Free –stat”将给出所有Free对象的摘要。
通常，如果堆中的碎片化程度在10%或以下，我不会太担心。所以当你有一个大堆时，如果看到Free对象的字节数量很大，但不超过10%，不要惊慌。查看Free对象之后的对象可能会给你一些关于谁在固定的线索。

当你确实需要固定时，以下是一些需要记住的事情：

1.短时间固定是廉价的。
多短算是“短时间”？嗯，如果没有发生GC，固定只会在对象头中设置一个位，解除固定则清除它。但是当GC发生时，我们必须确保不移动被固定的对象。所以“短时间固定”意味着在GC没有注意到这个对象被固定的情况下。
这意味着当你固定了一些对象，在你解除固定之前，如果没有太多的分配发生，那么这是可以的。

2.固定一个较老的对象不如固定一个年轻的对象有害。
通过“一个较老的对象”，我指的是有机会迁移到Gen2并且被压缩到一个相对稳定位置的对象。

3.创建保持在一起的固定缓冲区，而不是分散各处。这样你创造的空洞会更少。
以下是一些好的技术示例：

1.在LOH中分配一个固定缓冲区，并一次给出一个块。
缺点是“块”不是对象，很少有API接受非对象。

2.分配一组小型对象缓冲区，并在需要时提供。
例如，我有一组缓冲区，方法M需要一个需要固定的字节数组。如果缓冲区已经在Gen2中，那么固定它是可以的。这个想法是希望你的方法不需要长时间使用缓冲区，这样缓冲区就会在Gen0或Gen1收集时被释放。
当缓冲区不是Gen2缓冲区时，你从缓冲池中获取一个缓冲区来使用——到那时，缓冲池中的所有缓冲区很可能都在Gen2中：

void M(byte[] b) { if (GC.GetGeneration(b) == GC.MaxGeneration) { RealM(b); }

// 如果缓冲池中没有可用的缓冲区，GetBuffer将分配一个。
byte[] TempBuffer = BufferPool.GetBuffer();
RealM(TempBuffer);
CopyBackToUserBuffer(TempBuffer, b);
BufferPool.Release(TempBuffer);
}

弱引用（Weak References）

弱引用是如何实现的？

弱引用有一个托管部分和一个本机部分。托管部分是WeakReference类本身。在其构造函数中，我们请求创建一个GC句柄，这是它的本机部分——它在该AppDomain的句柄表中插入一个条目（
请注意，GCHandle都是以这种方式创建的——它们只是以它们各自的类型插入）。弱引用所引用的对象将在没有强引用指向它时死亡。由于弱引用本身就是一个托管对象，它将像其他托管对象一样被释放。

这意味着如果你有一个非常小的对象，比如说一个DWORD字段，该对象将是12字节（最小对象的大小）。如果你有一个WeakReference，它有一个IntPtr和一个bool字段，以及一个指针大小的GC句柄，
你为了用弱引用引用对象而付出的代价超过了对象的大小。所以很明显，你不想陷入创建许多弱引用来引用一个小对象的情况。

弱引用的用途是什么？

1.暂时保持对象
class A

{

    WeakReference _target;   

 

    MyObject Target

    {

        set

        {

            _target = new WeakReference(value);

        }

 

        get

        {

            Object o = _target.Target;

            if (o != null)

            {

                return o;

            }

            else

            {

                // my target has been GC’d – clean up

                Cleanup();   

                return null;

            }

        }

    }

 

    void M()

    {

        // target needs to be alive throughout this method.

        MyObject target = Target;

 

        if (target == null)

            // target has been GC’d, don’t bother

            return;

        else

        {

            // always need target to be alive.

            DoSomeWork();

        }

 

        GC.KeepAlive(target);

    }

}

 

Option B):

 

class A

{

    WeakReference _target;

    MyObject ShortTemp;

 

    MyObject Target

    {

        set

        {

            _target = new WeakReference(value);

        }

 

        get

        {

            Object o = _target.Target;

            if (o != null)

            {

                return o;

            }

            else

            {

                // my target has been GC’d – clean up

                Cleanup();       

                return null;

            }

        }

    }

 

    void M()

    {

        // target needs to be alive throughout this method.

        MyObject target = Target;

        ShortTemp = target;

 

        if (target == null)

        {

            // target has been GC’d, don’t bother

            return;

        }

        else

        {

            // could assert that ShortTemp is not null.

            DoSomeWork();

        }

 

        ShortTemp = null;

    }

}

2.维护缓存。

你可以有一个弱引用数组：

WeakReferencesToObjects WeakRefs[];

在这个数组中，每个项都通过弱引用来引用一个对象。我们可以定期遍历这个数组，看看哪些对象已经死亡，并释放那些对象的弱引用。

如果我们总是在它们引用的对象死亡时清除弱引用，那么每次进行垃圾回收时，缓存都会被无效化。

如果这对你来说还不够，你可以使用两级缓存机制：

· 维持一个强引用，用于缓存项目，持续x段时间；

· 当x时间到了之后，将强引用转换为弱引用。弱引用会在考虑清除强引用之前被视为被踢出缓存。

你可以为缓存设置不同的策略，比如基于缓存项目的查询次数——查询次数较少的项目可以通过弱引用来维护或转换；或者基于缓存中的项目数量，当缓存大小超过一定大小时，维护超出部分的弱引用。
这一切都取决于你的缓存使用情况。调整缓存是一个单独的完整话题——也许有一天我会写关于它的文章。