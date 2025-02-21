<h1>understand-the-problem-before-you-try-to-find-a-solution</h1>
So far I’ve never written a blog entry that gives out philosophical advices on doing performance work. 
But lately I thought perhaps it’s time to write such an entry because I’ve seen enough people who looked really hard at some performance counters (often not correct ones) or some other data and asked tons of questions such as “is this allocation rate too high? It looks too high to me.” or “my gen1 size is too big, right? It seems big…”, before they have enough evidence to even justify such an investigation and questions.

 

Now, if you are just asking questions to satisfy your curious mind, that’s great. 
I am happy to answer questions or point you at documents to read. But for people who are required to investigate performance related issues, 
especially when the deadline is close, my advice is “understand the problem before you try to find a solution”. Determine what to look at based on evidence, 
not based on your lack of knowledge in the area unless you already exhausted areas that you do know about.
 Before you ask questions related to GC, ask yourself if you think GC is actually the problem. 
 If you can’t answer that question it really is not a good use of your time to ask questions related to GC.

 

I’ve seen many cases when something seems to go wrong in a managed application, people immediately suspect GC without any evidence to support that suspicion. 
Then they start to ask questions – often very random – hoping that they would somehow find the solution to fix their problem without understand what the problem is. 
That’s not logical is it? So stop doing it!

 

So how do you know the right problems to solve, I would recommend the following:

 

1) Having knowledge about fundamentals really helps.

 

What are fundamentals? Performance in general comes down to 2 things – memory and CPU. 
Knowing basics of these 2 areas helps a lot in determining which area to look at. Obviously this involves a lot of reading and experimenting. 
I will list some memory fundamentals to get you started:

 

Some fundamentals of memory

 

·          Each process has its own and separated virtual address space; all processes on the same machine share the physical memory (plus the page file if you have one). 
On 32-bit each process has a 2GB user mode virtual address space by default.

 

·          You, as an application author, work with virtual address space – you don’t ever manipulate physical memory directly. 
If you are writing native code usually you use the virtual address space via some kind of win32 heap APIs (crt heap or the process heap or the heap you create) – these heap APIs will allocate and free virtual memory on your behalf; if you are writing managed code, GC is the one who allocates/frees virtual memory on your behalf.

 

·          Virtual address space can get fragmented – in other words, there can be “holes” (free blocks) in the address space. When a VM allocation is requested, 
the VM manager needs to find one free block that’s big enough to satisfy that allocation request – if you only got a few free blocks whose sum is big enough it won’t work. 
This means even though you got 2GB, you don’t necessarily see all 2GB used.

 

·          VM can be in different states – free, reserved and committed. Free is easy. The difference between reserved and committed is what confused people sometimes.
 First of all, you need to recognized that they are different states. Reserved is saying “I want to make this region of memory available for my own use”. 
 After you reserve a block of VM that block can not be used to satisfy other reserve requests. 
 At this point you can not store any of your data in that block of memory yet – to be able to do that you have to commit it, 
 which means you have to back it up with some physical storage so you can store stuff in it. When you are looking at the memory via perf counters and what not, 
 make sure you are looking at the right things. You can get out of memory if you are running out space to reserve, or space to commit.

 

·          If you have a page file (by default you do) you can be using it even if your physical memory pressure is very low. 
What happens is the first time your physical memory pressure gets high and the OS needs to make room in the physical memory to store other data, 
it will back up some data that’s currently in physical memory in the page file. 
And that data will not be paged in until it’s needed so you can get into situations where the physical memory load is very low yet you are observing paging.


 

2) Knowing what your performance requirements are is a must.


 

If you are writing a server application, 
it’s very likely that you want to use all the memory and CPU that’s available because people delicate the machine completely to run your app so why waste resources? 
If you are writing a client application, 
totally different story – you’ll have to know how to cope with other applications running on the same machine. 
There’re no rules such as “you have to make your app use as little memory as possible”.

 

When you’ve decided that there is a problem, dig into it instead of guess what might be wrong. If your app is using too much memory, 
look at who is using the memory. If you’ve decided that the managed heap is using too much memory, look at why.
 Managed heap using too much memory generally means you survive too much in your app. Look at what is holding on to those survivors.

 https://devblogs.microsoft.com/dotnet/understand-the-problem-before-you-try-to-find-a-solution/

 <h1>性能优化的哲学思考</h1> 我从未写过关于性能优化方法论的文章，但最近意识到是时候分享一些经验了——我见过太多开发者紧盯着性能计数器（通常是错误的指标）或数据，提出诸如"这个分配率是不是太高了？看起来好高啊"或"我的Gen1堆太大了对吧？"等问题，而他们甚至没有足够的证据支撑这样的调查。
先问正确的问题
如果你只是出于好奇提问，这很好。但如果是需要解决实际性能问题的开发者，我的建议是：在寻找解决方案之前先理解问题。根据证据而非知识盲区来选择调查方向。在提出与GC相关的问题之前，先问问自己GC是否真的是问题的根源。如果连这个问题都无法回答，那么针对GC的提问只会浪费时间。

我曾目睹太多案例：当托管应用出现异常，开发者会毫无根据地怀疑GC，然后开始提出各种随机问题，试图在不理解问题本质的情况下找到解决方案。这就像无头苍蝇乱撞，既无逻辑又低效。

性能优化的基本原则
1. 夯实基础至关重要
性能问题归根结底是内存和CPU两大核心：

内存基础认知

每个进程拥有独立的虚拟地址空间（32位系统默认2GB用户空间）

应用程序操作的是虚拟地址空间，开发者通过堆API（本地代码）或GC（托管代码）间接管理物理内存

虚拟地址空间可能出现碎片化：即使总剩余空间足够，分散的空闲块仍可能导致分配失败

内存的三种状态：

Free：未使用的空间

Reserved：已预留但未实际使用的区域（不可被其他请求使用）

Committed：已提交并关联物理存储的区域

即使物理内存压力低，页面文件仍可能被使用（当OS需要腾出物理内存时）

2. 明确性能需求
服务器应用通常需要最大化利用硬件资源，而客户端应用则要考虑与其他进程共存。不存在"必须最小化内存使用"的绝对规则，关键是根据场景定义合理的性能目标。

问题诊断方法论
当确定存在性能问题后，应系统性地深入分析而非盲目猜测：

内存问题诊断：

使用性能计数器（如".NET CLR Memory"中的"# Total committed bytes"）

区分虚拟内存与物理内存使用情况

分析内存持有者（使用CLRProfiler或SOS调试扩展）

CPU问题诊断：

通过任务管理器查看线程调度情况

分析高优先级线程对低优先级线程的压制

检查是否存在优先级误用导致的锁竞争

避免常见误区
不要过度调用GC.Collect：合理设置触发间隔（如5秒），并通过GC.CollectionCount验证必要性

谨慎设置内存限制：通过作业对象（Job Object）管理时需预备OOM处理机制

权衡本地缓存与托管堆：当需要处理远超物理内存的数据时，考虑使用低碎片化的本地缓存方案

性能优化是科学与艺术的结合。记住：没有银弹，只有对问题本质的深刻理解。下次当你准备说"我的应用卡死了"时，请先打开任务管理器，看看是CPU在狂欢，还是内存已枯竭——答案往往就藏在最基础的指标里。

 

