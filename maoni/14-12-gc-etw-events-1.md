<h1>GC ETW events – 1<h1>
GC ETW series –

GC ETW Events – Part 1 (this post)

GC ETW Events – Part 2

GC ETW Events – Part 3

GC ETW Events – Part 4

Processing GC ETW Events Programmatically with the GLAD Library

A lot of people have mentioned to me that I have not posted anything for a long time. I do realize it and do appreciate being asked to write more. Well, it's end of year and I am starting vacation so I thought I'd write something light that perhaps makes good reading around Christmas time 🙂

Perf counters vs ETW events

Perf counters and ETW events, like anything else, each have their pros and cons. In my Defrag Tools series of videos I mentioned that I really would like folks to start using ETW (via the PerfView tool) if they haven't. I would state that again here because ETW just gives you richer info that you simply cannot get with perf counters. And if you only get the basic GC events, the overhead is very, very low that you could afford to have it on in your production environment. So what's the con? Well, ETW is a newer thing (internally at MS many groups have started using ETW events, perhaps for a long time now; but clearly there are still groups that are using perf counters and not ETW events) which means there's a learning curve. And the learning curve of using ETW is steeper than what you had with perf counters. I'd like to think that the PerfView made that fairly easy to get you started though. Aside from the less rich info, perf counters, as I explained here, also suffer from the precision problem. Just last week I had someone asking me why he's seeing % time in GC being 99% for a few minutes straight. It was because the % time in GC counter is only updated at the end of a GC and in his process since it was idling for a long time after the last gen2 GC (ie, no GCs were happening), the 99% value lasted till next time a GC happened so to him this was misleading.

Collecting GC events with PerfView

After you download perfview from Microsoft.com, you just unzip it and run perfview.exe – yes it's that simple – there's no install involved. There are many different options in perfview to collect ETW events with but for our purpose we want to collect just some GC events to start with. There are 2 ways you can do that:

1) run perfview.exe, click on Collect, then Collect again (or just do Alt+C). You will see a dialog box popping up, click on Advanced options, uncheck “Kernel Base” and .NET and check “GC Collect Only”. This tells it to only collect events fired during GCs. If you don't do this and simply click on “Start Collection” it will collect these GC events as well, but a lot of other events which makes it less sustainable if you aim to run this for many hours on your production server as the problem will likely take that long to repro.

2) use the commandline – I am a commandline person so this is what I use. It also makes it much easier to include in your automation scripts. Run this commandline (if you run this from a non admin cmd window it will pop up some UI asking for permission to run as admin as collecting ETW requires admin privilege; or you just run it from an admin cmd window):

perfview /GCCollectOnly /AcceptEULA /nogui collect

/GCCollectOnly, as name suggests, is the one that's telling it to collect only GC events.

Then you run your scenario for long enough for it to show the symptoms. And you can stop it by either clicking on “Stop collection” or press ‘s' in the cmd window to stop the collection. If you know approximately how long it takes for the symptoms to show up, say 10 mins, you can also do

perfview /GCCollectOnly /AcceptEULA /nogui /MaxCollectSec:600 collect

which will stop after it's collected for 600 seconds, ie, 10 mins.

BTW, you can see all the available commandline args in Help\Command Line Help. This gives you 2 resulting files: PerfViewGCCollectOnly.etl.zip and PerfViewGCCollectOnly.log.txt. The latter is logging what perfview did. The former is what we will be diving into.

The GCStats view in PerfView

Open the PerfViewGCCollectOnly.etl.zip file in perfview, meaning either by running Perfview and browsing to that directory and double clicking on that file; or running the “perfview PerfViewGCCollectOnly.etl.zip” commandline. You will see that there are multiple nodes under that filename. What we are interested in is the GCStats view under the “Memory Group” node. Double click on it to open it. At the top we have something like this:

GCStats

Process 3768: WDExpress
Process 6484: perfview /GCCollectOnly /AcceptEULA /nogui collect
Process 1108: sqlservr -sMSSQLSERVER
I ran Visual Studio Express which is a managed app – that's the WDExpress process at the top. For each of these processes it will have the following format:

Summary – this includes things like the commandline, CLR startup flags, % time pause for GC and etc.

GC stats rollup by generation – for gen0/1/2 it has things like how many GCs of that gen were done, their mean/average pauses and etc.

GC stats for GCs whose pause time was >200ms

LOH Allocation Pause (due to background GC) > 200 Msec for this process

Gen2 GC stats

All GC stats

Condemned reasons for GCs

Summary explanation

I will explain the summary section in this blog entry and the others in the next one.

Commandline and runtime version are self-explanatory.

CLR Startup Flags: the main thing for GC investigation purpose is to look for CONCURRENT_GC and SERVER_GC. If you don't have either one it means you are using Workstation GC with concurrent GC disabled.

Total CPU Time and Total GC CPU Time: these would always be 0 unless you are actually collecting CPU samples.

Total Allocs: total allocations you've done during this trace for this process.

MSec/MB Alloc: this is 0 unless you collect CPU samples (it would be Total GC CPU time / Total allocated bytes).

Total GC pause: total wallclock time paused by GCs. Note that this includes suspension time, ie, the time it takes to suspend managed threads before GC can start.

% Time paused for Garbage Collection: this is the total GC pause / total process running time.

% CPU Time spent Garbage Collecting: this is NaN% unless you collect CPU samples.

Max GC Heap Size: maximum managed heap size during this trace for this process.

Something worth mentioning is the difference between GC pauses and GC CPU time. The former is elapsed time (ie, the time elapsed between suspension starts and resumption ends for this GC) and the latter is, as the name suggests, how many CPU samples we actually collected for this process during GC.

Edited on 03/01/2020 to add the indices to GC ETW Event blog entries

https://devblogs.microsoft.com/dotnet/gc-etw-events-1/

GC ETW 系列 –

GC ETW 事件 – 第1部分（本文）

GC ETW 事件 – 第2部分

GC ETW 事件 – 第3部分

GC ETW 事件 – 第4部分

使用 GLAD 库编程处理 GC ETW 事件

很多人向我提到，我已经很久没有发布任何内容了。我确实意识到了这一点，并且非常感谢大家要求我多写一些。嗯，现在是年底，我开始休假了，所以我想写一些轻松的内容，或许适合在圣诞节期间阅读 🙂

性能计数器 vs ETW 事件

性能计数器和 ETW 事件，像其他任何事物一样，各有优缺点。在我的 Defrag Tools 系列视频中，我提到我真的很希望那些还没有使用 ETW（通过 PerfView 工具）的人开始使用它。
我在这里再次重申这一点，因为 ETW 提供了更丰富的信息，这些信息是你无法通过性能计数器获得的。而且如果你只获取基本的 GC 事件，开销非常低，你甚至可以在生产环境中启用它。
那么缺点是什么呢？嗯，ETW 是一个相对较新的东西（在微软内部，许多团队已经开始使用 ETW 事件，可能已经有一段时间了；但显然仍有一些团队在使用性能计数器而不是 ETW 事件），
这意味着有一个学习曲线。而且使用 ETW 的学习曲线比使用性能计数器要陡峭。我想说的是，PerfView 使得入门变得相当容易。除了信息不够丰富之外，性能计数器，正如我在这里解释的那样，还存在精度问题。
就在上周，有人问我为什么他看到 % time in GC 在几分钟内一直保持在 99%。这是因为 % time in GC 计数器只在 GC 结束时更新，而在他的进程中，由于在上一次 Gen2 GC 之后长时间处于空闲状态（即没有发生 GC），
99% 的值一直持续到下一次 GC 发生，所以对他来说这是误导性的。

使用 PerfView 收集 GC 事件

从 Microsoft.com 下载 PerfView 后，你只需解压缩并运行 perfview.exe —— 是的，就这么简单 —— 不需要安装。PerfView 中有许多不同的选项来收集 ETW 事件，但为了我们的目的，我们只想收集一些 GC 事件。
有两种方法可以做到这一点：

运行 perfview.exe，点击 Collect，然后再次点击 Collect（或直接按 Alt+C）。你会看到一个对话框弹出，点击 Advanced options，取消选中“Kernel Base”和 .NET，并选中“GC Collect Only”。
这告诉它只收集在 GC 期间触发的事件。如果你不这样做，而是直接点击“Start Collection”，它也会收集这些 GC 事件，但还会收集许多其他事件，这使得如果你打算在生产服务器上长时间运行它，可能会不太可持续，
因为问题可能需要很长时间才能重现。

使用命令行 —— 我是一个命令行爱好者，所以我使用这种方法。这也使得将其包含在你的自动化脚本中变得更加容易。运行以下命令行（如果你从非管理员 cmd 窗口运行此命令，它会弹出一些 UI 要求以管理员权限运行，
因为收集 ETW 需要管理员权限；或者你可以直接从管理员 cmd 窗口运行）：

复制
perfview /GCCollectOnly /AcceptEULA /nogui collect
/GCCollectOnly，顾名思义，是告诉它只收集 GC 事件的选项。

然后你运行你的场景，直到它显示出症状。你可以通过点击“Stop collection”或在 cmd 窗口中按‘s’来停止收集。如果你知道症状出现的大致时间，比如 10 分钟，你也可以这样做：

复制
perfview /GCCollectOnly /AcceptEULA /nogui /MaxCollectSec:600 collect
这将在收集 600 秒（即 10 分钟）后停止。

顺便说一下，你可以在 Help\Command Line Help 中看到所有可用的命令行参数。这会生成两个结果文件：PerfViewGCCollectOnly.etl.zip 和 PerfViewGCCollectOnly.log.txt。后者记录了 PerfView 的操作。
前者是我们将要深入研究的文件。

PerfView 中的 GCStats 视图

在 PerfView 中打开 PerfViewGCCollectOnly.etl.zip 文件，意味着要么运行 PerfView 并浏览到该目录并双击该文件；要么运行“perfview PerfViewGCCollectOnly.etl.zip”命令行。你会看到该文件名下有多个节点。
我们感兴趣的是“Memory Group”节点下的 GCStats 视图。双击它以打开它。在顶部，你会看到类似这样的内容：

复制
GCStats

Process 3768: WDExpress
Process 6484: perfview /GCCollectOnly /AcceptEULA /nogui collect
Process 1108: sqlservr -sMSSQLSERVER
我运行了 Visual Studio Express，这是一个托管应用程序 —— 这是顶部的 WDExpress 进程。对于每个进程，它将具有以下格式：

摘要 —— 包括命令行、CLR 启动标志、GC 暂停时间百分比等内容。

按代划分的 GC 统计汇总 —— 对于 Gen0/1/2，它有诸如该代进行了多少次 GC、它们的平均暂停时间等内容。

暂停时间 >200ms 的 GC 统计

LOH 分配暂停（由于后台 GC）> 200 毫秒的进程

Gen2 GC 统计

所有 GC 统计

GC 的 condemned 原因

摘要解释

我将在本文中解释摘要部分，其他部分将在下一篇中解释。

命令行和运行时版本是不言自明的。

CLR 启动标志：对于 GC 调查目的，主要看 CONCURRENT_GC 和 SERVER_GC。如果你没有其中任何一个，这意味着你使用的是禁用并发 GC 的工作站 GC。

总 CPU 时间和总 GC CPU 时间：除非你实际收集 CPU 样本，否则这些值将始终为 0。

总分配：在此跟踪期间为该进程完成的总分配。

MSec/MB 分配：除非你收集 CPU 样本，否则此值为 0（它将是总 GC CPU 时间 / 总分配的字节数）。

总 GC 暂停：GC 暂停的总挂钟时间。请注意，这包括挂起时间，即在 GC 开始之前挂起托管线程所需的时间。

垃圾回收暂停时间百分比：这是总 GC 暂停时间 / 总进程运行时间。

垃圾回收占用的 CPU 时间百分比：除非你收集 CPU 样本，否则此值为 NaN%。

最大 GC 堆大小：在此跟踪期间该进程的最大托管堆大小。

值得一提的是 GC 暂停时间和 GC CPU 时间之间的区别。前者是经过的时间（即从挂起开始到恢复结束的时间），而后者是，顾名思义，我们在此 GC 期间为该进程实际收集的 CPU 样本数量。

2020年3月1日编辑，添加了 GC ETW 事件博客条目的索引。