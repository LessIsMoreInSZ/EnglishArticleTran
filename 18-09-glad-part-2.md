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