<h1>Debugging with the Right Tools</h1>

Wow, it’s been almost a year since I last blogged J We just shipped CLR V4.0. Yay!

 

An internal email thread prompted me to write this blog entry – one very powerful tool I wanted to point out when you need to debug/investigate issues is your debugger
 (if your debugger is also windbg/cdb that is J since that’s what I use and that’s what I will talk about in this article).

 

For those of you who are interested in investigating memory related issues at all, whether it’s because you don’t like your current app’s memory usage or you simply want to improve, 
learning to use the debugger is invaluable. If you haven’t started using windbg/cdb, I would strongly encourage you to – I promise you won’t regret it.

 

I’ve talked about using SoS before. We’ve added some more SoS commands in V4.0 and some of those are related to GC like !AnalyzeOOM, 
!GCWhere and !FindRoots. You can read about them on the MSDN page. But I wanted to talk about some techniques that may not be obvious from reading reference material.

 

What do you want to know about GC when your program is? 2 of the most common things are

 

1) Why are GCs being triggered?

2) Why do GCs take this much time?

 

And the answers are, respectively:

 

1) The difference of the heap before and after a GC

2) The survivors from that GC

 

Let me explain those in detail. Each generation has its own allocation budget which I explained here. 
It’s a value that we set and when exceeded we’d want to trigger a GC. 
If we do a collection and discovered that there’s a lot of memory survived we would set this allocation budget very big which means 
you’d need to allocate more into that generation for us to want to trigger a GC. 
The rationale is that then we’d have a chance to reclaim some sizeable space back next time we do a GC. Otherwise we’d do all this work for a GC and not be able to find much dead memory.

 

So if you do a !dumpheap right before a GC you see what objects you have; then you do a !dumpheap right after that GC you see some of those objects disappeared.
 It’s those objects that triggered this GC. Why? ‘cause if we didn’t have any objects disappearing, we wouldn’t be doing GC (‘cause we’d get no dead space back). 
 When you hear people say “you are doing gen2 GCs because you are churning gen2” this is preciously the explanation for that.

 

As of now, the CLR GC doesn’t compact the LOH. So if you want to look at the LOH you can see exactly which parts of the memory disappeared. 
Here’s a sample output of a heap range before and after a gen2 GC. Before the gen2 GC (I formatted the !dumpheap with the methodtable pointer replaced with their readable names):

 

Address              MethodTable          Size

————————————————–

00000007f7ffe1e0     System.Byte[]        2,080,000

00000007f81f9ee0     System.String        100,000

00000007f8212580     System.Byte[]        140,000

 

After this gen2 GC for the same heap range:

 

Address              MethodTable          Size

————————————————–

00000007f7ffe1e0     Free                 2,320,000

 

So those 3 objects were collected. As the application author those objects may very well ring a bell for you why they were allocated 
so you can go from there see if/how you can do some optimizations.

 

The survivors from the collection are what you see in the output of !dumpheap at the end of that GC.

 

In this MSDN article I talked about how to set a breakpoint at the end of a gen2 GC. To set a bp at the beginning of a gen2 GC simply replace RestartEE with SuspendEE.
 If you want to do this for gen0 GCs, replace 2 with 0. 
 Note that this applies to only non concurrent/background GCs though since concurrent/background GCs would not have the EE suspended for the most part.

 

Often I see people quickly go search for some other tools when they really could whip out the debugger and set a few breakpoints to see what things look like. 
Of course there are many useful tools out there – and they tend to aggregate info for you so you don’t have to do that yourself – but sometimes it’s so convenient to just use the most direct and handy tool which is your debugger.

 

Another classic example is when people ask me for help with debugging memory leaks and the easiest and quickest solution is to set a conditional bp on kernel32!VirtualAlloc for
native memory leaks. I’ve seen this over and over again and this is how the conversation would usually go:

 

someone, “Maoni, I have a managed app that’s leaking memory. Can you help me with it?”

me, “Is it a managed memory leak?”

someone, “I don’t know.”

me, “How big is your managed heap? Can you do the SoS !dumpheap command?”

someone, “it’s only X MBs” [but his whole process uses much more than that]

me, “Does it grow? If you run your app for a while does the managed heap size change?”

someone, “No” (or “Not much, not nearly as much as the whole memory usage increases”)

me, “Can you take a look at your virtual memory address space by doing !address?” [!address gives you info on each virutal memory chunk. 
It’s also a windbg/cdb debugger extension but it’s in ext.dll  which comes with the debugger package.]

someone, “It looks like there’re a lot of chunks of size S…”

me, “Can you set a conditional bp on kernel32!VirtualAlloc when the allocate size is S?” [Which means: bp kernel32!virtualalloc “.if (@edx==S) {kp} .else {g}”]

[some time elapses]

someone, “It looks like they all have the same callstack!!!”

 

At this point it’s clear what causes their memory leak.

 

If you need to see continuous output of some commands without disturbing the process too much you can always log it to a file (.logopen c:\mydebugsession.txt) and analysis the log 
after the process runs for a while.

 

As I mentioned there are tools that will aggregate this info for you (eg, the DebugDiag tool you can find on microsoft.com 
which shows you the aggregated callstack from all the VirtualAlloc calls) but if you’ve never used those tools it may take a while to set them up and learn how to use them. 
The debugger is something you have right there and use all the time so it’s something you can fairly conveniently solve with a debugger I tend to use that.

https://devblogs.microsoft.com/dotnet/debugging-with-the-right-tools/

哇，距离我上次写博客已经快一年了！我们刚刚发布了CLR V4.0，太棒了！
一封内部邮件提醒我写了这篇博客——我想指出的是，当你需要调试或调查问题时，一个非常强大的工具就是你的调试器（如果你的调试器是windbg/cdb，那就更好了，因为这是我使用的工具，也是本文要讨论的内容）。

对于那些对内存相关问题感兴趣的人来说，无论是因为你不喜欢当前应用程序的内存使用情况，还是你只是想优化它，学习使用调试器都是非常宝贵的。如果你还没有开始使用windbg/cdb，我强烈建议你尝试一下——我保证你不会后悔。

我之前已经讨论过使用SoS（SOS调试扩展）。在V4.0中，我们添加了一些新的SoS命令，其中一些与GC相关，比如!AnalyzeOOM、!GCWhere和!FindRoots。你可以在MSDN页面上阅读它们的详细信息。但我想讨论一些从参考材料中可能不太明显的技巧。

当你的程序运行时，你想了解GC的哪些信息？
最常见的两个问题是：

为什么触发了GC？

为什么GC花了这么长时间？

答案分别是：

GC前后堆的差异

GC中存活的对象

让我详细解释一下。每个代（generation）都有自己的分配预算（allocation budget），我在这里解释过。这是一个我们设置的值，当超过这个值时，我们希望触发GC。如果我们进行一次GC后发现有很多内存存活下来，我们会将这个分配预算设置得非常大，这意味着你需要在该代中分配更多内存才能触发GC。其理由是，这样我们下次进行GC时有机会回收大量空间。否则，我们做了这么多GC工作，却找不到多少死亡内存。

因此，如果你在GC之前运行!dumpheap，你会看到当前堆中的对象；然后在GC之后再次运行!dumpheap，你会发现一些对象消失了。正是这些对象触发了这次GC。为什么？因为如果没有对象消失，我们就不会进行GC（因为我们无法回收任何死亡空间）。当你听到有人说“你正在进行gen2 GC，因为你在gen2中频繁分配”时，这正是解释。

大对象堆（LOH）的情况
目前，CLR GC不会压缩大对象堆（LOH）。因此，如果你想查看LOH，你可以准确地看到哪些内存部分消失了。以下是一个gen2 GC前后堆范围的示例输出。

在gen2 GC之前（我将!dumpheap的输出格式化，将方法表指针替换为可读的名称）：

复制
Address              MethodTable          Size
————————————————–
00000007f7ffe1e0     System.Byte[]        2,080,000
00000007f81f9ee0     System.String        100,000
00000007f8212580     System.Byte[]        140,000
在gen2 GC之后，相同的堆范围：

复制
Address              MethodTable          Size
————————————————–
00000007f7ffe1e0     Free                 2,320,000
所以这三个对象被回收了。作为应用程序的作者，这些对象可能会让你想起它们为什么被分配，从而你可以从这里开始，看看是否/如何进行一些优化。

如何设置GC断点
在这篇MSDN文章中，我讨论了如何在gen2 GC结束时设置断点。要在gen2 GC开始时设置断点，只需将RestartEE替换为SuspendEE。如果你想对gen0 GC执行此操作，将2替换为0。请注意，这仅适用于非并发/后台GC，因为并发/后台GC在大部分时间内不会挂起EE（Execution Engine）。

调试内存泄漏的经典示例
我经常看到人们在遇到问题时迅速寻找其他工具，而实际上他们可以拿出调试器并设置几个断点来查看情况。当然，有很多有用的工具——它们通常会为你聚合信息，这样你就不必自己动手——但有时使用最直接和方便的工具（即调试器）是如此方便。

另一个经典的例子是，当人们向我寻求帮助调试内存泄漏时，最简单和最快的解决方案是为本机内存泄漏设置一个条件断点kernel32!VirtualAlloc。我一次又一次地看到这种情况，通常对话是这样的：

某人：“Maoni，我有一个托管应用程序在泄漏内存。你能帮我吗？”

我：“是托管内存泄漏吗？”

某人：“我不知道。”

我：“你的托管堆有多大？你能运行SoS的!dumpheap命令吗？”

某人：“它只有X MB”（但他的整个进程使用了比这多得多的内存）

我：“它会增长吗？如果你运行应用程序一段时间，托管堆的大小会变化吗？”

某人：“不会”（或“变化不大，远不及整个内存使用量的增加”）

我：“你能通过运行!address查看你的虚拟内存地址空间吗？”（!address会显示每个虚拟内存块的信息。它也是windbg/cdb调试器扩展，但它在ext.dll中，随调试器包一起提供。）

某人：“看起来有很多大小为S的内存块……”

我：“你能在kernel32!VirtualAlloc上设置一个条件断点，当分配大小为S时触发吗？”（即：bp kernel32!virtualalloc ".if (@edx==S) {kp} .else {g}"）

[一段时间后]

某人：“看起来它们都有相同的调用栈！！！”

此时，内存泄漏的原因已经非常清楚了。

记录调试会话
如果你需要查看某些命令的连续输出而不太干扰进程，你可以将其记录到文件中（.logopen c:\mydebugsession.txt），然后在进程运行一段时间后分析日志。

正如我提到的，有些工具会为你聚合这些信息（例如，你可以在microsoft.com上找到的DebugDiag工具，它会显示所有VirtualAlloc调用的聚合调用栈），但如果你从未使用过这些工具，可能需要一些时间来设置和学习如何使用它们。调试器是你随时可用的工具，因此我倾向于使用它来方便地解决问题。

总结
调试器是一个强大的工具，尤其是在调查内存相关问题时。通过设置断点和使用SoS命令，你可以深入了解GC的行为和内存泄漏的原因。虽然有许多工具可以帮助你聚合信息，但调试器是最直接和方便的选择，尤其是当你已经熟悉它时。