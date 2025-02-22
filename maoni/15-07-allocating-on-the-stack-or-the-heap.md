<h1>Allocating on the stack or the heap?</h1>

A recent discussion prompted me to write this blog entry. The question there was “when should I allocate something on the stack vs on the heap?”. 
I searched around and there are plenty of articles that talk about *what* gets allocated on the stack vs on the heap, but not how you should decide what to allocate on the stack vs heap.

It makes sense to understand the implications of allocating on the stack vs on the heap in order to make the right decisions because as with most perf advice, 
there’s no one rule fits all. Often the right approach is to understand what things mean, then see what consequences they have in your specific context. 
This is meant to help you understand these implications better so you can think how they fit into your scenario.

You already know there are syntactic things that you can do with references that you can’t do with structs like you can’t inherit from a struct so I won’t focus on those here. 
This is about how you make the decisions based on performance.

At the allocation time, in the best case heap allocations can be as fast as stack allocations – they both advance a pointer and clear memory they hand out. 
Of course stack has more chances to be have better locality; and heap allocations can also be more expensive when we use the freelist in gen0, 
but since we only go to the GC for memory every few kbytes, the effect of using the freelist shouldn’t be too bad (If I use 100 cycles as an approximation of reading the freelist 
and 1 cycle to clear 1 byte, 100/8192 = 1.2%).

There is an important point which is for small allocations you save the 2-ptr sized words if you allocate on the stack. If the object is >2k, then the 2-ptr size also is <1%. 
So I would say after 2k the advantage of space saving for stack starts to diminish. Also since struct is embedded, if you are allocating an array of struct, 
the payload is just struct_size * number_of_elements, whereas an array of class, the payload is ptr_size * number_of_elements (because each element is a pointer that points to an instance) 
+ space each class instance takes up.

However, the consequences of stack and heap allocations can be very different –

For stack: we have a very limited amount of virtual address space on the stack. 
It’s not uncommon to see stacks with 256 frames these days which means each frame gets 4k in case of the 1mb stack size. Of course most frames don’t consume nearly 4k.

[Note: I used to mention specific stack sizes here but I’ve since removed it – apparently my memory was incorrect and my understanding of how stack size is set was also insufficient :-). 
A few years ago we made a change to reduce our stack size to be half of the default size but that was only limited to the utility threads we created, 
not to “threads created by managed code” in general. How the stack size is set is frankly unnecessarily complicated – the OS sets one, the linker sets one and we set one 
(and of course users can just call CreateThread and specify a size). And I just did a little experiment – if you build with /platform:x64
 (I am doing all this with the compiler from the 64-bit framework) I see threads have 4mb stack but if you don’t specify it, 
 the resulting binary is still 64-bit but it has a 1mb stack size! So, I am not going to attempt to give the exact sizes of the stacks because I don’t know the exact numbers in all situations.]

Note you can also stackalloc stuff yourself but it can only be used in unsafe code.

Comparison: struct needs to compare memory the struct occupies.

Access: fields in a struct can be enregistered; there’s rarely write barrier cost on assignment; since it’s passed by value, the copy cost can be significant (we have seen such cases).

For heap: there are 3 aspects wrt GC:

1) how much you are allocating compared to the rest. You need to pay at least fixed cost for each GC for suspension/resumption. 
If the amount you are pondering to allocate on the heap (vs on the stack) is small compared to other allocations you are already making, 
then obviously you shouldn’t be concerned with this point.

2) how temporary these objects are – if they are not truly temporary they will be promoted to gen1 and that GC cost will come into play.

3) if you pin, and there are other allocations inbetween, then pinning will come into play.

Comparison: class comparison is in general just a pointer comparison (there are fancier cases which is why I am saying in general).

Access: on assignment there’s likely write barrier cost (in Server GC there’s always write barrier cost).

So, if performance is a concern, these are the things you can watch out for.

https://devblogs.microsoft.com/dotnet/allocating-on-the-stack-or-the-heap/

最近的一次讨论促使我写了这篇博客。问题是：“我应该在栈上分配还是在堆上分配？”。我搜索了一下，发现有很多文章讨论了什么会被分配在栈上，
什么会被分配在堆上，但并没有讨论如何决定在栈上还是堆上分配。

为了做出正确的决策，理解在栈上分配和在堆上分配的影响是很有意义的，因为与大多数性能建议一样，没有一种规则适用于所有情况。
通常，正确的方法是理解这些选择的意义，然后看看它们在你的具体场景中会带来什么后果。本文旨在帮助你更好地理解这些影响，以便你能够思考它们如何适用于你的场景。

你已经知道，引用类型（references）有一些语法特性是结构体（structs）无法做到的，比如你不能从结构体继承，因此我不会在这里讨论这些。本文的重点是基于性能的决策。

分配时的性能
在分配时，堆分配在最佳情况下可以像栈分配一样快 —— 它们都是通过移动指针并清除内存来完成的。当然，栈分配通常具有更好的局部性；
而当我们使用gen0中的空闲列表时，堆分配可能会更昂贵，但由于我们每隔几千字节才会向GC请求内存，使用空闲列表的影响应该不会太大（
如果我假设读取空闲列表需要100个周期，清除1字节需要1个周期，那么100/8192 = 1.2%）。

有一个重要的点是，对于小对象分配，如果你在栈上分配，可以节省2个指针大小的空间。如果对象大于2KB，那么2个指针大小的空间也小于1%。
因此，我认为在2KB之后，栈分配的空间优势开始减弱。此外，由于结构体是嵌入的，如果你分配一个结构体数组，有效负载只是结构体大小 * 元素数量，而如果是类数组，
有效负载是指针大小 * 元素数量（因为每个元素是一个指向实例的指针）加上每个类实例占用的空间。

栈分配和堆分配的后果
然而，栈分配和堆分配的后果可能非常不同 ——

栈分配：
栈的虚拟地址空间非常有限。如今，看到有256个栈帧的情况并不少见，这意味着在1MB的栈大小下，每个帧只能获得4KB的空间。当然，大多数帧不会占用接近4KB的空间。

[注：我曾经在这里提到过具体的栈大小，但后来删除了 —— 显然我的记忆有误，而且我对栈大小设置的理解也不够充分 :-)。几年前我们做了一个更改，将栈大小减少到默认大小的一半，
但这仅限于我们创建的实用线程，而不是“由托管代码创建的线程”一般情况。栈大小的设置实际上非常复杂 —— 操作系统设置一个值，链接器设置一个值，
我们也设置一个值（当然用户也可以直接调用CreateThread并指定大小）。我做了一个小实验 —— 如果你用/platform:x64构建（我使用的是64位框架的编译器），
线程的栈大小为4MB，但如果你不指定它，生成的二进制文件仍然是64位的，但栈大小为1MB！因此，我不会试图给出确切的栈大小，因为我不知道所有情况下的确切数字。]

你也可以使用stackalloc在栈上分配内存，但这只能在不安全代码中使用。

比较：结构体需要比较其占用的内存。

访问：结构体的字段可以被寄存器化；在赋值时很少会有写屏障成本；由于它是按值传递的，复制成本可能很高（我们见过这样的案例）。

堆分配：
有三个与GC相关的方面：

分配量：与其它分配相比，你分配了多少。每次GC至少需要支付暂停/恢复的固定成本。如果你正在考虑在堆上分配（而不是栈上）的量相对于你已经进行的其他分配来说很小，
那么显然你不应该担心这一点。

对象的临时性：如果这些对象不是真正的临时对象，它们将被提升到gen1，GC成本就会增加。

固定（pinning）：如果你固定对象，并且在中间有其他分配，那么固定操作就会产生影响。

比较：类的比较通常只是指针比较（当然也有更复杂的情况，所以我只是说通常）。

访问：在赋值时可能会有写屏障成本（在Server GC中总是有写屏障成本）。

总结
因此，如果性能是一个关注点，你可以注意以下几点：

栈分配适合小对象和临时对象，尤其是当对象的生命周期很短且不需要跨方法调用时。

堆分配适合较大的对象或生命周期较长的对象，尤其是当对象需要跨方法调用或作为引用传递时。

通过理解这些影响，你可以根据具体的场景做出更明智的决策。