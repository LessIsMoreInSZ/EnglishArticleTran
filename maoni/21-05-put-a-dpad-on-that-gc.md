Say what? Yes, there's a new feature called DPAD for regions – regions is what we are currently working on in .NET 6 to convert segments to. In this blog post I will first give some introduction to regions then talk about the DPAD feature. Note that it's unlikely we will officially support regions by the end of 6.0 as there's a lot of work involved – our current plan is to ship it in clrgc.dll as an experimental feature that you can turn on with a config. In fact, this is the way I want to ship large GC features from now on – we ship them first with the standalone GC (ie, in clrgc.dll) so folks can try them out, then we officially turn them on in coreclr.dll so they are on by default.

As you know, so far we've been operating on segments. Segments served us well for years but I had started to notice the limitation it had as folks put more kinds of workloads on our framework. They are a foundation for our memory management so it's a big deal to switch away from them. When we were approaching .NET 6 I decided it was time that we evolved away from them so this is what our team has been spending a lot of time on lately. So what are the key differences between segments and regions? Segments are large units or memory – on Server GC 64-bit if the segment sizes are 1GB, 2GB or 4GB each (for Workstation it's much smaller – 256MB) on SOH. Regions are much smaller units, they are by default 4MB each. So you might ask, "so they are smaller, why is that significant?". To answer that, let's first review how segments work.

This is currently what SOH looks like on a heap when we have only one segment –


![alt text](images/dpad.png)

when we have multiple segments it could look like this

dpad
![alt text](images/dpad-1.png)

or this

dpad
![alt text](images/dpad-2.png)

The blue and yellow spaces are all committed memory (for an explanation of generation starts please see this video) on a segment. Each segment keeps track of what's committed on the segment so we know if we need to commit more. And all the free spaces on that segment are also committed memory. This works well when we use the free spaces to accommodate objects because we can use the memory right away – it's already committed. But imagine a scenario where we have free spaces in one generation, say gen0 because there's some async IO going on that caused us to demote a bunch of pins in gen0, that we don't actually use (this could be due to not waiting for so long to do the next GC or we'd have accumulated too much survival which means the GC pause would be too long). Wouldn't it be nice if we could use those free spaces for other generations if they need them! Same with free spaces in gen2 and LOH – you might have some free spaces in gen2, it would be nice to use them to allocate some large objects. We do decommit on a segment but only the end of the segment which is after the very last live object on that segment (denoted by the light gray space at the end of each segment). And if you have pinning that prevents the GC from retracting the end of the segment, then we can only form free spaces and free spaces are always committed memory. Of course you might ask, "why don't you just decommit the middle of a segment that has large free spaces?". But that requires bookkeeping to remember which parts in the middle of a segment are decommitted so we need to re-commit them when we want to use them to allocate objects. And now we are getting into the idea of regions anyway, which is to have much smaller amounts of memory being manipulated separately by the GC.

With regions the generations look uniformed and we no longer have this "ephemeral segment" concept. We have gen0 and gen1 regions, just like we have gen2 regions.


![alt text](images/dpad-3.png)
Of course the number of regions in each generation could be vastly different. But they all consist of these small memory units. LOH does have larger regions (it's 8x the size of SOH regions, so 32MB each). When we free a region, we return it to the free region pool and regions in that pool can be grabbed by any generation, or even by any other heap if needed. So you will no longer see this scenario where you have some huge free spaces in gen2 or LOH and they are not getting used for a long time (this can happen if your app behavior goes through phases where one phase could survive a lot more memory than another, and GC hasn't felt the need to do a full compacting GC).

We always have to make tradeoffs in GC work. And with regions we do gain a lot of flexibility. But we also have to give up some things. One thing that makes segments very attractive is we do have a contiguous ephemeral range since gen0 and gen1 always live on the ephemeral segment and always right next to each other. We take advantage of this when we set cards in the write barrier. If you do obj0.f = obj1, and we detect that obj1 is not in the ephemeral range, we don't need to set the card 'cause we don't need it (a card only needs to be set if obj1 is in a younger generation than obj0 and if obj1 is not in the ephemeral range it means it's either in gen2 or LOH/POH which are all considered logically as part of generation 2 (but internally are tracked as gen3 and gen4 and I’m using LOH and gen3 interchangeably in this post). And that means it's either in the same generation as obj0 or in an older generation than obj0). But we only do this optimization for Workstation GC as Server GC has multiple ephemeral ranges and we didn't want to have to compare against all of them in the write barrier code. In regions we'll either set cards unconditionally (which would regress Workstation GC pauses a bit but keep the same performance for Server GC) or check the region of obj1 in the write barrier which would be more expensive than checking for an ephemeral range in the most optimal type of write barriers. The benefits regions bring should more than justify this though.

Now we can talk about the DPAD feature. DPAD stands for Dynamic Promotion And Demotion. Strictly speaking, demotion is already dynamic as it only happens dynamically based on pinning situations. If you've read my mem-doc, demotion is explained there (and if you haven't, I strongly encourage you to). Basically demoting an object means it's not getting promoted like it normally would. For segments, demotion means we set a range of the ephemeral segment to be the "demoted range" and this range can only be from one point in the middle of the ephemeral segment to the end of that segment. In other words we don't ever set a range in the middle of the ephemeral segment to be the demoted range. This is exactly because with segments gen1 has to be right before gen0 on the ephemeral segment (on the same heap, that is). So we can't have a gen1 portion, followed by a gen0 portion and then followed by a gen1 portion again.

Promotion is a common concept in GC – it means if an object survives a generation, it's now considered part of the higher generation. So if you have a long lived small object on the SOH, it will eventually be promoted into gen2. But that means it will require 2 GCs for that to happen. I am planning to provide an API that gives users an option to tell the GC to allocate a new object directly into a certain generation so you could allocate objects you know will survive to gen2 directly in gen2 (I haven't implemented this API so far as it's also much easier with regions support so I'm planning to implement it when we have converted to regions). But that doesn't cover all cases since sometimes it's difficult for the users to know whether an object will "most likely survive to gen2". And you might be using a library and have no control over the allocation of those objects. One very distinct scenario where this would happen is with datastructure resizing. Let's say you or the library you are using allocated a list which needs to grow its capacity. So it allocates a new T[] object that can hold twice the amount of elements of the old one. Now it creates a bunch of child elements for the 2nd half. Now, if the new array is large enough to get on LOH and the new children are all small objects so they are in gen0 –


![alt text](images/dpad-4.png)
(I'm only showing an 8-element array and 4 new children for illustration purpose, if this was an object[] obviously it'd require more elements to get on the LOH)

In the segment case we'd see this –
![alt text](images/dpad-5.png)


Since the new array is considered part of gen2, it means all the new elements that got created in gen0 will survive to gen2 (unless a gen2 GC happens soon after and discovers the parent array dead already, which could happen but not very likely; and if it does happen that's pretty unfortunate 'cause you'd be paying all this cost to create a large object only to discard it right away). But for that to happen, it'd need to go through at least 2 GCs. It's very likely we'll first observe a gen0 or gen1 GC which survives these children to gen1.

![alt text](images/dpad-6.png)

And then the next gen1 GC will discover they are all still alive because they are held live by that array in LOH. Now it promotes them all to gen2.

![alt text](images/dpad-7.png)

For this scenario we'd prefer to just promote them directly into gen2. But this is difficult to do for segments. We could keep track of which plugs consist of these objects or mostly these objects but when we mark we don't know which objects would form plugs together. And when we are forming plugs we have lost that info. We could keep track of this info on a larger granularity. But guess what, that's basically like regions! Because we'd want to divide up a segment into smaller units to track that info. So for regions this is pretty easy. When we mark, we know exactly how much survives on each region – as we mark each object we track which region we need to attribute the survived bytes to. So we know how much of the survival is done by card marking.

With regions when we encounter a region that mostly consists of objects like these child objects that are held live due to card marking, we have a choice –

![alt text](images/dpad-8.png)

We can choose to promote this region directly to gen2 –
![alt text](images/dpad-9.png)


So that region is threaded into gen2. The other region that belonged to gen0 had survivors that were compacted into the gen1 region and gen0 gets a new region for allocations.

With the current implementation I'm doing this only for regions that mostly filled with objects like these. And since regions are small, it's very likely some regions are filled with these and then we have another region that's partly filled with these and partly some truly temporary objects. The complexity of separating them is unlikely worthy (you can think of it as we are back to the segments case for this particular region).

There are some complications (with GC there's almost always some complications…) when we do this. One example is since we now just made gen0 objects survive in gen2 we'll need to make sure if they point to any gen that's not gen2, the cards need to be set for those objects. We do this when we are going through survivors in the relocate phase (since we already have to go through each object anyway).

So pun (partially) intended, this DPAD feature is kind of like a D-pad… you can tell a region which direction it needs to go – up or down (in GC terms that would be older or younger). There are various scenarios where we'd want to dynamically promote or demote a region and the example I gave above is just one of them. The point is with regions we can dynamically assign a generation number we want a region to end up in because generations are no longer contiguous and there's no specific orders generations have to be relative to each other (of course as you can see above, there are implementation details that need to be taken care of for different scenarios). This is much more flexible than the limited amount of demotion we were doing with segments. And when we thread regions at the end of the GC, we just need to thread them into their assigned regions. With my initial checkin for DPAD, I've implemented 3 scenarios where we would dynamically promote or demote regions. In the future we'll implement more.


https://devblogs.microsoft.com/dotnet/put-a-dpad-on-that-gc/

### 什么是 DPAD？

DPAD 是一个新特性，全称为 **Dynamic Promotion And Demotion**（动态提升与降级）。严格来说，降级已经是动态的，因为它仅基于固定对象的情况动态发生。如果你读过我的内存文档（mem-doc），降级在那里有详细解释（如果你还没读过，我强烈建议你去阅读）。简而言之，降级一个对象意味着它不会像通常那样被提升到更高的代。

对于段（segments）来说，降级意味着我们在临时段（ephemeral segment）上设置一个范围作为“降级范围”。这个范围只能从临时段中间的一个点延伸到该段的末尾。换句话说，我们永远不会在临时段的中间设置一个范围作为降级范围。这是因为，在段模型中，gen1 必须紧挨着 gen0 存在于临时段上（在同一堆中）。因此，我们不能先有一个 gen1 区域，然后是一个 gen0 区域，再跟着另一个 gen1 区域。

---

#### 提升的概念

提升是垃圾回收器中的一个常见概念——它意味着如果一个对象在一个代中存活下来，那么它现在被视为更高代的一部分。例如，如果你在 SOH（小型对象堆）上有生命周期较长的小型对象，它最终会被提升到 gen2。但这意味着需要经过两次 GC 才能完成这一过程。

我计划提供一个 API，让用户可以选择告诉 GC 直接将新对象分配到某个特定的代中，这样你可以直接将那些你知道会存活到 gen2 的对象分配到 gen2 中（目前还没有实现这个 API，因为有了区域支持后会更容易实现，所以我计划在转换为区域模型后再实现它）。然而，这并不能覆盖所有情况，因为有时用户很难知道某个对象是否会“很可能存活到 gen2”。而且你可能正在使用一个库，对这些对象的分配没有控制权。一个非常典型的场景是数据结构的扩容。

---

#### 数据结构扩容的场景

假设你或你使用的库分配了一个列表，需要增加其容量。于是它分配了一个新的 `T[]` 对象，可以容纳旧数组两倍数量的元素。接着，它为新数组的后半部分创建了一组子元素。如果新数组足够大以至于进入了 LOH（大对象堆），而这些新子元素都是小型对象，因此位于 gen0 中：

![示意图](images/dpad-4.png)

（这里为了说明，只展示了一个包含 8 个元素的数组和 4 个新子元素；如果是 `object[]`，显然需要更多元素才能进入 LOH。）

在段模型的情况下，我们会看到如下情况：

![段模型](images/dpad-5.png)

由于新数组被视为 gen2 的一部分，这意味着所有在 gen0 中创建的新元素都会存活到 gen2（除非紧接着发生一次 gen2 GC，并发现父数组已经死亡，这种情况虽然可能发生，但不太可能；如果确实发生了，那就很不幸，因为你付出了创建大对象的成本，却马上将其丢弃）。但是，要达到这一点，至少需要经历两次 GC。我们很可能会首先观察到一次 gen0 或 gen1 GC，这些子元素因此被提升到 gen1。

![第一次GC](images/dpad-6.png)

接下来的一次 gen1 GC 将发现这些子元素仍然存活，因为它们被 LOH 上的数组引用。此时，它们全部被提升到 gen2。

![第二次GC](images/dpad-7.png)

对于这种场景，我们更希望直接将这些子元素提升到 gen2。但在段模型下，这很难做到。我们可以跟踪哪些块由这些对象组成或主要由这些对象组成，但在标记阶段，我们不知道哪些对象会形成块。当我们形成块时，这些信息已经丢失了。我们可以以更大的粒度跟踪这些信息，但猜猜看，这实际上就像区域一样！因为我们想要将段划分为更小的单元来跟踪这些信息。因此，在区域模型下，这就变得非常简单。在标记阶段，我们知道每个区域中存活了多少对象——随着我们标记每个对象，我们跟踪哪些区域需要归因于存活字节。因此，我们知道有多少存活是由卡标记（card marking）完成的。

---

### 区域（Regions）简介

区域是我们当前在 .NET 6 中正在研究的一项功能，旨在将段转换为更小的内存管理单元。在这篇博客中，我将首先介绍区域，然后讨论 DPAD 特性。

需要注意的是，我们不太可能在 .NET 6.0 结束时正式支持区域，因为这项工作涉及的内容很多。我们目前的计划是以实验特性的方式将其发布到 `clrgc.dll` 中，用户可以通过配置启用它。事实上，这就是我希望从现在开始发布大型 GC 功能的方式——我们首先在独立的 GC（即 `clrgc.dll`）中发布它们，以便大家尝试，然后再将其正式整合到 `coreclr.dll` 中，使其默认启用。

---

#### 段与区域的区别

到目前为止，我们一直在使用段进行操作。段多年来为我们服务良好，但我已经开始注意到，随着用户将更多类型的工作负载放到我们的框架上，段模型存在一些局限性。它们是我们的内存管理基础，因此切换到其他模型是一项重大任务。当我们在接近 .NET 6 时，我认为是时候从段模型进化了，这就是我们团队最近花费大量时间研究的内容。

那么，段和区域之间有哪些关键区别呢？段是较大的内存单位——在服务器 GC 的 64 位系统中，SOH（小型对象堆）上的段大小可能是 1GB、2GB 或 4GB（工作站 GC 的段要小得多，只有 256MB）。而区域是更小的单位，默认情况下每个区域为 4MB。因此，你可能会问：“它们只是更小，为什么这很重要？” 为了回答这个问题，让我们先回顾一下段是如何工作的。

这是当前只有一个段时，SOH 在堆上的样子：

![单段](images/dpad.png)

当有多个段时，它可能看起来像这样：

![多段1](images/dpad-1.png)

或者这样：

![多段2](images/dpad-2.png)

蓝色和黄色区域表示段上的已提交内存（有关代起始点的解释，请参见此视频）。每个段都会跟踪段上已提交的内存，以便我们知道是否需要提交更多内存。并且该段上的所有空闲空间也是已提交的内存。这在使用空闲空间容纳对象时效果很好，因为我们可以直接使用这些内存——它们已经被提交。但想象一种场景：我们在某一代中有空闲空间，比如 gen0，因为某些异步 IO 操作导致我们在 gen0 中降级了一组固定对象，但我们实际上并没有使用这些空闲空间（这可能是因为没有等待足够长的时间来进行下一次 GC，或者我们积累了过多的幸存对象，这意味着 GC 暂停时间会过长）。如果我们能够将这些空闲空间用于其他代，那不是很好吗？同样，LOH 和 gen2 中的空闲空间也是如此——如果你在 gen2 中有一些空闲空间，能够用它们分配一些大对象就很不错了。我们确实会在段的末尾取消提交内存（用浅灰色表示），但这仅限于段末尾，即该段上最后一个存活对象之后的部分。如果你有固定对象阻止 GC 收缩段的末尾，那么我们只能形成空闲空间，而空闲空间始终是已提交的内存。当然，你可能会问：“为什么不直接取消提交段中间的大块空闲空间呢？” 这需要额外的簿记操作来记住段中间哪些部分已被取消提交，以便在需要分配对象时重新提交它们。无论如何，我们现在正逐渐转向区域的概念，即让 GC 单独操作更小的内存块。

在区域模型下，各代看起来更加统一，我们不再有“临时段”这一概念。我们有 gen0 和 gen1 区域，就像我们有 gen2 区域一样。

![区域模型](images/dpad-3.png)

当然，每代中的区域数量可能差异很大，但它们都由这些小的内存单元组成。LOH 的区域更大（是 SOH 区域的 8 倍，因此每个区域为 32MB）。当我们释放一个区域时，我们会将其返回到空闲区域池中，该池中的区域可以被任何代甚至其他堆使用（如果需要）。因此，你不会再看到这样的场景：gen2 或 LOH 中有一些巨大的空闲空间，但它们长时间未被使用（这可能发生在应用程序行为经历不同阶段时，某一阶段可能比另一阶段存活更多的内存，而 GC 没有觉得有必要执行完整的压缩 GC）。

---

#### 区域带来的权衡

在 GC 工作中，我们总是需要做出权衡。通过区域，我们获得了很大的灵活性，但也必须放弃一些东西。使段非常有吸引力的一点是，我们有一个连续的临时范围，因为 gen0 和 gen1 总是位于临时段上，并且总是紧挨着彼此。我们在写屏障中设置卡片时利用了这一点。如果你执行 `obj0.f = obj1`，并且我们检测到 `obj1` 不在临时范围内，则无需设置卡片，因为我们不需要它（只有当 `obj1` 位于比 `obj0` 更年轻的代中时才需要设置卡片，如果 `obj1` 不在临时范围内，则意味着它要么在 gen2 中，要么在 LOH/POH 中，这些都被逻辑地视为第 2 代的一部分（但内部被跟踪为第 3 代和第 4 代，在本文中我将 LOH 和第 3 代互换使用）。这意味着 `obj1` 要么与 `obj0` 同属一代，要么比 `obj0` 更老）。但我们仅在工作站 GC 中进行这种优化，因为服务器 GC 有多个临时范围，我们不想在写屏障代码中比较所有这些范围。在区域模型中，我们可能会无条件地设置卡片（这会使工作站 GC 的暂停时间稍微退步，但保持服务器 GC 的性能不变），或者在写屏障中检查 `obj1` 所属的区域，这比在最优类型的写屏障中检查临时范围更昂贵。不过，区域带来的好处应该足以证明这一点。