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

今天在修改代码时，发现了一个值得分享的优化案例。众所周知，WeakReference类提供了Target属性的getter和setter方法，用于获取和设置弱引用指向的对象。更多关于WeakReference的细节可以参考《高效使用GC – 第三部分》。

以下代码仅用于说明，并非生产环境中的实际代码。

原始代码分析
假设WeakReference类的代码最初是这样的：

csharp
复制
internal IntPtr m_handle;

public Object Target
{
    get
    {
        IntPtr h = m_handle;
        if (IntPtr.Zero == h)
            return null;

        Object o = GCHandle.InternalGet(h);
        h = Thread.VolatileRead(ref m_handle);
        GC.KeepAlive(this);
        return (h == IntPtr.Zero) ? null : o;
    }
    // setter省略
}
m_handle是实现弱引用功能的弱GCHandle，它指向你希望弱引用对象指向的目标对象。

问题：Thread.VolatileRead的性能开销
Thread.VolatileRead是一个相对重量级的操作，性能较差（虽然当初使用这个API有原因，但未必是最优选择）。

为什么需要GC.KeepAlive？
首先，我们来看一下为什么需要GC.KeepAlive。如果在Thread.VolatileRead调用期间，弱引用对象被回收，并且其终结器将m_handle设置为0，我们会读取到0，这没问题吧？

但考虑以下代码：

csharp
复制
Object o = new Object();

while (true)
{
    WeakReference wr = new WeakReference(o);

    if (wr.Target == null)
    {
        // ...
    }
}

o.GetHashCode();
我们知道o在while循环期间是存活的，因此如果wr.Target返回null，可能会被认为是一个bug。如果没有GC.KeepAlive，wr对象可能在Thread.VolatileRead调用时被回收，
导致h为0并返回null。因此，我们需要确保如果弱引用指向的对象在getter调用期间是存活的，你总能得到该对象而不是null。

m_handle的易失性问题
m_handle是一个实例成员，如果它被使用，this对象应该会被保持存活，对吧？实际上，由于m_handle不是volatile，JIT可能会生成一些代码将m_handle的值存储在寄存器中，
因此在最后一条语句时不需要重新读取m_handle的值。

m_handle为0的情况
另一个有趣的问题是，为什么在读取m_handle的值后还要检查IntPtr.Zero？m_handle只有在WeakReference类的终结器代码中才会被设置为0。如果我们已经通过GC.KeepAlive保持了弱引用对象的存活，
那么终结器不应该运行，对吧？

不幸的是，有一种情况会导致m_handle在getter中被设置为0，即在某个对象的终结器中调用getter。例如：

csharp
复制
public class ObjectA
{
    WeakReference wr;

    public ObjectA(WeakReference wr0)
    {
        wr = wr0;
    }

    ~ObjectA()
    {
        if (wr.Target == null)
        {
            // ...
        }
    }
}

ObjectA a = new ObjectA();
当a不再被使用时，a和wr0的终结器（假设a是唯一持有wr0引用的对象）会被放入终结队列。如果a的终结器先运行，那么在wr0的getter中，m_handle仍然是有效的。
但如果wr0的终结器先运行，那么在a的终结器运行时，m_handle已经被设置为0。当然，正如我们一直强调的，终结器应该只用于释放本地资源，因此这种情况并不常见。

优化方案：使用volatile
为了解决这个问题，我们将m_handle改为volatile，并消除对Thread.VolatileRead的调用。优化后的代码如下：

csharp
复制
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
    // setter省略
}
注意到GC.KeepAlive被移除了，因为我们读取的是一个volatile值，这意味着弱引用对象在getter执行期间会被保持存活。我们仍然需要在开始时检查h是否为IntPtr.Zero，因为我们仍然可能从另一个对象的终结器中被调用。

关于volatile的性能
有些人因为担心性能问题而避免使用volatile，因为volatile不能被编译器优化。但实际上，如果你将volatile值读入一个局部变量，并在需要频繁访问时使用该缓存值，就没有理由害怕使用volatile。