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