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