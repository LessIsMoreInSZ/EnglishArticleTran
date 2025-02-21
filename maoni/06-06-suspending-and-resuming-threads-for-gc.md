<h1>Suspending and resuming threads for GC</h1>
First of all, suspension and resumption of threads is not really part of the GC. GC calls functions to do the suspension and the resumption as a service provided in the CLR. 
Other components in the CLR also use this service such as the debugger implementation. 
But it’s true that suspending and resuming because of the GC is the most significant usage of this service.

Secondly, how threads are suspended and resumed works the same way in Workstation GC (with or without Concurrent GC) and Server GC.
 Workstation GC with Concurrent GC just suspends and resumes threads more times than the other two cases.

Now let’s talk about suspension. A managed thread could be running either managed code or native code at the time. If it’s running native code which can either be from

·         the EE (Execution Engine) of the CLR – for example, EE implements some helper function for your managed call which is on the stack at the time. 
In this case the EE code tells the suspension code how it should be suspended. 
A common scenario is the EE code will wait for GC to finish so the suspension code doesn’t need to suspend it (it already suspends itself). Or

·         the native code is not from the EE, which means it’s running some native code via Interop, we don’t need to suspend this thread 
because it shouldn’t be doing anything with managed objects that can are not pinned. When this thread needs to return to managed code, however, 
it will need to check if a GC is in progress and if so it needs to wait for GC to finish.

For the managed threads that are running managed code it can be in either

§         a mode where it can be stopped at any time (because the JIT recorded enough info for each instruction) in which case it can be suspended right there. 
We then redirect it to some function that will make this thread wait for GC to complete. Or

·         a mode where it can’t be stopped at any time, in which case the thread’s return address will be hijacked, 
meaning we replace the return address by a stub which will suspend this thread. What this stub does is it will suspend this thread to wait for GC to finish.

In both case the function that was used to suspend the thread knows how to resume them correctly when GC finishes.

Resumption works much simpler. When a GC is finished, all the threads that were suspended will wake up (they were waiting on an event for GC to finish) to resume execution. 

https://devblogs.microsoft.com/dotnet/suspending-and-resuming-threads-for-gc/

<h1>GC中的线程挂起和恢复</h1> 首先，线程的挂起和恢复并不是GC（垃圾回收）的一部分。GC调用CLR提供的服务来完成线程的挂起和恢复。CLR中的其他组件也会使用这项服务，例如调试器的实现。
但确实，由于GC而挂起和恢复线程是这项服务最主要的用途。
其次，线程的挂起和恢复在Workstation GC（无论是否启用并发GC）和Server GC中的工作方式是相同的。启用并发GC的Workstation GC只是比其他两种情况更频繁地挂起和恢复线程。

现在我们来谈谈挂起。托管线程在挂起时可能正在运行托管代码或本地代码。如果它正在运行本地代码，这些代码可能来自：

CLR的执行引擎（EE）：例如，EE实现了一些辅助函数，这些函数可能位于当前调用栈中。在这种情况下，EE代码会告诉挂起代码应该如何挂起它。常见的场景是EE代码会等待GC完成，因此挂起代码不需要主动挂起它（它已经自行挂起了）。

非EE的本地代码：这意味着线程通过Interop运行一些本地代码。我们不需要挂起这种线程，因为它不应该对未固定的托管对象进行任何操作。然而，当这种线程需要返回到托管代码时，它需要检查是否有GC正在进行，
如果有，则需要等待GC完成。

对于运行托管代码的托管线程，它可能处于以下两种模式之一：

可以随时停止的模式：因为JIT（即时编译器）为每条指令记录了足够的信息，因此可以立即挂起它。然后我们会将其重定向到一个函数，该函数将使该线程等待GC完成。

不能随时停止的模式：在这种情况下，线程的返回地址会被劫持，即我们用存根（stub）替换返回地址，该存根将挂起该线程。这个存根的作用是挂起线程以等待GC完成。

在这两种情况下，用于挂起线程的函数都知道如何在GC完成后正确地恢复它们。

恢复的过程则简单得多。当GC完成时，所有被挂起的线程都会唤醒（它们正在等待一个事件以表示GC完成），然后继续执行。