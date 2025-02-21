<h1>Finalization Uncovered</h1>

I’ve talked about finalization before but based on seeing questions related to it it appears that it deserves some clarification.

 

First of all, finalization is a mechanism we provide in the CLR wheras Dispose is a programming pattern. 
See Clearing up some confusion over finalization and other areas in GC for an explanation why we provide finalization. 
Inside of the GC, it’s completely not aware of Dispose. People often call GC.SuppressFinalize in their Dispose implementation but that’s just a choice they make when they write code.
 I will explain exactly what GC.SuppressFinalize does in a bit. Oh and I am not the owner of the “Dispose pattern” J

 

So what happens when you allocate an object with a finalizer? GC’s allocator will get called and it’s told this object is finalizable. 
So if GC can successfully allocate this object it will then record that this is a finalizable object. 
GC maintains a list to record finalizable objects so a new object will be in the gen0 part of that list. Recording just means writing the object address X to an entry in the gen0 part.

 

When GC promotes a finalizable object to another generation, it’ll move its address to the part of the list for that generation.
 Of course when the object is compacted we also need to update the entry in the finalize list with the new address.

 

When GC finishes marking objects, ie, it has determined which ones should be live, it will look at the list for the generation it’s collecting see if those objects are dead. 
For the dead ones it will then promote that object and move the address to the part of the list that’s for “ready for finalization” objects. 
If any of such objects are found, GC will signal to the finalizer thread that there’s work to do.

 

When the managed threads are restarted after GC is done, since the finalizer thread is also a managed thread, it also gets restarted and starts to do its work – running finalizers. 
It does this by asking for the entries in the “ready for finalization” part of the list. Those entries are removed from the list as the objects’ finalizers are run.

 

If you do a !finalizequeue you will see output like this:




0:015> !finalizequeue

SyncBlocks to be cleaned up: 0

MTA Interfaces to be released: 0

STA Interfaces to be released: 0

———————————-

——————————

Heap 0

generation 0 has 971 finalizable objects (000000000e31b958->000000000e31d7b0)

generation 1 has 346 finalizable objects (000000000e31ae88->000000000e31b958)

generation 2 has 139 finalizable objects (000000000e31aa30->000000000e31ae88)

Ready for finalization 0 objects (000000000e31d7b0->000000000e31d7b0)

——————————

Heap 1

generation 0 has 2686 finalizable objects (000000000d41aa10->000000000d41fe00)

generation 1 has 473 finalizable objects (000000000d419b48->000000000d41aa10)

generation 2 has 129 finalizable objects (000000000d419740->000000000d419b48)

Ready for finalization 0 objects (000000000d41fe00->000000000d41fe00)

——————————

Heap 2

generation 0 has 319 finalizable objects (000000000d298298->000000000d298c90)

generation 1 has 302 finalizable objects (000000000d297928->000000000d298298)

generation 2 has 241 finalizable objects (000000000d2971a0->000000000d297928)

Ready for finalization 0 objects (000000000d298c90->000000000d298c90)

——————————

Heap 3

generation 0 has 147 finalizable objects (000000000c982998->000000000c982e30)

generation 1 has 432 finalizable objects (000000000c981c18->000000000c982998)

generation 2 has 147 finalizable objects (000000000c981780->000000000c981c18)

Ready for finalization 0 objects (000000000c982e30->000000000c982e30) 

 

As you can see, there are “Finalizable” objects and “Ready for finalization” objects, as we talked about above.

 

It’s very common to see 0 “Ready for finalization” objects. Why? Because the finalizer thread runs at THREAD_PRIORITY_HIGHEST.
 As soon as managed threads are resumed it’s the first to run unless you have other threads of the same or higher priority. 
 Finalizers usually do very little work so it shouldn’t be long before the finalizers are run.
  If there’s another GC happening (other threads can still run and allocate if you have more than 1 CPU),
   if there are still “Ready for finalization” entries, we need to make sure those objects are promoted.

 

So what happens when you call GC.SuppressFinalize on an object? It just sets a bit for this object that tells the GC to not do the operations it would do for a finalizable object it finds dead,
 in other words, when GC scans the finalizable objects when it sees that bit is set, 
 it will treat it as if it was not a finalizable object
  (so the object will be considered completely dead at this point and we will no longer remember it in our “Finalizable” portion of the list).

 

When you call GC.ReRegisterForFinalize we will check to see if that bit is set, if it’s we simply clear it. Otherwise we will ask GC to insert it in the “Finalizable” portion of the list.

https://devblogs.microsoft.com/dotnet/finalization-uncovered/
 

 关于终结（Finalization）的澄清
我之前已经讨论过终结（Finalization），但根据看到的相关问题，似乎有必要对其进行一些澄清。

终结与Dispose的区别
首先，终结是CLR提供的一种机制，而Dispose是一种编程模式。关于为什么我们提供终结机制，可以参考这篇文章的解释。在GC内部，它完全不知道Dispose的存在。人们通常会在Dispose实现中调用GC.SuppressFinalize，但这只是他们在编写代码时做出的选择。稍后我会详细解释GC.SuppressFinalize的作用。另外，我并不是“Dispose模式”的所有者 😊。

分配具有终结器的对象时会发生什么？
当你分配一个具有终结器的对象时，GC的分配器会被调用，并被告知该对象是可终结的。如果GC能够成功分配该对象，它会记录这是一个可终结对象。GC维护一个列表来记录可终结对象，因此新对象会被记录在列表的gen0部分。记录仅仅意味着将对象地址X写入gen0部分的条目中。

当GC将一个可终结对象提升到另一个代时，它会将其地址移动到列表中该代的部分。当然，当对象被压缩时，我们还需要更新终结列表中的条目以反映新地址。

GC如何确定哪些对象需要终结？
当GC完成标记对象（即确定哪些对象应该存活）后，它会查看正在收集的代中的列表，看看哪些对象已经死亡。对于死亡的对象，GC会将其提升，并将其地址移动到“准备终结”部分的列表中。如果发现任何此类对象，GC会向终结器线程发出信号，告诉它有工作需要完成。

当GC完成后，托管线程被重新启动，由于终结器线程也是一个托管线程，它也会被重新启动并开始执行其工作——运行终结器。它通过请求“准备终结”部分的列表条目来完成这项工作。当对象的终结器运行时，这些条目会从列表中移除。

使用!finalizequeue查看终结队列
如果你运行!finalizequeue，你会看到类似以下的输出：

复制
0:015> !finalizequeue
SyncBlocks to be cleaned up: 0
MTA Interfaces to be released: 0
STA Interfaces to be released: 0
———————————-
Heap 0
generation 0 has 971 finalizable objects (000000000e31b958->000000000e31d7b0)
generation 1 has 346 finalizable objects (000000000e31ae88->000000000e31b958)
generation 2 has 139 finalizable objects (000000000e31aa30->000000000e31ae88)
Ready for finalization 0 objects (000000000e31d7b0->000000000e31d7b0)
Heap 1
generation 0 has 2686 finalizable objects (000000000d41aa10->000000000d41fe00)
generation 1 has 473 finalizable objects (000000000d419b48->000000000d41aa10)
generation 2 has 129 finalizable objects (000000000d419740->000000000d419b48)
Ready for finalization 0 objects (000000000d41fe00->000000000d41fe00)
Heap 2
generation 0 has 319 finalizable objects (000000000d298298->000000000d298c90)
generation 1 has 302 finalizable objects (000000000d297928->000000000d298298)
generation 2 has 241 finalizable objects (000000000d2971a0->000000000d297928)
Ready for finalization 0 objects (000000000d298c90->000000000d298c90)
Heap 3
generation 0 has 147 finalizable objects (000000000c982998->000000000c982e30)
generation 1 has 432 finalizable objects (000000000c981c18->000000000c982998)
generation 2 has 147 finalizable objects (000000000c981780->000000000c981c18)
Ready for finalization 0 objects (000000000c982e30->000000000c982e30)
如你所见，输出中分为“可终结对象”和“准备终结对象”，正如我们上面讨论的那样。

为什么“准备终结对象”通常为0？
通常你会看到“准备终结对象”为0。为什么呢？因为终结器线程以THREAD_PRIORITY_HIGHEST（最高优先级）运行。一旦托管线程被恢复，终结器线程会首先运行（除非你有其他相同或更高优先级的线程）。终结器通常只做很少的工作，因此终结器应该很快就会被运行。如果另一个GC正在发生（如果你有多个CPU，其他线程仍然可以运行和分配），如果仍有“准备终结”条目，我们需要确保这些对象被提升。

GC.SuppressFinalize的作用
当你对一个对象调用GC.SuppressFinalize时，它会为该对象设置一个标志位，告诉GC不要对它执行终结操作。换句话说，当GC扫描可终结对象时，如果发现该标志位被设置，它会将该对象视为非可终结对象（因此该对象将被视为完全死亡，我们不再在列表的“可终结”部分中记住它）。

GC.ReRegisterForFinalize的作用
当你调用GC.ReRegisterForFinalize时，我们会检查该标志位是否被设置。如果被设置，我们只需清除它。否则，我们会要求GC将其插入列表的“可终结”部分。

总结
终结是CLR提供的一种机制，用于在对象被回收之前执行清理操作。它与Dispose模式不同，后者是一种编程模式。通过GC.SuppressFinalize和GC.ReRegisterForFinalize，你可以控制对象的终结行为。理解这些机制有助于你更好地管理内存和资源。

