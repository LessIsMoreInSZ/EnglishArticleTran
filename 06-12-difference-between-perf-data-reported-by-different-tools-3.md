<h1>Difference Between Perf Data Reported by Different Tools – 3</h1>

Both the !SOS.gchandles command (added in CLR 2.0) and the .NET CLR Memory\# GC Handles counter show you the number of GC handles 
you have in your process.

The # GC Handles counter is one of the rare counters in the .NET CLR Memory category that doesn’t get updated at the end of each GC. 
Rather we update it in the handle table code, for example, when some code in the CLR calls the function to create a GC handle
 (possibly because the user has requested to create a handle via managed code), we increase this counter value by one. 
 r performance reasons we don’t use interlocked operations when we need to increase or decrease this value. 
 This means the value can get changed concurrently by multiple threads. 
 For this reason you should always trust the value returned by the !SOS.gchandles command if you ever doubt the counter value. 
 The SOS command is accurate because we always walk the handle table when you issue the command so it returns the # of handles truthfully.

 https://devblogs.microsoft.com/dotnet/difference-between-perf-data-reported-by-different-tools-3/

