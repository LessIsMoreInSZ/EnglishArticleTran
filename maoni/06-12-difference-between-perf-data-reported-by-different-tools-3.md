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

两个SOS。gchandles命令（在CLR 2.0中添加）和.NET CLR Memory\# GC Handles计数器显示GC句柄的数量
在你的过程中。

# GC Handles计数器是。net CLR内存类别中少见的不会在每次GC结束时更新的计数器之一。
相反，我们在句柄表代码中更新它，例如，当CLR中的某些代码调用该函数来创建GC句柄时
（可能是因为用户请求通过托管代码创建句柄），我们将该计数器的值增加1。
由于性能原因，当我们需要增加或减少这个值时，我们不使用联锁操作。
这意味着该值可以被多个线程并发地更改。
因此，您应该始终信任！SOS返回的值。如果您怀疑计数器的值，请使用Gchandles命令。
SOS命令是准确的，因为我们总是在发出命令时遍历句柄表，所以它会真实地返回句柄的个数。