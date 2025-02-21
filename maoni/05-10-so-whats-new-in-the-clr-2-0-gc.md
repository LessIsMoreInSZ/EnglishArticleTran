<h1>So, what’s new in the CLR 2.0 GC?</h1>
Certainly that’s one of the most frequently asked questions I get (at the PDC too!). So since PDC already happened I can tell the rest of you about the new stuff happened in GC in CLR 2.0. The slides can be downloaded here. And I will be referring to some of the slides. I must apologize for your having to click on the link to see the slide each time I refer to one since I don’t have a separated site where I can store pictures.

 

Most of what I am going to talk about was covered in my PDC GC talk. BTW, just a few words about the 2005 PDC (skip this paragraph if you are interested in only technical stuff 🙂). I was very pleased with the way it went. I got to talk to lots of customers who told me how awesome our GC was which certainly made me feel very good! There were a couple of customers who did tell us about the problems they had with our GC. One problem we already addressed in Whidbey and the other one was something we were perfectly aware of and had put on our todo list. So no surprises there. I was happy to see the number of people at my talk considering it was the latest possible and lots of people had already left the conference before that. So all in all it was great.

 

Some terminology before we start:

 

Ephemeral generations – gen0 and gen1.

 

Ephemeral segment – the segment that gen0 and gen1 live in (they always live in one segment) and there can only be one ephemeral segment for each heap. So for server GC, if we have 4 heaps we have 4 ephemeral segments (refer to my Using GC Efficiently – Part 2 for explaination for different flavors of GCs).

 

Small object heap – you know LOH (if you don’t, it’s covered in Using GC Efficiently – Part 1), so the rest of the managed heap is called small object heap 🙂

 

Fragmentation control

 

If you ask me what was the most significant improvement in the CLR 2.0 GC from the 1.1 GC, I’d have to say it’s the work we did in reducing fragmentation caused by pinning.

 

One of the biggest problems we faced for managed applications was wasted free space on the managed heap. And sometimes you get OOM while there’s plenty of free space on the managed heap. So reducing fragmentation, in other words, having less space wasted on the managed heap, was crucial. It was observed often in applications that perform IO which needs to pin the buffers for the native APIs to read or write to them. As such, we’ve done a lot of work in CLR 2.0 to help solve this

problem.

 

Slide 25 demonstrates the fragmentation problem caused by pinning. Let’s say we have an application where each request performs some type of socket or file IO. Before GC number X we have a heap that has one segment, which is our ephemeral segment, has bunch of stuff allocated in gen0, including some pinned objects. Now a GC is triggered. After the collection what survived from gen1 was promoted to gen2. For a simplistic view let’s just say nothing from gen0 except the pins survived. In reality what survived is the pinned objects and the other data associated with the requests that pinned those objects for this scenario. So after the collection, gen1 just has the pins ‘cause that’s all that survived from gen0. Gen0 then starts right after the 2nd pin.

 

You can imagine that if this process keeps repeating itself, and let’s say that the pins survived N more GCs, after these GCs we may have a situation where we have expanded the heap and the old ephemeral segment now is a gen2 segment and we acquired a new ephemeral segment. Again we start the new gen0 after the pins in the ephemeral segment.

 

This is a really bad situation because we now have lots of wasted space on our heap. Both our gen2 and gen1 look quite empty. Obviously the less wasted space we can make it, the better.

 

The fragmentation reduction work can be categorized into 2 things: one is called demotion which is to prevent fragmentation from getting into higher generation as much as we can; the other one is reusing existing gen2 segments so we can reuse the free space that has already made into gen2. Let’s talk about them in more details.

 

Demotion, as the opposite of promotion, means the object doesn’t end up in a generation that it’s supposed to be in. Imagine if after the compaction, there’s plenty of space between some pinned objects at the end of the ephemeral segment, it’s more productive to leave them in gen0 instead of promoting them to gen1 because when allocation requests come in the free space before the pins can be used to satisfy allocations. This is demonstrated with slide 26.

 

Segment reuse is relatively straightforward. It’s to take advantage of the existing gen2 segments that have a lot free space but not yet empty (because if they were empty they would have been deleted). Slide 27 demonstrates the problem without segment reuse. So before GC we are running out of space in the ephemeral segment and we need to expand the heap. We have 2 pinned objects. And I didn’t mark their generations because it’s not significant to indicate their generations – they can not be moved so they will stay in this segment and become gen2 objects by definition. There might be pinning on the gen2 segments as well but since that doesn’t affect illustrating the point I preferred to keep the picture simple and left them out.

So after GC we allocate a new segment. The old gen1 in the old ephemeral segment was promoted to gen2 and the old gen0 now lives in the new ephemeral segment as the new gen1 and we will start allocating after gen1 on this segment. The pinned objects are left in the old ephemeral seg since they can not be moved. They are part of gen2 now because they live in a gen2 segment.

 

Slide 28 demonstrates segment reuse. Same picture for before GC. During GC we found that segment 3 is empty enough so instead of allocating a new segment we decide to reuse this segment as the new ephemeral seg. The old ephemeral segment becomes a gen2 segment – as we said there can only be one ephemeral segment – and seg3 becomes the new ephemeral segment. The old gen0 now lives in this segment as the new gen1 and again, we start allocating in gen0 after that.

 

Fixed premature OOM bugs

 

Premature OOM means you have lots of free space yet you are getting an OOM exception. As I said above, having fragmentation can be a form of premature OOM because you can get OOM while you still have lots of free space on your managed heap. Besides that we also fixed some other premature OOM bugs so if you were getting premature OOM I say definitely try CLR 2.0 out if possible. The premature OOM bugs included bugs for both large and small object heap.

 

VM hoarding

 

First of all, this is a feature I would recommand you to not use unless absolutely necessary. The reason we added this feature was for 2 situations – one is when segments are created and deleted very frequently. If you simply can not avoid this, you can specify the VM hoarding feature which instead of releasing the memory back to the OS it puts the segment on a standby list. Note that we don’t do this for the segments that are larger than the normal segment size (generally this is 16MB, for server GC the segments are larger). We will use these segments later to satisfy new segment requests. So next time we need a new segment we will use one from this standby list if we can find one that’s big enough.

 

This feature is also useful for apps that worry about fragmenting the VM space too much and want to hold onto the segments that they already acquired, like some server apps that need to frequently load and unload small DLLs and they want to keep what they already reserved so the DLLs don’t fragment VM all over the place.

Since the feature should be used very conservatively it’s only available via hosting API – you can turn it on by specifying the STARTUP_HOARD_GC_VM flag.

 

That’s all I wanted to talk about for this blog entry. There were of course other changes and it’s not possible to list them all but I think those are (mainly the first two) what affect users the most.

 

https://devblogs.microsoft.com/dotnet/so-whats-new-in-the-clr-2-0-gc/

<h1>那么，CLR 2.0 GC 有什么新特性？</h1> 这确实是我最常被问到的问题之一（在PDC上也是如此！）。既然PDC已经结束，我可以告诉大家CLR 2.0中GC的新特性了。幻灯片可以在这里下载。我会引用其中的一些幻灯片。我必须为每次引用幻灯片时你需要点击链接查看而道歉，因为我没有一个单独的网站来存储图片。
我即将讨论的大部分内容都在我的PDC GC演讲中涵盖了。顺便说一下，简单提一下2005年的PDC（如果你只对技术内容感兴趣，可以跳过这一段 🙂）。我对这次会议的进展感到非常满意。我与许多客户进行了交流，他们告诉我我们的GC有多么出色，这当然让我感到非常高兴！也有几位客户向我们反馈了他们使用GC时遇到的问题。其中一个问题我们已经在Whidbey中解决了，另一个问题我们也已经意识到并列入待办事项。所以没有什么意外的情况。我很高兴看到我的演讲有这么多人参加，考虑到这是最晚的一个演讲，很多人在此之前已经离开了会议。总的来说，这次会议非常成功。

在开始之前，先介绍一些术语：

短暂代（Ephemeral generations） – gen0 和 gen1。

短暂段（Ephemeral segment） – 包含gen0和gen1的段（它们总是位于一个段中），每个堆只能有一个短暂段。因此，对于服务器GC，如果我们有4个堆，就会有4个短暂段（关于不同GC类型的解释，可以参考我的《高效使用GC – 第二部分》）。

小对象堆（Small object heap） – 你知道LOH（如果不知道，可以参考《高效使用GC – 第一部分》），所以托管堆的其余部分被称为小对象堆 🙂

碎片控制

如果你问我CLR 2.0 GC相比1.1 GC最重要的改进是什么，我会说是我们在减少由固定（pinning）引起的碎片方面所做的工作。

托管应用程序面临的最大问题之一是托管堆上的空闲空间浪费。有时你会在托管堆上还有很多空闲空间时遇到OOM（内存不足）错误。因此，减少碎片，换句话说，减少托管堆上的空间浪费，是至关重要的。经常观察到执行IO操作的应用程序会出现这种情况，这些操作需要固定缓冲区以便本地API读取或写入数据。因此，我们在CLR 2.0中做了大量工作来解决这个问题。

幻灯片25展示了由固定引起的碎片问题。假设我们有一个应用程序，每个请求都执行某种类型的套接字或文件IO。在GC X之前，我们有一个堆，其中包含一个段，即我们的短暂段，其中分配了一堆gen0的对象，包括一些固定的对象。现在触发了一次GC。收集后，从gen1存活的对象被提升到gen2。为了简化，我们假设除了固定的对象外，gen0中没有其他对象存活。实际上，存活的是固定的对象以及与这些固定对象相关的请求数据。因此，收集后，gen1中只有固定的对象，因为这是gen0中唯一存活的对象。然后，gen0从第二个固定对象之后开始分配。

你可以想象，如果这个过程不断重复，假设这些固定的对象在N次GC后仍然存活，我们可能会遇到堆扩展的情况，旧的短暂段现在变成了gen2段，我们获得了一个新的短暂段。再次，我们从新的短暂段中的固定对象之后开始分配gen0。

这是一个非常糟糕的情况，因为我们的堆上有大量的空间浪费。我们的gen2和gen1看起来都很空。显然，我们能减少的空间浪费越少越好。

碎片减少的工作可以分为两个方面：一个是降级（demotion），即尽可能防止碎片进入更高的代；另一个是重用现有的gen2段，以便我们可以重用已经进入gen2的空闲空间。让我们更详细地讨论它们。

降级，与提升相反，意味着对象不会进入它应该进入的代。想象一下，如果在压缩后，短暂段末尾的固定对象之间有大量的空闲空间，将它们留在gen0中比提升到gen1更有意义，因为当分配请求到来时，固定对象之前的空闲空间可以用来满足分配。这在幻灯片26中有所展示。

段重用相对简单。它是为了利用现有的gen2段，这些段有很多空闲空间但尚未完全清空（如果它们完全清空，它们就会被删除）。幻灯片27展示了没有段重用的情况。在GC之前，我们的短暂段空间不足，需要扩展堆。我们有两个固定的对象。我没有标记它们的代，因为它们的代并不重要——它们不能被移动，所以它们会留在这个段中，并成为gen2对象。gen2段上也可能有固定对象，但由于这不影响说明问题，我选择保持图片简单，没有将它们包含在内。

在GC之后，我们分配了一个新段。旧的短暂段中的gen1被提升到gen2，旧的gen0现在作为新的gen1存在于新的短暂段中，我们将在这个段中的gen1之后开始分配。固定的对象留在旧的短暂段中，因为它们不能被移动。它们现在是gen2的一部分，因为它们位于gen2段中。

幻灯片28展示了段重用。GC之前的图片相同。在GC期间，我们发现段3有足够的空闲空间，因此我们决定重用这个段作为新的短暂段，而不是分配一个新段。旧的短暂段变成了gen2段——正如我们所说，每个堆只能有一个短暂段——而段3变成了新的短暂段。旧的gen0现在作为新的gen1存在于这个段中，我们再次在gen0之后开始分配。

修复了过早OOM的bug

过早OOM意味着你有很多空闲空间，但仍然遇到了OOM异常。正如我上面所说，碎片化可能是过早OOM的一种形式，因为当托管堆上还有很多空闲空间时，你可能会遇到OOM。除此之外，我们还修复了其他一些过早OOM的bug，所以如果你曾经遇到过过早OOM，我建议你尽可能尝试CLR 2.0。这些过早OOM的bug包括大对象堆和小对象堆的bug。

VM囤积（VM hoarding）

首先，这是一个我建议你除非绝对必要，否则不要使用的功能。我们添加这个功能的原因是为了应对两种情况——一种是段被频繁创建和删除的情况。如果你无法避免这种情况，你可以指定VM囤积功能，它不会将内存释放回操作系统，而是将段放在备用列表中。请注意，我们不会对大于正常段大小的段执行此操作（通常为16MB，服务器GC的段更大）。我们稍后会使用这些段来满足新的段请求。因此，下次我们需要一个新段时，如果我们能找到足够大的段，我们将从备用列表中使用一个段。

这个功能对于担心过度碎片化VM空间并希望保留已经获取的段的应用程序也很有用，比如一些需要频繁加载和卸载小型DLL的服务器应用程序，它们希望保留已经保留的内容，以免DLL在VM中到处碎片化。

由于这个功能应该非常谨慎地使用，它只能通过托管API启用——你可以通过指定STARTUP_HOARD_GC_VM标志来启用它。

这就是我想在这篇博客中讨论的内容。当然还有其他变化，不可能一一列出，但我认为这些（主要是前两个）是对用户影响最大的变化。