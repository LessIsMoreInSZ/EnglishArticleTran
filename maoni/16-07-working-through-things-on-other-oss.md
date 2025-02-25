<h1>working-through-things-on-other-oss</h1>

We just shipped CoreCLR 1.0. That was a significant milestone for us – now we officially run on non Windows OSs and that’s very exciting. 
Folks who told me that they really missed using C# or wanted to use C# but couldn’t because they were not using Windows can now do so. Yay!

For GC it would seem like there shouldn’t’ve been much work to get it to run on non Windows because if you look at the GC code, it looks pretty portable – we don’t use any fancy 
language features in GC (I like this aspect a lot and will keep it that way); there’s a bunch of arithmetic operations on simple types, 
some locks (mostly implemented with low level stuff like Interlocked operations) and there’re a few OS calls (we have to goto the OS at some point). 
It turned out the last category actually created quite a bit of work for us which is what this blog entry is about.

GC uses a feature from the Windows VMM called the write watch. You can allocate pages with write watch specified (VirtualAlloc and specify MEM_WRITE_WATCH) and 
when modifications are made to those pages you can call an API to get back the addresses of the ones that have been modified. 
We use this to track modifications made to the pages for the GC heap while a background GC is in progress. 
We’ve used this feature since V1.0 and have hit 2 functional bugs in the 20+ years (it’s completely rare that we encounter bugs in the OS). 
I’ve needed to do things in the GC to compensate for its perf restrictions but that wasn’t too difficult.

So I looked around on Linux (I am just gonna use Linux from here on instead of other OSs for simplicity) and found this soft-dirty bit 
on the PTE which looked awfully like it should be exactly for this purpose – it tells you when a page was written to. 
But it’s very awkward to use – you need to manipulate some files to read and clear these bits yourself 
(which also made me very suspicious of its reliability). On Windows you just call an API to get the dirtied pages and clear the bits atomically which is easy and reliable. 
And to make things more difficult, it required admin capability to read for some recent versions of Linux till someone figured out that shouldn’t be the case. 
And it’s had a few bugs even though it hasn’t been that long since it was added. All factors considered, this didn’t seem like a promising approach so we abandoned it.

Then we looked into just simulating the implementation ourselves in user mode. 
We make pages read only and when they are modified we make them read write in the page fault handler. 
Obviously there’s perf concerns with this approach but this happens while the user threads are running so while it does regress throughput, 
it doesn’t make GC pauses worse – in fact it can make GC pauses shorter because we can make retrieving written pages much faster 
by making the addresses of written pages readily available when GC needs them. So it’s a tradeoff. 
Then we hit the infamous low limit of memory mappings that we then discovered other frameworks that work on Linux have also hit. 
If you have 2 adjacent pages are both RO, and now you make the 2nd page RW, 
it will create another page mapping for the 2nd page and make the original page mapping only for the 1st page (the equivalent of VADs on Windows). 
So if you are so unlucky and all your adjacent pages have different attributes, 
you can only address 256MB of virtual memory in your process which is a small amount of VA. 
With our tests that had 1+ GB heaps we easily hit this limit. In order to change this limit (which is recommended by other people who hit it) you need to be su. 
So, we abandoned this approach too.

What we ended up doing was modifying our write barrier code to record the modified pages so when you do obj.referenceField = y; the page obj.x is on is recorded. 
This obviously increases the write barrier cost but the tradeoff is it only records the pages that GC cares about instead of pages modified by anything 
(eg, if you do obj.integerField = 8; GC does not care about this modification). In general, 
I want to reserve write barrier changes for things that really need it (otherwise we just keep increasing write barrier which affects everyone) 
but this deserved it considering the situation we were in.

We also needed to deal with the OOM situations on Linux (to folks who are used to Linux this probably sounds pretty familiar). 
Aside from the policy aspect (eg, by default when the oom killer kicks, what the overcommit_ratio is set to), 
we also hit bugs that were very surprising to me. When we got OOM when trying to commit pages out of a range of pages that we already reserved (on Linux this is mmap with PROT_NONE), it actually unreserved the whole range. There are other things like if we use cgroups to limit memory, with some OOM settings it simply hangs when you’ve allocated too much memory. We are still in the process of understanding the OOM behavior on Linux in order to get to a more stable place.

https://devblogs.microsoft.com/dotnet/working-through-things-on-other-oss/

我们刚刚发布了 CoreCLR 1.0。这对我们来说是一个重要的里程碑——现在我们正式在非 Windows 操作系统上运行，这非常令人兴奋。
那些告诉我他们非常怀念使用 C# 或者想使用 C# 但因为不使用 Windows 而无法使用的人，现在可以这么做了。耶！

对于垃圾回收器（GC）来说，似乎不应该有太多工作需要做才能让它运行在非 Windows 系统上，因为如果你看 GC 的代码，
它看起来非常可移植——我们在 GC 中没有使用任何花哨的语言特性（我非常喜欢这一点，并会继续保持）；代码中有一些对简单类型的算术操作，
一些锁（主要通过像 Interlocked 操作这样的底层机制实现），还有一些操作系统调用（我们必须在某些时候调用操作系统）。
事实证明，最后一类操作实际上给我们带来了相当多的工作，这也是这篇博客要讨论的内容。

GC 使用了 Windows 虚拟内存管理器（VMM）的一个功能，称为写监视（write watch）。你可以分配带有写监视标志的页面（通过 VirtualAlloc 并指定 MEM_WRITE_WATCH），
当这些页面被修改时，你可以调用一个 API 来获取被修改的页面的地址。
我们使用这个功能来跟踪在后台 GC 进行期间对 GC 堆页面的修改。
我们从 V1.0 开始就使用了这个功能，在 20 多年中只遇到了 2 个功能性问题（在操作系统中遇到问题的情况非常罕见）。
我需要在 GC 中做一些事情来弥补其性能限制，但这并不太难。

于是我在 Linux 上看了看（为了简单起见，我将从这里开始使用 Linux 而不是其他操作系统），发现了 PTE 上的软脏位（soft-dirty bit），
它看起来非常像是专门为这个目的设计的——它会告诉你一个页面何时被写入。
但它的使用非常麻烦——你需要操作一些文件来读取和清除这些位（这也让我对其可靠性产生了怀疑）。在 Windows 上，你只需调用一个 API 来获取被修改的页面并原子地清除这些位，
这既简单又可靠。
更糟糕的是，在某些较新的 Linux 版本中，读取这些位需要管理员权限，直到有人意识到这不应该是一个限制。
而且，尽管这个功能添加的时间不长，但它已经有一些 bug。综合考虑，这种方法看起来并不太有前途，所以我们放弃了它。

然后我们研究了如何在用户模式下自己模拟实现这个功能。
我们将页面设置为只读，当它们被修改时，我们在页面错误处理程序中将它们设置为可写。
显然，这种方法存在性能问题，但这发生在用户线程运行时，因此虽然它会降低吞吐量，但不会使 GC 暂停时间变长——事实上，它可以使 GC 暂停时间更短，
因为我们可以通过使被修改页面的地址在 GC 需要时立即可用，从而更快地获取被修改的页面。所以这是一个权衡。
然后我们遇到了臭名昭著的内存映射低限制问题，后来我们发现其他在 Linux 上运行的框架也遇到了这个问题。
如果你有两个相邻的页面都是只读的，现在你将第二个页面设置为可写，它会为第二个页面创建另一个页面映射，并使原始页面映射仅适用于第一个页面（类似于 Windows 上的 VAD）。
所以如果你运气不好，所有相邻的页面都有不同的属性，你只能在你的进程中寻址 256MB 的虚拟内存，这是一个很小的虚拟地址空间。
在我们的测试中，堆大小超过 1GB，我们很容易就达到了这个限制。为了改变这个限制（其他遇到这个问题的人也推荐这样做），你需要有超级用户权限。
所以，我们也放弃了这个方法。

最终我们做的是修改我们的写屏障（write barrier）代码，以记录被修改的页面，所以当你执行 obj.referenceField = y; 时，obj.x 所在的页面会被记录下来。
这显然增加了写屏障的成本，但好处是它只记录 GC 关心的页面，而不是被任何东西修改的页面（例如，如果你执行 obj.integerField = 8;，GC 并不关心这个修改）。
一般来说，我希望将写屏障的修改保留给真正需要它的地方（否则我们会不断增加写屏障的成本，影响所有人），但考虑到我们所处的情况，这是值得的。

我们还需要处理 Linux 上的 OOM（内存不足）情况（对于习惯使用 Linux 的人来说，这可能听起来很熟悉）。
除了策略方面的问题（例如，默认情况下当 OOM killer 启动时，overcommit_ratio 的设置是什么），我们还遇到了一些让我非常惊讶的 bug。
当我们尝试从已经保留的页面范围中提交页面时（在 Linux 上这是通过 mmap 使用 PROT_NONE 实现的），如果遇到 OOM，它实际上会取消保留整个范围。
还有一些其他问题，比如如果我们使用 cgroups 来限制内存，在某些 OOM 设置下，当你分配了太多内存时，它就会直接挂起。
我们仍在研究 Linux 上的 OOM 行为，以便达到一个更稳定的状态。