<h1>Finalization implementation details</h1>

Years ago I wrote a document on making finalization scanning concurrent. At the time there was an internal team that was using finalization as a way to resurrect objects and putting them back in their cache. While we’ve always advised to folks that finalization was for releasing native resources I couldn’t fault this team for using it the way they did. And of course finalization scanning was taking quite some time because well, there were so many finalizable objects to scan so I wanted to make this part concurrent. As part of the on-going latency reduction effort we’ve finally put this on the agenda. Of course between the time I wrote that doc and now, things have changed quite a bit and there are new considerations I have to make so I’m still thinking about the design. While I’m thinking about it I thought it would be interesting to explain some of the finalization implementation details so that’s what I’ll do in this blog entry. I’m in the process of working on a design doc for concurrent finalization scanning that I will be checking into the coreclr repo.

First of all I wanted to clarify some terminology. I’ve seen some confusing terms in various articles and books. So instead of using something that’s interpreted differently by different people I will just use the terms used in the GC code.

Internally we have a CFinalize class that manages the finalization. The only consumers of this class are the GC itself and of course the finalizer thread. And there’s only one finalizer thread (over the years there were talks about having multiple finalizer threads but there wasn’t enough justification; I have heard that there’s usage that actually depends on the fact there’s only one finalizer thread. While we have no plans to make this multiple I would strong suggest against doing that). The finalizer thread runs at THREAD_PRIORITY_HIGHEST. And this is an important detail as it will have performance implications that I will mention below.

The CFinalize class has an array which we use to track all finalizable objects. And we divide this array into a few parts that I will call segs (as in segments, not to be confused with the GC heap segments). The array looks like this – from lower addresses to higher addresses:

Gen2 seg | Gen1 seg | Gen0 seg | Critical Finalizer seg | Finalizer seg | Free seg

(I’m just calling each portion a “seg” so it’s easier to refer to them)

The genX segs are to record finalizable objects that are still live, ie, either in the generation that GC hasn’t collected so all objects in those generations are live by definition, or is held live by user code. When GC discovers an object to be dead, it then promotes the object which means the object is now tracked by either the Critical Finalizer seg or the Finalizer seg which I will refer to as CF/F segs (this is what’s usually referred to as f-reachable).

When the finalizer thread actually runs finalizers they would pick them off from the CF/F segs. And the ones whose finalizers have been run are now tracked in the Free seg (although saying tracked can be misleading as currently we just treat this seg as “we no longer care about the objects in this seg” and the seg is really just to indicate the free slots we have in the array).

Within GC, each heap has its own CFinalize instance which is called finalize_queue. For lifetime tracking, GC does the following with finalize_queue:

The CF/F segs are considered as a source of roots so they will be scanned to indicate which objects should be live. This is what CFinalize::GcScanRoots does. Because the finalizer thread runs at high priority, it’s very likely that there’s nothing here to scan because when we were done with the GC that discovered these objects and moved them to these segs, the finalizer thread was allowed to run and would’ve quickly finished running the finalizers (unless of course the finalizers were blocked for some reason – so there you go, another reason why it’s bad to have your finalizers block which means GC will need to scan the CF/F segs again).
Of course there are other sources of roots like the stack or GC handles. After we are done marking all the objects held live, directly or indirectly by all those roots, we now have the full knowledge of object lifetime.

Then we scan the genX segs to see which objects tracked there are dead, in other words, not promoted (!IsPromoted(obj)), and for those objects we need to promote them so the finalizers can run. So these newly promoted objects are promoted due to finalization. This is what CFinalize::ScanForFinalization does. There are exceptions to this – if it’s a WeakReference or `WeakReference<T>`, we just run what the finalizer is supposed to do (remember finalizers are written in managed code and run on the finalizer thread – we wouldn’t be running the finalizer the normal way when the EE is suspended) and *not* promote those objects ’cause now we have no reason to. This is a nice perf shortcut. The other exception is when the finalizer is suppressed.
There’s another relevant thing to mention which is the short and long weak handles. These are the handle types we use in the runtime. They are exposed as GCHandleType.Weak and GCHandleType.WeakTrackResurrection respectively. Before we do ScanForFinalization, we null the target of short weak handles if the target object is not promoted. After ScanForFinalization, we do that for long weak handles. So what distinguishes these 2 handle types is the promotion due to finalization.

https://devblogs.microsoft.com/dotnet/finalization-implementation-details/几年前，我写过一篇关于使终结扫描（finalization scanning）并发化的文档。当时有一个内部团队使用终结机制来复活对象并将它们放回缓存中。虽然我们一直建议大家将终结用于释放本机资源，但我不能责怪这个团队以他们的方式使用它。当然，终结扫描花费了相当长的时间，因为需要扫描的可终结对象太多了，所以我希望让这一部分实现并发化。作为正在进行的延迟降低工作的一部分，我们终于把这个任务提上了日程。然而，从我写那篇文档到现在，事情已经发生了很大的变化，因此我需要考虑一些新的因素，目前仍在思考设计。在思考的过程中，我认为解释一些终结实现的细节会很有趣，这就是我将在本文中要做的事情。我正在撰写一份关于并发终结扫描的设计文档，并将其提交到 coreclr 仓库。

首先，我想澄清一些术语。我在各种文章和书籍中看到过一些令人困惑的术语。为了避免不同的人对某些术语有不同的理解，我将直接使用 GC 代码中使用的术语。

在内部，我们有一个名为 CFinalize 的类，用于管理终结过程。这个类的唯一消费者是 GC 本身以及终结线程。而且只有一个终结线程（多年来有过关于引入多个终结线程的讨论，但没有足够的理由支持这样做；我听说有些用例实际上依赖于只有一个终结线程的事实。虽然我们目前没有计划将其改为多个线程，但我强烈建议不要依赖这一点）。终结线程运行在 THREAD_PRIORITY_HIGHEST 优先级下。这是一个重要的细节，因为它会对性能产生影响，我会在下面提到。

CFinalize 类维护一个数组，用于跟踪所有可终结的对象。我们将这个数组分为几个部分，我称之为“segs”（段，不要与 GC 堆段混淆）。数组的结构如下（从低地址到高地址）：

深色版本
Gen2 seg | Gen1 seg | Gen0 seg | Critical Finalizer seg | Finalizer seg | Free seg
（我只是将每个部分称为“seg”，以便更容易引用它们）

genX segs 用于记录仍然存活的可终结对象，即要么处于尚未被 GC 收集的代中（因此这些代中的所有对象按定义都是存活的），要么被用户代码保持存活。当 GC 发现某个对象已死亡时，它会提升该对象，这意味着该对象现在由 Critical Finalizer seg 或 Finalizer seg 跟踪（这通常被称为 f-reachable）。

当终结线程实际运行终结器时，它会从 CF/F segs 中取出这些对象。那些已经运行过终结器的对象现在被跟踪在 Free seg 中（尽管说“跟踪”可能会让人误解，因为目前我们只是将这个段视为“我们不再关心该段中的对象”，这个段实际上只是用来表示数组中可用的空闲槽位）。

在 GC 内部，每个堆都有自己的 CFinalize 实例，称为 finalize_queue。对于生命周期跟踪，GC 对 finalize_queue 执行以下操作：

CF/F segs 被视为根的来源之一，因此它们会被扫描以确定哪些对象应该存活。这是 CFinalize::GcScanRoots 所做的工作。由于终结线程运行在高优先级下，很可能这里没有什么需要扫描的内容，因为在我们完成发现这些对象并将其移动到这些段的 GC 后，终结线程被允许运行，并且会快速完成运行终结器（当然，除非终结器因某种原因被阻塞——所以这也是为什么终结器阻塞是不好的另一个原因，这意味着 GC 可能需要再次扫描 CF/F segs）。

当然还有其他根的来源，例如栈或 GC 句柄。在我们完成标记所有由这些根直接或间接保持存活的对象后，我们现在对对象的生命周期有了完整的了解。

然后我们扫描 genX segs，查看其中哪些对象已死亡（即未被提升，!IsPromoted(obj)），对于这些对象，我们需要提升它们以便终结器可以运行。因此，这些新提升的对象是由于终结而被提升的。这是 CFinalize::ScanForFinalization 所做的工作。有一些例外情况：如果是一个弱引用（WeakReference 或 WeakReference<T>），我们会直接运行终结器应该执行的操作（记住终结器是用托管代码编写的，并在终结线程上运行——当 EE 暂停时，我们不会以正常方式运行终结器），并且 不 提升这些对象，因为我们现在已经没有理由保留它们。这是一个很好的性能优化。另一种例外情况是终结被抑制。

还有一个相关的事情值得一提，那就是短弱句柄和长弱句柄。这些是我们运行时中使用的句柄类型，分别暴露为 GCHandleType.Weak 和 GCHandleType.WeakTrackResurrection。在执行 ScanForFinalization 之前，如果目标对象未被提升，我们会将短弱句柄的目标置为空。在 ScanForFinalization 之后，我们会对长弱句柄做同样的事情。因此，区分这两种句柄类型的特征是由于终结而导致的提升。