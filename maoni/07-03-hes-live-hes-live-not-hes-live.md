<h1>hes-live-hes-live-not-hes-live</h1>

I was making some code changes today and thought this was interesting to share. 
As you know, the WeakReference class has a getter and a setter method to get and set the Target which is what the weakref points to. 
See Using GC Efficiently – Part 3 for more details on WeakReference.

 

Note that the code below is only for illustration purposes – it is not necessarily what’s in the production code.

 

So let’s say the code used to look like this in the WeakReference class:

 

internal IntPtr m_handle;

 

public Object Target

{

    get

    {

        IntPtr h = m_handle;

        if (IntPtr.Zero == h)

            return false;

 

        Object o = GCHandle.InternalGet(h);

        h = Thread.VolatileRead(ref m_handle);

        GC.KeepAlive (this);

        return (h == IntPtr.Zero) ? null : o;

    }

 

    …

}

 

m_handle is the weak GCHandle that we create to implemente the weakref funtionality.
 It’s a weak handle that points to the object that you want your weakref object to point to.

 

The problem is Thread.VolatileRead kind of a heavy weight thing – not very performant 
(there was a reason why we used this API in the first place…not necessarily a good one but it’s what we ended up with).

 

First of all let’s take a look at the old code. Notice that we have a GC.KeepAlive in there. Why would we do that? 
I mean if during the call of Thread.VolatileRead, the weakref object is dead and its finalizer sets m_handle to 0, we’d just read 0.
 That’s fine right?

 

But imagine this code:

 

Object o = new Object();

 

while (true)

{

    WeakReference wr = new WeakReference (o);

 

    if (wr.Target == null)

    {

        …

    }

}

 

o.GetHashCode();

 

We know that o is live during the while loop which means it would be really nice if wr.Target is null (some would consider it a bug if wr.Target was ever null in this case). If we didn’t have the KeepAlive, object wr could be considered dead as soon as its address is passed in to the  Thread.VolatileRead call as the argument. So then h could be 0 and we’d return null. So we want to make sure that if the object the weakref points to is guaranteed to be live during the getter call, you will always get back that object instead of null.

 

Wouldn’t that achieve the same effect since m_handle is an instance data member and if it’s used, 
the *this* object should be kept alive where the last statement is?

 

Well, actually since m_handle is not a volatile, 
jit could generate some code that stores the value of m_handle in a register so it will not need to read the value of m_handle again when it’s at that last statement.

 

Another interesting thing about this code is that it does the IntPtr.Zero check after it first read the value of m_handle.
 But how can m_handle be 0? m_handle is only set to 0 in the WeakReference class’s finalizer code. 
 If we are already KeepAlive-ing the weakref object, it means the object should be live therefore by definition the finalizer should have not been run, right?

 

Well, unfortunately there is a case where m_handle can be 0 while we are in the getter which is when the getter is called in an object’s finalizer. Imagine you have this object hireachy:

 

public class ObjectA

{

    WeakReference wr;

 

    public ObjectA(WeakReference wr0)

    {

        …

        wr = wr0;

    }

 

    …

 

    public ~ObjectA()

    {

        …

        if (wr.Target == null)

        {

            …

        }

        …

    }

}

 

ObjectA a = new ObjectA();

 

When a is not used anymore, at some point both a’s and wr0’s finalizer (assuming a is the only object that contains a reference to wr0) will be put on the finalize queue.

 

Now if a’s finalizer gets to run first, we are fine ‘cause when we are in wr0’s getter, m_handle is still valid. But if wr0’s finalizer gets to run first, then when a’s finalizer is run, m_handle is already set to 0. Of course as we’ve been saying that a finalizer should do no more than releasing native resources, this shouldn’t be a common scenario.

 

So, the idea is to change m_handle to volatile and eliminate the need for calling Thread.VolatileRead. The resulting code looks like this:

 

internal volatile IntPtr m_handle;

 

public Object Target

{

    get

    {

        IntPtr h = m_handle;

        if (IntPtr.Zero == h)

            return null;

 

        Object o = GCHandle.InternalGet(h);

        return (m_handle == IntPtr.Zero) ? null : o;

    }

 

    …

}

 

Notice that KeepAlive is gone because we are reading a volatile value which means the weakref object will be kept alive though out the getter. We still need to check if h is IntPtr.Zero at the beginning because we are still subject to be called from another object’s finalizer.

 

Some people don’t use volatile’s in fear of losing performance ‘cause volatile’s can’t be optimzed by the compiler. In reality though, if you read the volatile into a local when you need to access it frequently and that you are fine with the cached value, no reason to be afraid of using volatile’s.

