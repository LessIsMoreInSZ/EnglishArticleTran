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

<h1>高效使用GC - 第二部分</h1> 在本文中，我将讨论运行时中现有的不同类型的GC，每种GC的设计目标以及它们之间的不同工作方式，以便你可以为你的应用程序选择合适的GC类型。
现有GC类型

今天我们有以下几种GC类型：

（1）关闭并发GC的工作站GC
（2）开启并发GC的工作站GC
（3）服务器GC
如果你正在编写一个独立的托管应用程序，并且没有进行任何自己的配置，默认情况下你会得到2) 类型。这可能会让很多人感到惊讶，因为我们的文档并没有详细提到并发GC，有时将其称为“后台GC”（同时将工作站GC称为“前台GC”）。

如果你的应用程序被托管，宿主可能会为你更改GC类型。

值得提到的一点是，如果你请求服务器GC，并且应用程序在单处理器机器上运行，你实际上会得到1)，因为工作站GC针对单处理器机器的高吞吐量进行了优化。

设计目标

类型1) 的设计目标是实现单处理器机器上的高吞吐量。我们在GC中使用动态调整来观察分配和存活模式，因此我们调整调整以使每个GC尽可能高效。

类型2) 的设计适用于响应时间至关重要的交互式应用程序。并发GC允许更短的暂停时间。它以一些内存和CPU为代价来实现这个目标，因此你会得到一个稍大的工作集和稍长的收集时间。

顾名思义，类型3) 是为服务器应用程序设计的，其中典型的场景是有一池工作线程都在做类似的事情。例如，处理相同类型的请求或进行相同类型的交易。所有这些线程倾向于有相同的分配模式。服务器GC旨在多处理器机器上实现高吞吐量和高度可扩展性。

它们如何工作

让我们从关闭并发GC的工作站GC开始。流程如下：

1.一个托管线程正在进行分配；
2.它分配完了（我将解释这意味着什么）；
3.它触发一个GC，该GC将在此线程上运行；
4.GC调用SuspendEE来挂起托管线程；
5.GC完成它的工作；
6.GC调用RestartEE来重新启动托管线程；
7.托管线程再次开始运行。
在步骤5) 中，如果你在调试器中断时，你会看到所有托管线程都停止等待GC完成。SuspendEE不关心本机线程，例如，如果线程正在调用一些Win32 API，它将在GC工作时运行。

对于代，我们有一个称为“预算”的概念。每个代都有自己的预算，这个预算是动态调整的。因为我们总是在Gen0中分配，你可以将Gen0预算视为一个分配限制，当超过这个限制时，将触发GC。
这个预算与GC堆段大小完全不同，比段大小小得多。

CLR GC可以进行压缩或非压缩收集。非压缩收集，也称为清除，比涉及复制内存的压缩收集要便宜，后者是一个昂贵的操作。

当并发GC开启时，最大的不同在于Suspend/Restart。如我所述，并发GC允许更短的暂停时间，因为应用程序需要保持响应。所以，不是在GC期间不让托管线程运行，并发GC只在这些线程上非常短的时间暂停几次。
其余时间，托管线程正在运行并在需要时进行分配。我们在并发GC情况下为Gen0设置了一个更大的预算，这样应用程序在GC运行时有足够的空间进行分配。
然而，如果在并发GC期间应用程序已经分配了Gen0预算中的内容，如果需要更多分配，分配线程将被阻塞（等待GC完成）。

注意，由于Gen0和Gen1收集非常快，当我们进行Gen0和Gen1收集时，进行并发GC是没有意义的。因此，我们只考虑在需要执行Gen2收集时进行并发GC。
如果我们决定进行Gen2收集，我们然后会决定这个Gen2收集应该是并发的还是非并发的。

服务器GC完全是另一回事。我们为每个CPU创建一个GC线程和一个独立的堆。GC在这些线程上发生，而不是在分配线程上。流程如下：

1.一个托管线程正在进行分配；
2.它在它正在分配的堆上分配完了；
3.它发出一个事件信号来唤醒GC线程进行GC，并等待它完成；
4.GC线程运行，完成GC并发出一个事件信号，表示GC已完成（当GC正在进行时，所有托管线程都会像在工作站GC中一样被挂起）；
5.托管线程再次开始运行。
配置

要关闭并发GC，请在你的应用程序配置文件中使用以下配置：

<configuration> <runtime> <gcConcurrent enabled=”false”/> </runtime> </configuration>
要开启服务器GC，请在你的应用程序配置文件中使用以下配置，如果你使用的是Everett SP1或Whidbey：

<configuration> <runtime> <gcServer enabled=“true”/> </runtime> </configuration>
在Everett SP1之前，唯一支持的方式是通过宿主API（查看CorBindToRuntimeEx）。