<h1>Difference Between Perf Data Reported by Different Tools – 4</h1>
.NET CLR Memory\% Time in GC counter and !runaway on thread(s) doing GC.

The 2 common ways people use to look at the time spent in GC are the % Time in GC performance counter under .NET CLR Memory,
 and the CPU time displayed by the !runaway debugger command in cdb/windbg. What do they mean exactly? % Time in GC is calculated like this:

When the nthGC starts (ie, after the managed threads are suspended and before the GC work starts), we record the timestamp at that time. 
Let’s call this TA(n);

When nth GC ends (ie, after the GC work is done and before we resume the managed threads), we record another timestamp. Let’s call this TB(n);

So Time spent in this GC is TB(n) – TA(n). And the time since the last GC ended is TB(n) – TB(n-1). 
So % Time in GC is (TB(n) – TA(n)) / (TB(n) – TB(n-1)).

Since we only record the timestamps we don’t actually discount the time when the thread was switched out – for example, 
if you are on a single proc machine and another process has a thread of the same priority (as the thread that’s doing GC) 
that’s also ready to run it may take away some time from the thread that’s doing the GC. None the less it’s a good approximation.

One common scenario where it’s not a good approximation is when paging occurs. 
In this case you will see a very high % Time in GC but really the time that’s actually spent doing GC work is low ‘cause most of the time is spent doing IO. 
To verify if you hit this case you can look at the Memory\Pages/sec to see how much it’s paging.

!runaway is more accurate in the sense that it does record the actual time spent on the threads. 
However I did observe 2 common mistakes when using !runaway to look at time spent in GC.

1) mistake “time spent on the GC thread(s)” as “time spent in GC”.

Let’s take Server GC as an example. When a GC is needed the GC threads first perform the suspension work. 
Obviously this takes time. Sometimes it can take quite a bit time if you have many threads or some threads get stuck and it takes a long time to suspend them.

2) using the current !runaway output to judge how much time GC has taken without taking into count that some user threads have died. 
Besides looking at the User/Kernel Mode time you may want to also look at Elapsed Time. The following output is from one of the issues I looked at:

Elapsed Time

  Thread       Time

   0:2094      0 days 11:57:10.406

   1:2098      0 days 11:57:10.152

   2:2044      0 days 11:57:09.898

   3:27cc      0 days 11:57:09.882

   4:20c0      0 days 11:57:09.597

  13:810       0 days 11:57:09.565

  12:21d4      0 days 11:57:09.565

  11:2308      0 days 11:57:09.565

  10:1d24      0 days 11:57:09.565

   9:23e0      0 days 11:57:09.565

   8:24d0      0 days 11:57:09.565

   7:2168      0 days 11:57:09.565

   6:2134      0 days 11:57:09.565

   5:2124      0 days 11:57:09.565

  14:570       0 days 11:57:08.582

  15:10fc      0 days 11:57:00.000

  16:1ac4      0 days 11:56:58.256

  17:1900      0 days 11:56:58.176

  18:1624      0 days 11:56:57.494

  19:11f8      0 days 10:35:01.881

  21:1620      0 days 0:41:45.017

  22:1cc8      0 days 0:37:30.496

  23:2de0      0 days 0:29:01.820

  24:2e10      0 days 0:29:01.711

  25:2d88      0 days 0:22:26.988

  26:1668      0 days 0:19:26.175

  27:2cb0      0 days 0:16:16.814

  29:2d1c      0 days 0:11:53.779

  28:1de8      0 days 0:11:53.779

  30:2ed4      0 days 0:11:35.466

  32:2030      0 days 0:11:22.544

  31:2084      0 days 0:11:22.544

  33:12b0      0 days 0:07:53.510

  35:28e4      0 days 0:03:13.801

  34:2654      0 days 0:03:13.801

  36:2d68      0 days 0:02:31.988

 

The orange lines are GC threads. They were created about 12 hours ago. The green lines are user threads and they were created 2 mins to 40mins ago. So naturally at the current state these user threads couldn’t’ve spent much time.
橙色的线是GC线程。它们是12小时前创建的。绿线是用户线程，它们是在2分钟到40分钟前创建的。因此，在当前状态下，这些用户线程自然不会花费太多时间。

. net CLR内存\%时间在GC计数器中，并且在执行GC的线程上失控。

人们用来查看GC所花费时间的两种常用方法是：.NET CLR内存下的GC性能计数器% time；
和cdb/windbg中！runaway debugger命令显示的CPU时间。它们到底是什么意思？GC中的%时间是这样计算的：

当nthGC启动时（即，在托管线程挂起之后和GC工作开始之前），我们记录当时的时间戳。
设它为TA(n)

当第n次GC结束时（即，在GC工作完成之后，在我们恢复托管线程之前），我们记录另一个时间戳。设它为TB(n)

所以GC的时间是TB(n) - TA(n)最后一次GC结束的时间是TB(n) - TB（n-1）
所以在GC %时间(TB (n) - TA (n)) /结核病(TB (n) - (n - 1))。

因为我们只记录了时间戳，所以实际上并没有忽略线程被切换出去的时间——例如，
如果您在单进程机器上，而另一个进程具有相同优先级的线程（与执行GC的线程相同）
它可能会占用正在执行GC的线程的一些时间。尽管如此，这是一个很好的近似。

当分页发生时，它不是一个很好的近似值。
在这种情况下，您将看到GC的% Time非常高，但实际上用于GC工作的时间很低，因为大部分时间都用于IO。
要验证您是否遇到了这种情况，您可以查看Memory\Pages/sec，以查看它的分页量。

runaway更准确，因为它确实记录了花在线程上的实际时间。
然而，在使用！runaway查看GC所花费的时间时，我确实发现了两个常见的错误。

1)将“花费在GC线程上的时间”误认为“花费在GC中的时间”。

让我们以服务器GC为例。当需要GC时，GC线程首先执行挂起工作。
显然，这需要时间。有时候，如果有很多线程，或者有些线程卡住了，需要很长时间才能挂起它们，这可能需要很长时间。

2)使用当前失控的输出来判断GC花费了多少时间，而不考虑一些用户线程已经死亡。
除了查看用户/内核模式时间，您可能还想查看运行时间。下面的输出来自我看到的一个问题：