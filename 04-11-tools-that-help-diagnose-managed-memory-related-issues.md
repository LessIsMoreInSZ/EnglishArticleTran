<h1>Tools that help diagnose managed memory related issues</h1>
I was writing an internal wiki page on performance and thought this info is useful to many external readers as well so here it goes.

vadump is a good start. It’s an mstools tool – meaning you can find it on your NT CD under bin\mstools. You can take a snapshot of the process and see if the GC heap is an issue or not. It was created a long time ago before managed code was popular so the GC heap is  part of “Other data” in the output of vadump (heh…:P). If it’s a small portion of your working set it’s better to look elsewhere. It shows a lot of useful info though and I’ve worked with many external customers that didn’t know about it so I thought it’d be useful to mention it.

 

Type vadump -? for help. Below is an example using vadump:

 

E:\clr\ndp\clr\src\VM>tlist | findstr /i devenv

1812 devenv.exe        Microsoft Development Environment [design] – gcpriv.h

E:\clr\ndp\clr\src\VM>vadump -so -p 1812

Catagory                        Total        Private Shareable    Shared

                           Pages    KBytes    KBytes    KBytes    KBytes

      Page Table Pages       116       464       464         0         0

      Other System           127       508       508         0         0

      Code/StaticData       3470     13880      1000      7828      5052

      Heap                  6521     26084     26084         0         0

      Stack                   51       204       204         0         0

      Teb                     14        56        56         0         0

      Mapped Data            943      3772         0       444      3328

      Other Data            6595     26380     26376         4         0

 

      Total Modules         3470     13880      1000      7828      5052

      Total Dynamic Data   14124     56496     52720       448      3328

      Total System           243       972       972         0         0

Grand Total Working Set    17837     71348     54692      8276      8380

 

Module Working Set Contributions in pages

    Total   Private Shareable    Shared Module

       10         4         6         0 devenv.exe

       75         4         0        71 ntdll.dll

       64         3         5        56 kernel32.dll

… [more modules omitted]

 

       20         2         0        18 NETAPI32.dll

 

Heap Working Set Contributions

2564 pages from Process Heap (class 0x00000000)

       0x00240000 – 0x00340000 242 pages

       0x02A80000 – 0x02B80000 153 pages

       0x02E10000 – 0x03010000 482 pages

       0x1AF10000 – 0x1B310000 911 pages

       0x207D0000 – 0x20FD0000 776 pages

  13 pages from Private Heap 0 (class 0x00001000)

       0x00350000 – 0x00360000 13 pages

… [more heap data omitted]

 

Stack Working Set Contributions

  33 pages from stack for thread 00000DE8

   2 pages from stack for thread 0000071C

   0 pages from stack for thread 00000CEC

   1 pages from stack for thread 00000B3C

… [more stack data omitted]

 

Usually when we investigate issues, a perfmon log is essential. It shows you the size of each generation in the GC heap, the number of collections done on each generation, the number of GC handles and etc. And most importantly it shows you a trend so you know how your app behaves over a period of time which is often extremely important for discovering problems. See my GC perf counter blog entry for an explanation of these counters.

 

When people have trouble finding out what’s holding onto memory CLRProfiler is a great tool to help figure that out. The CLRProfiler is a visual tool that shows you a tremendous amount of info on the managed memory usage such as the types of the objects that were allocated and relocated, when GCs were triggered and how much memory is allocated by managed API calls.  Since it comes with a great in-depth document I won’t repeat stuff here.

 

The SoS(Son of Strike) debugger extension is also a very useful and powerful tool. It gives you insights that are not easily found in other tools. It comes with the latest Windows Debugger Package (sos.dll in the clr10 directory). Michael Stanton has a nice blog entry about using it.

 

Abhi Khune has a blog entry that illustrates using some of the tools mentioned above to solve memory related problems. Please take a look at it here.

https://devblogs.microsoft.com/dotnet/tools-that-help-diagnose-managed-memory-related-issues/
