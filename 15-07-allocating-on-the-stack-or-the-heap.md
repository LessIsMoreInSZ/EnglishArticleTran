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