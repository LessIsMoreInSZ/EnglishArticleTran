<h1>Job support in the GC</h1>

I’ve noticed that more and more of our customers started to use job objects to restrict some form of resources for their processes such as CPU and memory. 
So I wanted to talk about the kind of support the GC offers to help with your managed processes running in a job object.

The first thing to notice is that a managed process actually just gets the benefits of a job object without GC doing anything since, well, 
your process just looks like a normal process to the OS. So without doing anything special in the GC, you already get the following benefits:

if you limit the # of CPUs in your job object, eg, you have 16 CPUs but you run your process in a job object that allows 8, GC will think you only have 8 CPUs which means if you are using Server GC, you get 8 heaps instead of 16.

if you limit the memory usage, ie, commit size and working set, you also get the corresponding benefits in your managed process, 
eg, if you have total 10GB commit size but specified 5GB on your job object, when your process is trying to commit more than 5GB (this includes the commit for your managed heap), it will fail.

However, there’s one scenario that’s not covered. GC becomes more aggressive when it detects that the physical memory load is high (90+%). But the OS simply does not reflect “physical memory load” in a job object, ie, if you specify your working set limit to 2GB on a machine with 8GB and you are using 1.5GB physical memory, GlobalMemoryStatusEx will return 1.5/8=19% for dwMemoryLoad instead of 1.5/2=75%.

So I needed to make some changes in the GC to account for this. This went into coreclr a couple of months ago. This worked out well for customers who previously were getting OOMs in their job objects.

https://devblogs.microsoft.com/dotnet/job-support-in-the-gc/