<h1>Using GC Efficiently – Part 2</h1>
In this article I’ll talk about different flavors of GC, the design goals behind each of them and how they work differently from each other so you can make a good decision of which flavor of GC you should choose for your applications.

 

Existing GC flavors in the runtime today

 

We have the following flavors of GC today:

 

1)      Workstation GC with Concurrent GC off

2)      Workstation GC with Concurrent GC on

3)      Server GC

 

If you are writing a standalone managed application and don’t do any config on your own, you get 2) by default. This might come as a surprise to a lot of people because our document doesn’t exactly mention much about concurrent GC and sometimes refers to it as the “background GC” (while refering Workstation GC as “foreground GC”).

 

If your application is hosted, the host might change the GC flavor for you.

 

One thing worth mentioning is if you ask for Server GC and the application is running on a UP machine, you will actually get 1) because Workstation GC is optimized for high throughput on UP machines.

 

Design goals

 

Flavor 1) is designed to achieve high throughput on UP machines. We use dynamic tuning in our GC to observe the allocation and surviving patterns so we adjust the tuning as the program runs to make each GC as productive as possible.

 

Flavor 2) is designed for interactive applications where the response time is critical. Concurrent GC allows for shorter pause time. And it trades some memory and CPU to achieve this goal so you will get a slightly bigger working set and slightly longer collection time.

 

Flavor 3), as the name suggests, is designed for server applications where the typical scenario is you have a pool of worker threads that are all doing similar things. For example, handling the same type of requests or doing the same type of transactions. All these threads tend to have the same allocation patterns. The server GC is designed to have high throughput and high scalibility on multiproc machines.

 

How they work

 

Let’s start with Workstation GC with Concurrent GC off. The flow goes like this:

 

1)      A managed thread is doing allocations;

2)      It runs out of allocations (and I’ll explain what this means);

3)      It triggers a GC which will be running on this very thread;

4)      GC calls SuspendEE to suspend managed threads;

5)      GC does its work;

6)      GC calls RestartEE to restart the managed threads;

7)      Managed threads start running again.

 

In step 5) you will see that all managed threads are stopped waiting for GC to complete if you break into the debugger at that time. SuspendEE isn’t concerned with native threads so for example if the thread is calling out to some Win32 APIs it’ll run while the GC is doing it work.

 

For the generations we have this concept called a “budget”. Each generation has its own budget which is dynamically adjusted. Since we always allocate in Gen0, you can think of the Gen0 budget as an allocation limit that when exceeded, a GC will be triggered. This budget is completely different from the GC heap segment size and is a lot smaller than the segment size.

 

The CLR GC can do either compacting or non compacting collections. Non compacting, also called sweeping, is cheaper than compacting that involves copying memory which is an expensive operation.

 

When Concurrent GC is on, the biggest difference is with Suspend/Restart. As I mentioned Concurrent GC allows for shorter pause time because the application need to be responsive. So instead of not letting the managed threads run for the duration of the GC, Concurrent GC only suspends these threads for a very short period of times a few times during the GC. The rest of the time, the managed threads are running and allocating if they need to. We start with a much bigger budget for Gen0 in the Concurrent GC case so we have enough room for the application to allocate while GC is running. However, if during the Concurrent GC the application already allocated what’s in the Gen0 budget, the allocating threads will be blocked (to wait for GC to finish) if they need to allocate more.

 

Note since Gen0 and Gen1 collections are very fast, it doesn’t make sense to have Concurrent GC when we are doing Gen0 and Gen1 collections. So we only consider doing a Concurrent GC if we need to do a Gen2 collection. If we decide to do a Gen2 collection, we will then decide if this Gen2 collection should be concurrent or non concurrent.

 

Server GC is a totally different story. We create a GC thread and a separated heap for each CPU. GC happens on these threads instead of on the allocating thread. The flow looks like this:

 

1)      A managed thread is doing allocations;

2)      It runs out of allocations on the heap its allocating on;

3)      It signals an event to wake the GC threads to do a GC and waits for it to finish;

4)      GC threads run, finish with the GC and signal an event that says GC is complete (When GC is in progress, all managed threads are suspended just like in Workstation GC);

5)      Managed threads start running again.

 

Configuration

 

To turn Concurrent GC off, use

 

<configuration>

    <runtime>

        <gcConcurrent enabled=”false”/>

    </runtime>

</configuration>

 

in your application config file.

 

To turn Server GC on, use

 

<configuration>

    <runtime>

        <gcServer enabled=“true”/>

    </runtime>

</configuration>

 

in your application config file, if you are using Everett SP1 or Whidbey. Before Everett SP1 the only supported way is via hosting APIs (look at CorBindToRuntimeEx).

