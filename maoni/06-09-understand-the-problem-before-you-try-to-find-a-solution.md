<h1>understand-the-problem-before-you-try-to-find-a-solution</h1>
So far I’ve never written a blog entry that gives out philosophical advices on doing performance work. 
But lately I thought perhaps it’s time to write such an entry because I’ve seen enough people who looked really hard at some performance counters (often not correct ones) or some other data and asked tons of questions such as “is this allocation rate too high? It looks too high to me.” or “my gen1 size is too big, right? It seems big…”, before they have enough evidence to even justify such an investigation and questions.

 

Now, if you are just asking questions to satisfy your curious mind, that’s great. 
I am happy to answer questions or point you at documents to read. But for people who are required to investigate performance related issues, 
especially when the deadline is close, my advice is “understand the problem before you try to find a solution”. Determine what to look at based on evidence, 
not based on your lack of knowledge in the area unless you already exhausted areas that you do know about.
 Before you ask questions related to GC, ask yourself if you think GC is actually the problem. 
 If you can’t answer that question it really is not a good use of your time to ask questions related to GC.

 

I’ve seen many cases when something seems to go wrong in a managed application, people immediately suspect GC without any evidence to support that suspicion. 
Then they start to ask questions – often very random – hoping that they would somehow find the solution to fix their problem without understand what the problem is. 
That’s not logical is it? So stop doing it!

 

So how do you know the right problems to solve, I would recommend the following:

 

1) Having knowledge about fundamentals really helps.

 

What are fundamentals? Performance in general comes down to 2 things – memory and CPU. 
Knowing basics of these 2 areas helps a lot in determining which area to look at. Obviously this involves a lot of reading and experimenting. 
I will list some memory fundamentals to get you started:

 

Some fundamentals of memory

 

·          Each process has its own and separated virtual address space; all processes on the same machine share the physical memory (plus the page file if you have one). 
On 32-bit each process has a 2GB user mode virtual address space by default.

 

·          You, as an application author, work with virtual address space – you don’t ever manipulate physical memory directly. 
If you are writing native code usually you use the virtual address space via some kind of win32 heap APIs (crt heap or the process heap or the heap you create) – these heap APIs will allocate and free virtual memory on your behalf; if you are writing managed code, GC is the one who allocates/frees virtual memory on your behalf.

 

·          Virtual address space can get fragmented – in other words, there can be “holes” (free blocks) in the address space. When a VM allocation is requested, 
the VM manager needs to find one free block that’s big enough to satisfy that allocation request – if you only got a few free blocks whose sum is big enough it won’t work. 
This means even though you got 2GB, you don’t necessarily see all 2GB used.

 

·          VM can be in different states – free, reserved and committed. Free is easy. The difference between reserved and committed is what confused people sometimes.
 First of all, you need to recognized that they are different states. Reserved is saying “I want to make this region of memory available for my own use”. 
 After you reserve a block of VM that block can not be used to satisfy other reserve requests. 
 At this point you can not store any of your data in that block of memory yet – to be able to do that you have to commit it, 
 which means you have to back it up with some physical storage so you can store stuff in it. When you are looking at the memory via perf counters and what not, 
 make sure you are looking at the right things. You can get out of memory if you are running out space to reserve, or space to commit.

 

·          If you have a page file (by default you do) you can be using it even if your physical memory pressure is very low. 
What happens is the first time your physical memory pressure gets high and the OS needs to make room in the physical memory to store other data, 
it will back up some data that’s currently in physical memory in the page file. 
And that data will not be paged in until it’s needed so you can get into situations where the physical memory load is very low yet you are observing paging.


 

2) Knowing what your performance requirements are is a must.


 

If you are writing a server application, 
it’s very likely that you want to use all the memory and CPU that’s available because people delicate the machine completely to run your app so why waste resources? 
If you are writing a client application, 
totally different story – you’ll have to know how to cope with other applications running on the same machine. 
There’re no rules such as “you have to make your app use as little memory as possible”.

 

When you’ve decided that there is a problem, dig into it instead of guess what might be wrong. If your app is using too much memory, 
look at who is using the memory. If you’ve decided that the managed heap is using too much memory, look at why.
 Managed heap using too much memory generally means you survive too much in your app. Look at what is holding on to those survivors.

 https://devblogs.microsoft.com/dotnet/understand-the-problem-before-you-try-to-find-a-solution/

 

