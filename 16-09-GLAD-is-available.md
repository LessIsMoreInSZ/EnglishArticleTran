<h1>GLAD is available</h1>

End of last year I mentioned we wanted to provide an API for you to really investigate GC/managed memory related performance called GLAD. Well, 
the source finally got opened source on github. So GLAD is available. The repo is called PerfView but you actually just need the TraceEvent project 
(but it’s much easier to just build the whole solution then add the reference to the resulting Microsoft.Diagnostics.Tracing.TraceEvent.dll). 
Below is a very simple example of getting the total GC pause time for each process (that has GC pauses) and printing out this info along with the process name and pid.

Copy
using System;
using System.Diagnostics.Tracing;
using Microsoft.Diagnostics.Tracing.Session;
using Microsoft.Diagnostics.Tracing.Parsers;
using Microsoft.Diagnostics.Tracing.Analysis;
using Microsoft.Diagnostics.Tracing.Analysis.JIT;
using Microsoft.Diagnostics.Tracing.Analysis.GC;
using Microsoft.Diagnostics.Tracing.Parsers.Clr;
using System.Collections.Generic;

namespace GCInfoProcessing
{
    class Program
    {
        // Given an .etl file, print out GC stats.
        static void DecodeEtl(string strName)
        {
            using (var source = new Microsoft.Diagnostics.Tracing.ETWTraceEventSource(strName))
            {
                Console.WriteLine("{0}", strName);
                source.NeedLoadedDotNetRuntimes();

                source.Process();
                List<TraceGC> GCs = null;

                foreach (var proc in source.Processes())
                {
                    var mang = proc.LoadedDotNetRuntime();
                    if (mang == null) continue;

                    int total_gcs = 0;
                    double total_pause_ms = 0;

                    // This is the list of GCs with processed info
                    GCs = mang.GC.GCs;
                    for (int i = 0; i < GCs.Count; i++) 
                    { 
                        TraceGC gc = GCs[i]; 
                        total_gcs++; 
                        total_pause_ms += gc.PauseDurationMSec; 
                    } 
                    if (total_gcs > 0)
                         Console.WriteLine("process {0} ({1}): total {2} GCs, pause {3:n3}ms", 
                                           proc.Name, proc.ProcessID, total_gcs, total_pause_ms);
                }
            }
        }

        static void Main(string[] args)
        {
            DecodeEtl(args[0]);
        }
    }
}
I’ll give a brief description of how things work for GLAD but with the code publicly available it should be fairly easy to just build and step through the code to see how it works.

TraceEvent\Computers\TraceManagedProcess.cs processes the GC ETW events and generates the info available in the TraceGC class (I edited the comments so they don’t cause trouble for html):

Copy
public class TraceGC
{
    // Primary GC information
    // Set in GCStart (starts at 1, unique for process)
    public int Number;
    // Type of the GC, eg. NonConcurrent, Background or Foreground
    // Set in GCStart
    public GCType Type;
    /// Reason for the GC, eg. exhausted small heap, etc.
    // Set in GCStart
    public GCReason Reason;
    /// Generation of the heap collected. If you compare Generation at the start and stop GC events they may differ.
    // Set in GCStop(Generation 0, 1 or 2)
    public int Generation;
    /// Time relative to the start of the trace. Useful for ordering
    // Set in Start, does not include suspension.
    public double StartRelativeMSec;
    /// Duration of the GC, excluding the suspension time
    // Set in Stop This is JUST the GC time (not including suspension) That is Stop-Start.
    public double DurationMSec;
    /// Duration the EE suspended the process
    // Total time EE is suspended (can be less than GC time for background)
    public double PauseDurationMSec;
    //......
}
You will see a bunch of fields with this comment:

Copy
[Obsolete("This is experimental, you should not use it yet for non-experimental purposes.")]
It’s not obsolete – it’s just experimental. We wanted to organize the info available in TraceGC in a user friendly way (and you are welcome to suggest/contribute to it!) and we want your input to really flesh out the API aspect. Do we just want to expose these as is, or want to have a more advanced class to represent info that’s less frequently used? It’d be great to hear some opinions on this. Feel free to either leave them as comments here or on the github repo. The process part is done at the beginning of this file. An example is

Copy
source.Clr.GCStart += delegate (GCStartTraceData data)
{
    // ....
};
If you want to see examples of how TraceGC is used, you can get plenty of such examples in PerfView\GcStats.cs – this is what generates the GCStats view in PerfView.

Looking forward to seeing the analysis that folks write on memory analysis for .NET 🙂

Edited on 03/01/2020 to add the indices to GC ETW Event blog entries

https://devblogs.microsoft.com/dotnet/556-2/