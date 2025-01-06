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
 

