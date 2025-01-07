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