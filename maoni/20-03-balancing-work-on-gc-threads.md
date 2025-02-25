<h1>Balancing work on GC threads</h1>

In Server GC, each GC thread will work on its heap in parallel (that’s a simplistic view and is not necessarily true for all phases but on the high level it’s exact the idea of a parallel GC). So that alone means work is already split between GC threads. But because GC work for some stages can only proceed after all threads are done with their last stage (for example, we can’t have any GC thread start with the plan phase until all GC threads are done with the mark phase so we don’t miss objects that should be marked), we want the amount of GC work balanced on each thread as much as possible so the total pause can be shorter, otherwise if one thread is taking a long time to finish such a stage the other threads will be waiting around not doing anything. There are various things we do in order to make the work more balanced. We will continue to do work like this to balance out more.

Balancing allocations

One way to balance the collection work is to balance the allocations. Of course even if you have the exact same amount of allocations per heap the amount of collection work can still be very different, depending on the survival. But it certainly helps. So we equalize the allocation budget at the end of a GC so each heap gets the same allocation budget. This doesn’t mean naturally each heap will get the same amount of allocations but it puts the same upper limit on the amount of allocations each heap can do before the next GC is triggered. The number of allocating threads and the amount of allocation each allocating thread does are of course up to user code. We try to make the allocations on the heap associated with the core that the allocating thread runs on but since we have no control, we need to check if we should balance to other heaps that are the least full and balance to them when appropriate. The “when appropriate” requires some careful tuning heuristics. Currently we take into consideration the core the thread is running on, the NUMA node it runs on, how much allocation budget it has left compared to other heaps and how many allocating threads have been running on the same core. I do think this is a bit unnecessarily complicated so we are doing more work to see if we could simply this.

If we use the GCHeapCount config to specify fewer heaps than cores, it means there will only be that many GC threads and by default they would only run on that many cores. Of course the user threads are free to run on the rest of the cores and the allocations they do will be balanced onto the GC heaps.

Balancing GC work

Most of the current balancing done in GC is focused on marking, simply because marking is usually the phase that takes the longest. If you are going to pick tasks to balance, it makes more sense to balance the longest part that is most prone to being unbalanced – balancing work does not come without a cost.

Marking uses a mark stack which makes it a natural target for working stealing. When a GC thread is done with its own marking, it looks around to see if other threads’ mark stacks are still busy and if so, steal an object to mark. This is complicated by the fact that we implement “partial mark”, which means if an object contains many references we only push a chunk of them onto the mark stack at a time to not overflow the stack. This means the entries on the stack may not be straightforward object addresses. Stealing needs to recognize specific sequences to determine whether it should search for other entries or read the right entry in that sequence to steal. Note that this is only turned on during full blocking GCs as the stealing does have noticeable cost in certain situations.

Performance work is largely driven by user scenarios. And as our framework is used more and more by high performance scenarios, we are always doing work to shorten the pause time. Folks have asked about concurrent compacting GCs and yes, we do have that on our roadmap. But it does not mean we will stop improving our current GC. One of the things we noticed from looking at customer data is when we are doing an ephemeral GC, marking young gen objects pointed to by objects in older generations usually takes the longest time. Recently we implemented working stealing for this in 5.0 by having each GC thread takes a chunk of the older generation to process each time. It atomically increases the chunk index so if another thread is also looking at the same generation it will take the next chunk that hasn’t been taken. The complication here is we might have multiple segments so we need to keep track of the current segment being processed (and its starting index). In the situation when one thread just gets to a segment which has already been processed by other threads, it knows to advance past this segment. Each chunk is guaranteed to only been processed by one thread. Because of this guarantee and the fact that relocating pointers to young gen objects shares the same code path, it means this relocation work is also balanced in the same fashion.

We also do balancing work at the end of the phase so it can balance the imbalance in earlier work happening in the same phase.

There are other kinds of balancing but those are the main ones. More kinds of work can be balanced for STW GCs. We chose to focus more on the mark phase because it’s the most needed. We have not balanced the concurrent work just because it’s more forgiving when you run concurrently. Clearly there’s merit in balancing that too so it’s a matter of getting to it.

Future work

As I mentioned, we are continuing the journey to make things more balanced. Aside from balancing the current tasks more, we are also changing how heaps are organized to make balancing more natural (so the threads aren’t so tightly coupled with heaps). That’s for another blog post.

https://devblogs.microsoft.com/dotnet/balancing-work-on-gc-threads/

在服务器垃圾回收（Server GC）中，每个 GC 线程会并行地处理其对应的堆。这是一个简化的观点，并不适用于所有阶段，但从高层次来看，这正是并行 GC 的核心思想。仅这一点就表明工作已经在多个 GC 线程之间分配了。然而，由于某些阶段的 GC 工作只能在所有线程完成上一个阶段后才能继续（例如，在所有 GC 线程完成标记阶段之前，任何线程都不能开始计划阶段，以免遗漏需要标记的对象），我们希望尽可能平衡每个线程的 GC 工作量，以缩短总暂停时间。否则，如果某个线程花费较长时间完成某个阶段，其他线程就会空闲等待。为了使工作更加平衡，我们做了许多努力，并将继续进行类似的工作来进一步优化。

### 平衡分配

一种平衡收集工作的方法是平衡分配。当然，即使每个堆的分配量完全相同，收集工作的量仍可能因对象存活率的不同而有很大差异。但无论如何，这确实有所帮助。因此，我们在每次 GC 结束时均衡分配预算，使每个堆获得相同的分配预算。这并不意味着每个堆自然会获得相同的分配量，但它为每个堆在触发下一次 GC 之前的分配量设定了相同的上限。分配线程的数量以及每个分配线程的分配量当然由用户代码决定。我们尝试将分配操作集中在与分配线程运行所在的内核相关联的堆上，但由于我们无法控制线程调度，因此需要检查是否应该将分配平衡到其他堆（即最不满的堆）上，并在适当的时候进行平衡。“适当的时候”需要一些细致的启发式调整。目前，我们会考虑线程运行的核心、它所在的 NUMA 节点、与其他堆相比剩余的分配预算，以及在同一核心上运行的分配线程数量。我认为这种方法有些过于复杂，因此我们正在进行更多工作，试图简化这一过程。

如果我们使用 `GCHeapCount` 配置指定的堆数少于核心数，这意味着只会存在那么多 GC 线程，默认情况下它们只会在那些核心上运行。当然，用户线程可以自由运行在其余核心上，而它们的分配操作会被平衡到 GC 堆上。

### 平衡 GC 工作

目前 GC 中所做的大部分平衡工作都集中在标记阶段，因为标记通常是耗时最长的阶段。如果你要选择任务进行平衡，那么最有意义的是平衡最长的部分，这部分最容易出现不平衡——平衡工作本身也是有代价的。

标记使用了一个标记栈，这使得它成为工作窃取（work stealing）的自然目标。当一个 GC 线程完成自己的标记任务后，它会查看其他线程的标记栈是否仍然忙碌，如果是，则窃取一个对象进行标记。这种情况被“部分标记”（partial mark）所复杂化，这意味着如果一个对象包含大量引用，我们不会一次性将所有引用推入标记栈，而是分批推送，以避免栈溢出。这意味着栈上的条目可能不是直接的对象地址。窃取需要识别特定序列，以确定是否应搜索其他条目或读取该序列中的正确条目进行窃取。需要注意的是，这种窃取机制仅在完全阻塞式 GC 中启用，因为在某些情况下窃取会有明显的成本。

性能优化工作主要由用户场景驱动。随着我们的框架越来越多地应用于高性能场景，我们一直在努力缩短暂停时间。许多人询问过并发压缩 GC 的问题，是的，这确实在我们的路线图上。但这并不意味着我们将停止改进当前的 GC。我们从客户数据中注意到的一件事是，在进行短暂代 GC 时，标记由老年代对象指向的年轻代对象通常需要最长时间。最近，我们在 5.0 版本中为此实现了工作窃取：每个 GC 线程每次处理老年代的一部分。它原子性地增加块索引，因此如果另一个线程也在处理同一个老年代，它会获取下一个未被处理的块。这里的复杂之处在于可能存在多个段，因此我们需要跟踪当前正在处理的段及其起始索引。当一个线程到达已被其他线程处理过的段时，它知道跳过这个段。每个块保证只由一个线程处理。由于这种保证以及重定位年轻代对象指针共享相同的代码路径，这意味着重定位工作也以同样的方式得到了平衡。

我们还在阶段结束时进行平衡工作，以便平衡该阶段早期发生的不平衡工作。

还有其他类型的平衡工作，但这些是最主要的。对于 STW GC 来说，可以平衡更多种类的工作。我们之所以更关注标记阶段的平衡，是因为这是最需要的。我们尚未对并发工作进行平衡，因为在并发执行时稍微不平衡是可以接受的。显然，对并发工作进行平衡也有其价值，只是需要安排合适的时间来做。

### 未来工作

正如我提到的，我们仍在继续努力使各方面更加平衡。除了进一步平衡当前的任务外，我们还正在改变堆的组织方式，以使平衡变得更加自然（从而使线程与堆之间的耦合不再如此紧密）。这将是另一篇博客的主题。