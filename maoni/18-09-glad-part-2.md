<h1>GLAD Part 2</h1>

2 years ago I had a blog entry to introduce the GLAD (GC Latency Analysis and Diagnostics) library which provides much more insight into the GC heap performance 
than perf counters and takes care of interpreting raw GC ETW events so our users don’t have to do that work themselves. 
Since then I have not heard too much usage from our customers 😛 So either not many people are using it, 
or people are and they’ve had no problem with it so I don’t hear from them (secretly hoping it’s the latter…).

In any case, I thought I’d give another example that uses GLAD to get GC info at real time – hopefully this will encourage folks 
who are interested in looking at GC perf but haven’t taken advantage of GLAD to take another look and incorporate it into their diagnostics pipeline. 
You just need to reference the TraceEvent dll (ie, Microsoft.Diagnostics.Tracing.TraceEvent.dll; I found that you also need Microsoft.Diagnostics.FastSerialization.dll in the same dir).

If you have used TraceEvent you probably have looked at the TraceEvent Programmers’ Guide doc in the PerfView repo. 
Most people know that you can use TraceEvent to decode an ETW trace after you collect such a trace; 
but it can also do real time processing (which really is just a feature with ETW – you could use plain Win32 ETW API and do either real time or post processing on ETW events) 
which the doc specifically talks about in the “Real Time Processing of Events” section. 
But we made it even easier than that for you so you don’t have to care about noticing 
when an individual GC is finished processing to get the processed info – you can just use the processed info in the TraceLoadedDotNetRuntime.GCEnd callback. 
Below is a method that takes a PID and prints out the GC start and end with the pause time as each GC happens 
(obviously the pause time is only available at the GC end – at that point a full GC event sequence is completely and the pause time info has been calculated and is available):

Copy
        public static void RealTimeProcessing(int pid)
        {
            Console.WriteLine("Monitoring process {0}", pid);

            using (var session = new TraceEventSession("My Session"))
            {
                var source = session.Source;
                source.NeedLoadedDotNetRuntimes();
                source.AddCallbackOnProcessStart(delegate (TraceProcess proc)
                {
                    proc.AddCallbackOnDotNetRuntimeLoad(delegate (TraceLoadedDotNetRuntime runtime)
                    {
                        runtime.GCStart += delegate (TraceProcess p, TraceGC gc)
                        {
                            if (p.ProcessID == pid)
                            {
                                Console.WriteLine("proc {0}: GC#{1} start at {2:0.00}ms", p.ProcessID, gc.Number, gc.PauseStartRelativeMSec);
                            }
                        };
                        runtime.GCEnd += delegate (TraceProcess p, TraceGC gc)
                        {
                            if (p.ProcessID == pid)
                            {
                                Console.WriteLine("proc {0}: GC#{1} end, paused for {2:0.00}ms)",
                                p.ProcessID, gc.Number, gc.PauseDurationMSec);
                            }
                        };
                    });
                });

                session.EnableProvider(ClrTraceEventParser.ProviderGuid);

                source.Process();

                foreach (var proc in source.Processes())
                {
                    Console.WriteLine("{0}", proc.ToString());
                }
            }
        }
(I just noticed that p.Name doesn’t seem to work, so I am using the pid instead)

Below is a sample output – I ran a test that keeps doing GCs and ran this tool. It prints out info about each one as it starts and ends:

Copy
E:\tests\RealtimeMon\realmon\realmon\bin\Release>realmon 11112
Monitoring process 11112
proc 11112: GC#30 start at 330.29ms
proc 11112: GC#30 end, paused for 213.52ms)
proc 11112: GC#31 start at 1014.82ms
proc 11112: GC#31 end, paused for 22.70ms)
proc 11112: GC#32 start at 1453.19ms
proc 11112: GC#32 end, paused for 29.50ms)
proc 11112: GC#33 start at 1826.29ms
proc 11112: GC#33 end, paused for 30.10ms)
proc 11112: GC#34 start at 2182.78ms
proc 11112: GC#34 end, paused for 34.25ms)
proc 11112: GC#35 start at 2522.57ms
proc 11112: GC#35 end, paused for 30.29ms)
proc 11112: GC#36 start at 3290.49ms
proc 11112: GC#36 end, paused for 120.48ms)
proc 11112: GC#37 start at 4285.94ms
proc 11112: GC#37 end, paused for 41.22ms)
proc 11112: GC#38 start at 4617.87ms
proc 11112: GC#38 end, paused for 17.90ms)
proc 11112: GC#39 start at 4938.86ms
^C
There are many interesting cases for diagnostics and we’d love to get your help to build up a repertoire, eg if you used it to build a cool GC perf diag tool, 
or found a problem and made a fix it would be greatly appreciated if you share with the rest of us!

https://devblogs.microsoft.com/dotnet/glad-part-2/

两年前，我写了一篇博客介绍 GLAD（GC Latency Analysis and Diagnostics） 库。这个库可以提供比性能计数器更深入的垃圾回收堆性能洞察，
并且会负责解析原始的 GC ETW 事件，因此用户无需自己完成这项工作。从那以后，我没有听到太多来自客户的使用反馈 😛 所以要么是很少有人在使用它，
要么是人们在使用它并且没有遇到问题，所以我没有收到他们的反馈（私下里我希望是后者……）。

无论如何，我想通过另一个示例来展示如何使用 GLAD 实时获取 GC 信息——希望这能鼓励那些对 GC 性能感兴趣但尚未利用 GLAD 的人重新审视它，并将其纳入他们的诊断流程中。
你只需要引用 TraceEvent DLL（即 Microsoft.Diagnostics.Tracing.TraceEvent.dll；我发现你还必须在同一目录下包含 Microsoft.Diagnostics.FastSerialization.dll）。

如果你已经使用过 TraceEvent，你可能已经看过 PerfView 仓库中的 TraceEvent Programmer's Guide 文档。大多数人知道你可以使用 TraceEvent 
在收集到 ETW 跟踪后对其进行解码；但它也可以进行实时处理（这实际上是 ETW 的一个特性——你可以使用普通的 Win32 ETW API 对 ETW 事件进行实时或事后处理）。
文档在“实时事件处理”部分专门讨论了这一点。但我们为你做了进一步简化，因此你不必关心单个 GC 完成处理的时间以获取处理后的信息——你可以在 TraceLoadedDotNetRuntime.GCEnd 
回调中直接使用已处理的信息。下面是一个方法，它接受一个进程 ID 并在每次发生 GC 时打印出 GC 的开始和结束时间以及暂停时间（
显然，暂停时间只能在 GC 结束时获得——此时完整的 GC 事件序列已完成，暂停时间信息已被计算并可用）：

csharp
深色版本
public static void RealTimeProcessing(int pid)
{
    Console.WriteLine("Monitoring process {0}", pid);

    using (var session = new TraceEventSession("My Session"))
    {
        var source = session.Source;
        source.NeedLoadedDotNetRuntimes();
        source.AddCallbackOnProcessStart(delegate (TraceProcess proc)
        {
            proc.AddCallbackOnDotNetRuntimeLoad(delegate (TraceLoadedDotNetRuntime runtime)
            {
                runtime.GCStart += delegate (TraceProcess p, TraceGC gc)
                {
                    if (p.ProcessID == pid)
                    {
                        Console.WriteLine("proc {0}: GC#{1} start at {2:0.00}ms", p.ProcessID, gc.Number, gc.PauseStartRelativeMSec);
                    }
                };
                runtime.GCEnd += delegate (TraceProcess p, TraceGC gc)
                {
                    if (p.ProcessID == pid)
                    {
                        Console.WriteLine("proc {0}: GC#{1} end, paused for {2:0.00}ms)",
                            p.ProcessID, gc.Number, gc.PauseDurationMSec);
                    }
                };
            });
        });

        session.EnableProvider(ClrTraceEventParser.ProviderGuid);

        source.Process();

        foreach (var proc in source.Processes())
        {
            Console.WriteLine("{0}", proc.ToString());
        }
    }
}
（我刚刚注意到 p.Name 似乎不起作用，所以我改用了进程 ID。）

下面是示例输出——我运行了一个不断触发 GC 的测试程序，并运行了这个工具。它会在每个 GC 开始和结束时打印相关信息：

深色版本
E:\tests\RealtimeMon\realmon\realmon\realmon\bin\Release>realmon 11112
Monitoring process 11112
proc 11112: GC#30 start at 330.29ms
proc 11112: GC#30 end, paused for 213.52ms)
proc 11112: GC#31 start at 1014.82ms
proc 11112: GC#31 end, paused for 22.70ms)
proc 11112: GC#32 start at 1453.19ms
proc 11112: GC#32 end, paused for 29.50ms)
proc 11112: GC#33 start at 1826.29ms
proc 11112: GC#33 end, paused for 30.10ms)
proc 11112: GC#34 start at 2182.78ms
proc 11112: GC#34 end, paused for 34.25ms)
proc 11112: GC#35 start at 2522.57ms
proc 11112: GC#35 end, paused for 30.29ms)
proc 11112: GC#36 start at 3290.49ms
proc 11112: GC#36 end, paused for 120.48ms)
proc 11112: GC#37 start at 4285.94ms
proc 11112: GC#37 end, paused for 41.22ms)
proc 11112: GC#38 start at 4617.87ms
proc 11112: GC#38 end, paused for 17.90ms)
proc 11112: GC#39 start at 4938.86ms
^C
有许多有趣的诊断案例，我们非常希望能得到你的帮助来建立一个案例库。例如，如果你用它构建了一个酷炫的 GC 性能诊断工具，或者发现了某个问题并进行了修复，请务必与我们分享！