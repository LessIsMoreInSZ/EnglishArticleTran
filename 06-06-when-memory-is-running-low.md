<h1>When memory is running low…</h1>
When I say memory I mean physical memory. Let’s assume that you have enough virtual memory space. 
When the physical memory gets low you may start getting OOMs or start paging.

You can experiment and see how you can avoid getting into the low memory situation but sometimes it’s hard to predict and hard to test all scenarios your app can get into. 
For example, if your app handles requests where the amount of memory each request consumes can vary dramatically, 
you may be able to handle 1000 concurrent requests some times before running out of memory but only 800 other times.

First of all, let’s talk about how you know when the memory is low on the machine. 
You can do this by either calling the Win32 API GlobalMemoryStatusEx, or monitor memory performance counters. 
Currently there’s no notification sent by the GC for low memory – one reason being the memory is used not only by the managed heap but also by native heaps/modules loaded/etc.

You can get the process info via either Win32 APIs, perf counters or the managed Process class.

To get the amount of memory that the managed heap is using, the most accurate way is to monitor the “# Total committed bytes” performance counter under .NET CLR Memory.

Something that server throttling code usually does is to monitor the memory usage and when the memory consumption gets high it stops accepting new requests and waits for the current requests to drain. 
When the memory load gets to a reasonably low point it starts accepting new requests again.

But the problem is, if the current requests don’t make any allocations or don’t make enough allocations to trigger a full GC your managed heap doesn’t have a chance to collect dead objects so after the current requests drain the memory consumption does not go down.

What do you do in this case? You can call GC.Collect. This is a legitimate case for calling GC.Collect yourself – but be careful how you call it! There are a few things you should keep in mind when calling GC.Collect:

Don’t call it too often –

you want to set a minimum interval of how often you call it. This would depend on your application. 
Let’s say you have 500MB worth of live objects to collect (meaning the resulting heap size after GC is about 500MB, not counting the garbage), 
perhaps you want to not call GC.Collect more often than every 5 seconds. 
You can determine this by doing a simple test on the type of machines that your app usually runs on and see how long it takes to collect certain number of bytes.
GC happens on its own schedule so you want to account for that. When it’s the time for you to induce a GC a GC may have already happened during the last interval so it’s not necessary for you to induce the GC.
 You can check for this by calling GC.CollectionCount(GC.MaxGeneration) after the interval elapses and if the collection count has already increased from the last time you induced a GC you don’t need to induce another one.
Don’t call it when it’s not productive – if after you call GC.Collect and the memory doesn’t drop by much, you want to call it less often. 
It’s a good measure to check say, if the memory hasn’t dropped by 1% of the total memory you got on the machine. 
If that’s the case perhaps you want to increase the interval by some percentage.

Recently I worked with a team that implemented just that and got good results.

Some people have asked why we don’t just give you a way to limit the amount of memory that a process is allowed to use.
 Imagine if we did, how would you use it? Do you set the amount of memory that your app is allowed to use to 90% of the physical memory? 
 90% sounds good… now, if everyone was to take this suggestion and set their app to use 90% of the memory we are back to square one. 
 If everyone sets it to 50% and if your app is the only active app on the machine you’d be wasting half of the memory.

If your app is running in a very controlled environment, in other words, you are probably using the hosting APIs to gain more control in a managed app, 
there are hosting APIs you can use to limit the amount of memory that managed heap can use. 
You can use IGCHostControl::RequestVirtualMemLimit to deny the request when GC needs to acquire a new segment. 
So you can check for the amount of memory that managed heap uses and if it exceeds the limit you set, you use this API to tell GC to not add new segments. 
The result of this is there will be a GC trigger trying to get back more memory. If after GC is done there’s still no space to satisfy your allocation request you will get an OOM. 
Note that this means you need to be prepared to handle OOMs which can be very difficult or even impossible. 
So to be able to use this API in an appropriate way generally means you’ll need to use other hosting APIs to do things like communicating the memory load to GC (memory load that you defined), 
tracking allocations and terminating tasks or unloading app domains. Refer to the hosting API documentation for details.

BTW, if you really want to limit the memory for a process, you even have a way to do it via Win32 APIs. 
You can create a job object via the CreateJobObject call that you associate with your process, 
and specify the amount of memory the process is allowed to use (or if you have only one process associated you can set the memory limit for the job) via SetInformationJobObject.

Now, if your application simply needs to use a lot more memory than the amount of physical memory, what do you do? 
Currently (meaning, on CLR 2.0 or earlier) if you need to constantly churning through all this data, it would mean you will incur full GCs often. 
If a large portion of the managed heap is stored in the page file and would need to be paged in to do a GC, you will take a pretty serious performance hit. 
It may be worse of the alternative which is storing it as native data and only page in what you need. 
You will still need to go to the page file very often but you don’t need to pay the price of paging in everything to do a GC. 
So right now doing caching in native code (in memory or in a file) could be a better choice, provided that the cache implementation has very low fragmentation. 
Sorry I don’t have a better answer but we are perfectly aware of this problem though and are working hard on it.

