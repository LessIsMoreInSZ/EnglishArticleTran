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

去年年底，我提到我们想提供一个 API，让你能够真正调查与 GC（垃圾回收）/托管内存相关的性能问题，这个 API 叫做 GLAD。
现在，它的源代码终于在 GitHub 上开源了。GLAD 已经可用。仓库名为 PerfView，但你实际上只需要 TraceEvent 项目（
不过直接构建整个解决方案然后引用生成的 Microsoft.Diagnostics.Tracing.TraceEvent.dll 会更容易）。
下面是一个非常简单的示例，展示了如何获取每个进程（具有 GC 暂停的进程）的总 GC 暂停时间，并打印出这些信息以及进程名称和 PID。

csharp
复制
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
        // 给定一个 .etl 文件，打印出 GC 统计信息。
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

                    // 这是包含处理后的 GC 信息的列表
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
我会简要描述 GLAD 的工作原理，但由于代码已经公开，构建并逐步调试代码以了解其工作原理应该相当容易。

TraceEvent\Computers\TraceManagedProcess.cs 处理 GC ETW 事件并生成 TraceGC 类中可用的信息（我编辑了注释以避免 HTML 问题）：

csharp
复制
public class TraceGC
{
    // 主要的 GC 信息
    // 在 GCStart 中设置（从 1 开始，进程内唯一）
    public int Number;
    // GC 的类型，例如 NonConcurrent、Background 或 Foreground
    // 在 GCStart 中设置
    public GCType Type;
    /// GC 的原因，例如小堆耗尽等。
    // 在 GCStart 中设置
    public GCReason Reason;
    /// 收集的堆的代。如果你比较 GC 开始和停止事件中的 Generation，它们可能不同。
    // 在 GCStop 中设置（0、1 或 2 代）
    public int Generation;
    /// 相对于跟踪开始的时间。用于排序
    // 在 Start 中设置，不包括暂停时间。
    public double StartRelativeMSec;
    /// GC 的持续时间，不包括暂停时间
    // 在 Stop 中设置。这只是 GC 时间（不包括暂停时间），即 Stop-Start。
    public double DurationMSec;
    /// EE 暂停进程的持续时间
    // EE 暂停的总时间（对于后台 GC，可能小于 GC 时间）
    public double PauseDurationMSec;
    //......
}
你会看到一些带有以下注释的字段：

csharp
复制
[Obsolete("This is experimental, you should not use it yet for non-experimental purposes.")]
它并没有过时——只是实验性的。我们希望以用户友好的方式组织 TraceGC 中的信息（欢迎你建议或贡献！），并且我们希望听取你的意见来完善 API 的设计。我们是直接暴露这些信息，还是希望有一个更高级的类来表示不常用的信息？听听大家的意见会很好。你可以在这里或 GitHub 仓库中留下评论。
这部分处理逻辑在这个文件的开头完成。例如：

csharp
复制
source.Clr.GCStart += delegate (GCStartTraceData data)
{
    // ....
};
如果你想查看 TraceGC 的使用示例，可以在 PerfView\GcStats.cs 中找到很多这样的示例——这是 PerfView 中生成 GCStats 视图的部分。

期待看到大家为 .NET 编写的内存分析工具 🙂

编辑于 2020 年 3 月 1 日：添加了 GC ETW 事件博客条目的索引。