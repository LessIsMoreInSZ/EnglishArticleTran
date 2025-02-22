<h1>gen2-free-list-changes-in-clr-4-6-gc</h1>

I wanted to mention this because I started seeing posts about it. In 4.6 we improved the way we use the gen2 free list to promote gen1 survivors into. 
Unfortunately there was a perf bug that I didn’t notice till it was fairly late so it wasn’t approved to be checked into 4.6 at the time. Now that I am seeing more people hitting it, 
I have checked the fix into the next hotfix rollup for 4.6 which you will be able to download when it’s released in the near future (I will update the exact location when it’s available).

The symptom for this bug is that you are seeing GC taking a lot longer than 4.5 – if you collect ETW events this will show that gen1 GCs are taking a lot longer.

Note that this is NOT due to any deficiency in background GC (I’ve seen some posts saying that background GC is at fault – this is not the case), 
it’s simply because of the bug in the free list management. 
You may see that the bug manifests itself a lot less when you disable Concurrent GC but that’s only because background GC does not compact, 
by design, so it uses the free list a lot more. Some people also observed turning on Server GC makes the problem “go away” but again, 
that’s just the way it manifests itself in Server GC (by turning on Server GC you just made GC throughput a lot higher 
so even though there’s more work to do the total GC pause is lower as work is now done by multiple GC threads).

What makes this bug show up is when the objects that die are small ones that scattered around the heap. 
This is not exactly very easy to debug so unless you want to put in a bunch of effort (to debug and potentially to change your code) I would recommend to wait for the hotfix rollup.

https://devblogs.microsoft.com/dotnet/gen2-free-list-changes-in-clr-4-6-gc/

我想提到这一点，因为我开始看到一些关于它的帖子。在.NET 4.6中，我们改进了使用gen2空闲列表来提升gen1存活对象的方式。不幸的是，有一个性能问题直到比较晚的时候我才注意到，
因此当时没有批准将其合并到4.6中。现在我看到越来越多的人遇到了这个问题，我已经将修复程序合并到了4.6的下一个热修复汇总中，你可以在不久的将来下载它
（当它发布时，我会更新具体的位置）。

这个问题的症状是，你发现GC花费的时间比4.5要长得多 —— 如果你收集了ETW事件，这将显示gen1 GC花费的时间要长得多。

请注意，这并不是由于后台GC（Background GC）的任何缺陷（我看到一些帖子说后台GC是罪魁祸首 —— 事实并非如此），而是由于空闲列表管理中的一个错误。你可能会发现，
当你禁用并发GC时，这个问题的表现会少很多，但这只是因为后台GC在设计上不会进行压缩，因此它更多地使用了空闲列表。有些人还观察到，启用服务器GC（Server GC）会使问题“消失”，
但这只是因为它在服务器GC中的表现方式不同（通过启用服务器GC，你大大提高了GC的吞吐量，因此即使有更多的工作要做，总的GC暂停时间也会更短，因为现在工作是由多个GC线程完成的）。


这个问题的出现是因为当死亡的对象是散布在堆中的小对象时，它更容易显现出来。这个问题并不容易调试，因此除非你愿意投入大量精力（调试并可能更改代码），
否则我建议等待热修复汇总的发布。