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