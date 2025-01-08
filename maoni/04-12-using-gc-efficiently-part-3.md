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