<h1>Job support in the GC</h1>

I’ve noticed that more and more of our customers started to use job objects to restrict some form of resources for their processes such as CPU and memory. 
So I wanted to talk about the kind of support the GC offers to help with your managed processes running in a job object.

The first thing to notice is that a managed process actually just gets the benefits of a job object without GC doing anything since, well, 
your process just looks like a normal process to the OS. So without doing anything special in the GC, you already get the following benefits:

if you limit the # of CPUs in your job object, eg, you have 16 CPUs but you run your process in a job object that allows 8, GC will think you only have 8 CPUs which means if you are using Server GC, you get 8 heaps instead of 16.

if you limit the memory usage, ie, commit size and working set, you also get the corresponding benefits in your managed process, 
eg, if you have total 10GB commit size but specified 5GB on your job object, when your process is trying to commit more than 5GB (this includes the commit for your managed heap), it will fail.

However, there’s one scenario that’s not covered. GC becomes more aggressive when it detects that the physical memory load is high (90+%). But the OS simply does not reflect “physical memory load” in a job object, ie, if you specify your working set limit to 2GB on a machine with 8GB and you are using 1.5GB physical memory, GlobalMemoryStatusEx will return 1.5/8=19% for dwMemoryLoad instead of 1.5/2=75%.

So I needed to make some changes in the GC to account for this. This went into coreclr a couple of months ago. This worked out well for customers who previously were getting OOMs in their job objects.

https://devblogs.microsoft.com/dotnet/job-support-in-the-gc/

我注意到越来越多的客户开始使用**作业对象（Job Objects）**来限制其进程的某些资源，例如CPU和内存。因此，我想讨论一下GC如何支持在作业对象中运行的托管进程。

作业对象的基本支持
首先需要注意的是，托管进程实际上可以在GC不做任何特殊处理的情况下，直接受益于作业对象，因为对于操作系统来说，你的进程看起来就像一个普通进程。因此，即使GC没有做任何特殊处理，你已经可以获得以下好处：

限制CPU数量：
如果你在作业对象中限制了CPU数量，例如你有16个CPU，但你在一个只允许8个CPU的作业对象中运行你的进程，GC会认为你只有8个CPU。这意味着如果你使用服务器GC（Server GC），你将获得8个堆，而不是16个。

限制内存使用：
如果你限制了内存使用（即提交大小和工作集），你的托管进程也会获得相应的好处。例如，如果你的总提交大小为10GB，但在作业对象中指定了5GB，当你的进程尝试提交超过5GB的内存时（包括托管堆的提交），它将失败。

GC对作业对象的增强支持
然而，有一个场景没有被覆盖。当GC检测到物理内存负载较高（90%以上）时，它会变得更加积极。但操作系统不会在作业对象中反映“物理内存负载”。例如，如果你在一台有8GB内存的机器上指定了2GB的工作集限制，并且你使用了1.5GB的物理内存，GlobalMemoryStatusEx将返回1.5/8=19%作为dwMemoryLoad，而不是1.5/2=75%。

因此，我需要在GC中进行一些更改来解决这个问题。这些更改在几个月前已经合并到CoreCLR中。这些更改对于那些之前在作业对象中遇到**OOM（内存不足）**问题的客户来说效果很好。

具体实现
为了实现这一点，GC现在会检查进程是否在作业对象中运行，并根据作业对象的内存限制调整其行为。具体来说：

检测作业对象：
GC会检测进程是否在作业对象中运行，并获取作业对象的内存限制。

调整内存负载计算：
如果进程在作业对象中运行，GC会根据作业对象的内存限制重新计算内存负载。例如，如果作业对象的内存限制为2GB，而进程使用了1.5GB，GC会将内存负载计算为1.5/2=75%，而不是1.5/8=19%。

更积极的GC行为：
当检测到高内存负载时，GC会变得更加积极，以释放更多内存，从而避免OOM。

示例场景
假设你在一台有16GB内存的机器上运行一个托管进程，并将其放入一个内存限制为4GB的作业对象中。以下是如何工作的：

内存使用情况：

进程使用了3GB内存。

作业对象的内存限制为4GB。

内存负载计算：

传统方式：3/16=18.75%（低内存负载，GC不会特别积极）。

新方式：3/4=75%（高内存负载，GC会变得更加积极）。

GC行为：

GC会检测到高内存负载（75%），并开始更频繁地回收内存，以避免进程接近作业对象的4GB限制。

总结
通过增强对作业对象的支持，GC现在可以更好地适应在资源受限环境中运行的托管进程。这对于那些需要在作业对象中运行托管进程并希望避免OOM的客户来说是一个重要的改进。如果你正在使用作业对象来限制资源，请确保你的运行时版本包含这些更改，以获得最佳的内存管理行为。

如果你有任何问题或反馈，请随时分享！