<h1>How to evaluate info you read on garbage collectors</h1>

Just a word before I actually start this blog entry – I apologize for approving some of the comments so late – it appears that our blogs’ policy has changed and 
it would make some comments as pending without obvious reasons to me.

Also as one of the ways to support the community I was thinking I could have a specific time on a regular cadence (could be an hour every month to start with)
 when I would answer questions on the .NET GC. I’m thinking perhaps gitter is the best forum but I am not sure – I don’t know how popular it is with .NET developers 
 (that care about memory performance) so if you have any suggestions please post them in comments.

From time to time I get someone that tells me “Hey Maoni, I just read this GC article that says their GC is so great! I think it’s better than your GC!”. 
Then I read the article and found out that it said nothing that actually indicated their GC is better than ours, in fact, 
it’s very often the case it didn’t even say nearly enough to make an assessment on that GC’s performance, much less to do a comparison with another GC. 
So I thought I’d write something that helps to explain how to evaluate information you read on GCs.

First of all, comparing GC performance is definitely non trivial work. When you run even a micro benchmark, it can easily involve enough from the non GC part of a runtime. 
So you are actually not comparing just the GCs unless you isolate it down to the parts that’s related to GC perf. 
I remember one time we had someone who showed us a comparison on a benchmark between our runtime and another runtime and it was spending much more time in GC in our runtime. 
It turned out it was due to our codegen making a decision that caused us to survive a lot more in our runtime so GC had to work a lot harder.

When you compare something that’s more than a micro benchmark, eg, a web server macro benchmark we are just talking comparing frameworks at this point, 
it’s a long way from drawing any conclusion on the GC performance alone. 
One framework could have a completely different allocation and survival pattern from the other which means one can impose a very different amount of work on GC from the other. 
For example, one framework could choose to allocate much fewer objects or tend to survive much fewer because of the programming patterns in that framework 
so it simply doesn’t need a stellar GC to manage memory.

Even when you have isolated it to a point that you are mostly comparing the GC performance, as anyone who’s even slightly familiar with GCs, 
you know there are tons of tradeoffs to make in a GC. Without actually understand the architecture, the mechanisms and the policy of a GC,
 it’s far from complete to run a few tests to determine what a GC’s performance is. For example, 
 one GC could have made a choice to optimize for generational behavior while another may not have (could be because the framework simply does not exhibit generational behavior).

When you understand the architecture of a GC you know its limits so you can make a statement about what it’s capable of in theory. 
Of course 2 GCs with the same architecture can still have dramatically different performance on the same exact test 
(let’s say you’ve isolated enough to make such a test) because how optimized the implementation is – on the hot path, 
one could be using a more clever way to implement something while the other falls short.

At this point I hope you realize how careful you need to be when you are not doing experiments yourself and simply reading some information written on a GC.

If a GC article explains how it does something (like an architectural doc, or a blog entry that explains how the GC handles a specific scenario), 
it’s totally fine to read the description. But without some knowledge on how effective/innovative/sophisticated this allows a GC to be, 
it’s impossible to say whether you should believe the author’s claim in how good the GC is, especially when you want to do a comparison. 
I’ve seen GC articles that made the most mundane stuff sound like it’s something that sets their GC apart from others GCs, 
when in reality it’s something that most other GCs of the same genre do or did at some point and already improved upon.

If a GC article makes a claim about the performance, this is where you really need to be careful when choosing whether you should believe the claim or not. 
A good article would tell you the workload, the machine config and the GC config that were used to measure the performance, and relevant performance data. 
If the workload is simply enough (eg, it doesn’t use layers of libraries) or you have enough understanding about the workload to know how much work GC is doing, 
you can get an idea about GC perf in that framework. And if you port this workload to a different framework and isolate it enough to compare just the GC performance, 
you can then make a statement about GC performance comparison.

On the other hand, an article that simply makes claims like “our GC’s latency is less than 20 milliseconds” without describing what was run to get that number is meaningless. 
It could simply be a false statement (as in, you could write a test that simply makes that GC have longer pauses); 
or it could be the case that the framework the GC lives in simply never (or in 99% of the case) needs to handle situations that would make it have pauses more than 20ms. 
Without knowing which case it is you cannot make any decision about this GC which in turn means you can’t compare it with another GC.

And even when you are reading numbers printed out by some GCs you need to be careful how you interpret those numbers. 
Some GCs would indicate which numbers are for STW (Stop The World). Obviously during STW, all your threads are stopped. 
But STW is not the only way your threads will get interrupted by a GC. Some GCs choose to aggressively do mutator assist 
(meaning the user threads will take on some GC work before it actually gets to allocate) so while a thread is doing the mutator assist work it’s the same as being stopped. 
I’ve seen some very long mutator assistant pauses in some GCs that are much longer than their STW pauses. 
Obviously in that case if you are counting on that thread to finish a request that request’s going to take a long time.

In order to accurately assess latency it’s best to measure it yourself. In our GC micro benchmarks we would measure around a request 
(which usually does one or more allocations and some assignments) for latency. Since it’s a GC benchmark, 
usually the latency is due to GC pauses but sometimes it may also be due to other factors like the thread isn’t getting scheduled.

Some of our first party customers implement a diagnostics pipeline that measures the latency of each request and for long requests
 they can attribute the latency to specific tasks that happened during that request – network IO, disk IO, GC or something else.
  Depending on what OS/framework you work with, you might have the option to implement this too.

https://devblogs.microsoft.com/dotnet/how-to-evaluate-info-you-read-on-garbage-collectors/


在正式开始这篇博客之前，我想先说几句——我为一些评论的延迟审核道歉——似乎我们博客的政策发生了变化，某些评论会被标记为“待审核”，而原因对我来说并不明显。

此外，作为支持社区的一种方式，我在考虑是否可以定期安排一个固定的时间（比如从每月一小时开始），专门回答关于 .NET GC 的问题。我觉得 Gitter 可能是最好的平台，但我不确定——我不清楚它在关心内存性能的 .NET 开发者中有多受欢迎。如果你有任何建议，请在评论中告诉我。

时不时会有人告诉我：“嘿，Maoni，我刚读了一篇关于 GC 的文章，说他们的 GC 非常棒！我觉得比你们的 GC 更好！”然后我去读了那篇文章，发现它并没有提供任何实际表明他们的 GC 比我们更好的信息。事实上，很多时候文章甚至没有提供足够的信息来评估那个 GC 的性能，更不用说与其他 GC 进行比较了。所以，我想写点东西，帮助大家理解如何评估你读到的关于 GC 的信息。

首先，比较 GC 的性能绝对是一项非平凡的工作。即使你运行一个微基准测试，它也很容易涉及到运行时的非 GC 部分。因此，除非你将测试隔离到与 GC 性能相关的部分，否则你实际上并不是在仅仅比较 GC。我记得有一次，有人向我们展示了一个基准测试的结果，比较了我们的运行时和另一个运行时，结果显示我们的运行时在 GC 上花费了更多时间。后来发现，这是由于我们的代码生成器做了一个决策，导致我们的运行时存活了更多的对象，因此 GC 不得不更加努力地工作。

当你比较的不仅仅是微基准测试时，比如一个 Web 服务器的宏基准测试，我们实际上是在比较框架，这离单独得出 GC 性能的结论还有很长的路要走。一个框架的分配和存活模式可能与另一个框架完全不同，这意味着一个框架可能给 GC 带来与另一个框架完全不同的工作量。例如，一个框架可能选择分配更少的对象，或者由于编程模式的原因，存活的对象更少，因此它根本不需要一个出色的 GC 来管理内存。

即使你已经将比较隔离到主要是在比较 GC 性能的程度，任何对 GC 稍有了解的人都知道，GC 中有大量的权衡。如果不真正理解一个 GC 的架构、机制和策略，仅仅通过运行几个测试来确定一个 GC 的性能是远远不够的。例如，一个 GC 可能选择优化分代行为，而另一个 GC 可能没有这样做（可能是因为框架本身并没有表现出分代行为）。

当你理解了一个 GC 的架构时，你就知道了它的极限，因此你可以在理论上说明它的能力。当然，两个具有相同架构的 GC 在完全相同的测试中仍然可能表现出截然不同的性能（假设你已经隔离得足够好来进行这样的测试），因为它们的实现优化程度不同——在热路径上，一个 GC 可能使用了更聪明的方式来实现某些功能，而另一个则没有。

说到这里，我希望你意识到，当你不亲自做实验，而只是阅读一些关于 GC 的信息时，你需要多么小心。

如果一篇 GC 文章解释了它是如何做某件事的（比如一篇架构文档，或者一篇博客文章，解释了 GC 如何处理特定场景），那么阅读这些描述是完全没问题的。但如果没有一些关于这种方法如何使 GC 变得有效/创新/复杂的知识，你不可能判断是否应该相信作者关于 GC 有多好的说法，尤其是当你想要进行比较时。我见过一些 GC 文章，把最普通的东西说得像是让他们的 GC 与其他 GC 区别开来的关键，而实际上，这是大多数同类 GC 在某个时刻已经做过或改进过的事情。

如果一篇 GC 文章声称其性能如何，那么在选择是否相信这种说法时，你真的需要非常小心。一篇好的文章会告诉你用于测量性能的工作负载、机器配置和 GC 配置，以及相关的性能数据。如果工作负载足够简单（例如，它没有使用多层库），或者你对工作负载有足够的了解，知道 GC 做了多少工作，你就可以对该框架中的 GC 性能有一个概念。如果你将这个工作负载移植到另一个框架，并隔离得足够好以仅比较 GC 性能，那么你就可以对 GC 性能进行比较。

另一方面，如果一篇文章只是简单地声称“我们的 GC 延迟小于 20 毫秒”，而没有描述是运行什么测试得到这个数字的，那么这种说法是毫无意义的。它可能只是一个错误的陈述（比如，你可以写一个测试，让那个 GC 的暂停时间更长）；或者可能是该 GC 所在的框架根本不需要（或在 99% 的情况下不需要）处理会导致暂停时间超过 20 毫秒的情况。在不知道是哪种情况的情况下，你无法对这个 GC 做出任何决定，也就无法将其与另一个 GC 进行比较。

即使你在阅读某些 GC 打印出的数字时，也需要小心如何解释这些数字。一些 GC 会明确指出哪些数字是 STW（Stop The World）的。显然，在 STW 期间，所有的线程都会停止。但 STW 并不是你的线程被 GC 中断的唯一方式。一些 GC 选择积极地执行 Mutator Assist（意思是用户线程在实际分配之前会承担一些 GC 工作），因此当一个线程在执行 Mutator Assist 工作时，它实际上就像被停止了一样。我见过一些 GC 的 Mutator Assist 暂停时间非常长，甚至比它们的 STW 暂停时间还要长。显然，在这种情况下，如果你指望那个线程完成一个请求，那么该请求将花费很长时间。

为了准确评估延迟，最好自己进行测量。在我们的 GC 微基准测试中，我们会围绕一个请求（通常执行一个或多个分配和一些赋值）测量延迟。由于这是一个 GC 基准测试，通常延迟是由于 GC 暂停引起的，但有时也可能是由于其他因素，比如线程没有被调度。

我们的一些第一方客户实现了一个诊断管道，用于测量每个请求的延迟，对于长时间请求，他们可以将延迟归因于该请求期间发生的特定任务——网络 IO、磁盘 IO、GC 或其他任务。根据你使用的操作系统/框架，你可能也有机会实现这一点。

