<h1>No GCs for your allocations?</h1>

Several people mentioned Java’s “no GC” GC proposal and asked if we can add such a thing. So I thought it deserves a blog post.

Short answer – we already have such a feature and that’s called the NoGCRegion. 
The GC.TryStartNoGCRegion API allows you to tell us the amount of allocations you’d like to do and when you stay within it, no GCs will be triggered. 
And you are free to revert back to doing normal GCs when you want to by calling GC.EndNoGCRegion. This is a better model than having a separate GC.

Long answer –

It’s a better model because you have the flexibility to revert back to doing GCs when you need to. 
Would you rather have to spin up a whole new process just so that you can then do work that needs to do GCs (‘cause if you don’t collect, well, 
your memory usage is just going to keep growing and unless you allocate really little you are going to run out pretty soon)?

And yes, there’s currently limitations on how much you can allocate with NoGCRegion. 
You can’t allocate an arbitrary amount on SOH – I limited it to the SOH segment size just because we’ve always had the same size for SOH segments so far. 
There’s no theoretical limit that segments have to be the same size. 
It’s a matter of doing the work to make sure we don’t have places that accidently (artificially) enforce such a limit. There are other design options but I won’t get into the details here.

So this means if you are using Server GC you are able to ask for a lot more memory on SOH because its SOH segment size is a lot larger. 
Currently (and this has been the case for a long time) the default seg size on 64-bit for Server GC is 4GB and we do cut that in half when you have > 4 procs and again if you have > 8 procs. 
So you have > 8 procs it means the SOH segment size is 1GB each.

On Desktop you have ways to make the SOH segment size larger –

use the gcSegmentSize app config or use the ICLRGCManager::SetGCStartupLimits

For LOH as long as we can reserve and commit that much memory this will allow you to allocate ‘cause LOH can already have totally different sized segments.

We choose to make sure we can commit the amount of memory you ask for instead of randomly throwing you OOM because 
when you use such a feature you really should have a good idea how much you want to allocate. 
If you do have a scenario that says “I just want to allocate till I get OOM and I don’t care if I randomly get OOM” please let me know – I’d like to understand what the rationale is.

I did have some bugs when I first checked this into 4.6.x (thanks Matt Warren for reporting). I’ve made fixes in the current release. 
But I wanted to explain how it’s supposed to work so you know what’s a bug and what’s a design limitation.

And we also are still paying the write barrier cost while you are allocating in the NoGCRegion – I kept it that way 
because when you revert back to doing GCs you do need the info that the write barrier recorded if we don’t do anything special. 
However, if we so choose, we can just not have the write barrier for NoGCRegion and when we need to revert back 
to doing GCs we just promote everything that’s still live to gen2 
(would be reasonable to do as at this point you are very likely done with all the temporary stuff and what’s left is justified to be in the old generation) 
and get the write barrier back in the picture again. I didn’t do it this way because write barrier cost generally doesn’t come up – not to say that it can’t ever be a problem. It’s just that we always need to prioritize the work based on how much it’s needed.

https://devblogs.microsoft.com/dotnet/no-gcs-for-your-allocations/许多人提到了 Java 的“无 GC”GC 提案，并询问我们是否可以添加类似的功能。因此，我认为这个问题值得写一篇博客来详细说明。

简短回答——我们已经具备了这样的功能，它被称为 NoGCRegion。

GC.TryStartNoGCRegion API 允许你告诉系统你希望分配多少内存，并且只要你保持在该范围内，就不会触发垃圾回收（GC）。当你需要时，
你可以通过调用 GC.EndNoGCRegion 回到正常的 GC 模式。这比单独设置一个 GC 更为灵活。

更详细的回答如下：

这是一种更好的模型，因为它允许你在需要时灵活地切换回执行 GC 的模式。难道你宁愿启动一个新的进程，仅仅是为了完成需要进行 GC 的工作吗？
（因为如果不收集垃圾，你的内存使用量只会不断增加，除非你分配的内存非常少，否则很快就会耗尽内存）。

是的，目前 NoGCRegion 存在一些限制。例如，你不能在大对象堆（SOH）上分配任意数量的内存——我将其限制为 SOH 段大小，这是因为到目前为止 SOH 段的大小一直保持一致。
实际上，段大小并没有理论上的限制必须相同。这只是需要确保我们没有意外地（人为地）施加这样的限制。当然，还有其他设计选项，但这里就不深入细节了。

这意味着，如果你使用的是服务器端 GC（Server GC），你可以请求更多的 SOH 内存，因为它的 SOH 段大小更大。目前（并且长期以来一直是这样），
64 位系统中 Server GC 的默认段大小为 4GB。如果你有超过 4 个处理器，我们会将这个值减半；如果处理器数超过 8 个，还会再次减半。
所以，当处理器数超过 8 个时，SOH 段大小为每个 1GB。

在桌面环境中，你可以通过以下方式增加 SOH 段大小：

使用 gcSegmentSize 应用程序配置选项。
或者使用 ICLRGCManager::SetGCStartupLimits。
对于大对象堆（LOH），只要我们可以保留并提交所需数量的内存，就可以进行分配，因为 LOH 已经可以拥有完全不同的段大小。

我们选择确保能够提交你请求的内存数量，而不是随机抛出 OOM（Out of Memory）异常，因为在使用这种功能时，你应该对要分配的内存量有一个明确的预期。如果你有一个场景，
比如“我只是想分配内存直到 OOM，我不在乎是否随机出现 OOM”，请告诉我——我想了解这种需求的合理性。

当我在 4.6.x 版本中首次引入这一功能时确实有一些 bug（感谢 Matt Warren 报告问题）。我已经在当前版本中修复了这些问题。但我希望解释它是如何工作的，
这样你就知道哪些是 bug，哪些是设计上的限制。

此外，在使用 NoGCRegion 分配内存时，我们仍然会支付写屏障（write barrier）的成本。我之所以保持这种方式，是因为当你回到执行 GC 的模式时，如果没有采取特殊措施，
我们需要写屏障记录的信息。然而，如果我们选择不为 NoGCRegion 添加写屏障，当我们需要回到 GC 模式时，可以简单地将所有仍存活的对象提升到第 2 代（gen2）。
这是合理的，因为在这一点上，你很可能已经完成了所有的临时操作，剩下的对象应该被归类为老一代对象。我没有采用这种方式，
因为写屏障的成本通常不是问题——并不是说它永远不会成为问题。我们总是需要根据实际需求优先级来决定工作重点。