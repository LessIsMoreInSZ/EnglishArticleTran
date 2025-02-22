<h1>Are you GLAD?</h1>

Holidays are almost upon us. Last year around Christmas time I wrote a set of GC ETW blog entries to help with diagnosing GC heap related issues. 
This year I want to discuss taking that a (big) step further and making an SDK that does the analysis for you using these ETW events – so before I leave for vacation 
I’d like to share with you the plan I have for the SDK.

The SDK is called GLAD which stands for GC Latency Analysis and Diagnostics. 
This will essentially be based on the analysis and diagnostics work for the GC we have in PerfView today but I wanted to isolate it out and make it into an SDK so everyone can benefit from it. 
And it will be open sourced on github – we will continue our development on GLAD there; and everyone else is welcome to contribute to it.

We will start GLAD off by converting our analysis part into this form. The diagnostics will come after. 
The difference between the two is analysis shows you info on each GC while diagnostics actually looks at the histogram of the GCs and tells you what issues it found.

I should note that I am not an API person so I will not take offence if you tell me my API design sucks, but at the same time I am asking you to help me out and suggest something better 🙂

I would greatly appreciate your feedback on it so please feel free to comment (or if you are simply GLAD to see it, you can say just that too!). 
I will look at all the comments around Christmas (seems like an appropriate time to talk about this kind of stuff).

Motivation

It’s certainly not a trivial task to look at the GC behavior. Historically we’ve built this knowledge in our own tool, PerfView, which takes ETW traces and produces insights in the GC behavior. 
PerfView is used by many teams internally at Microsoft. However, we recognize that some teams already have or are adding their own ETW processing pipeline. 
Obviously they would like to add GC pauses as part of their ETW analysis. Interpreting GC ETW events however presents a challenge for them. For example, even accounting for GC pauses, 
which is a basic and fundamental metric, is often incorrectly done due to the complexity introduced by Background GC. 
Therefore we would like to make the code available for anyone who would want to incorporate the GC behavior analysis into their own ETW processing.

Background

For those of you who are not familiar with how the workflow for analysis with ETW events usually look like, let me use PerfView as an example.

After you collect a trace (you can do this with PerfView as I mentioned in my ETW blog entries, or you could use WPA which is another Microsoft tool that’s built by the OS folks,
 or something else of your choice), you need to convert the events from the binary form into a human consumable form. In order for this to happen you need to know the event layout. 
 The GC ETW events’ layout is documented here. For example, the GCStart version 1 event has the following fields along with their types and explanation:

假期即将来临。去年圣诞节前后，我写了一系列关于GC ETW（事件跟踪）的博客文章，帮助诊断与GC堆相关的问题。今年，我想进一步讨论如何利用这些ETW事件，开发一个SDK来自动进行分析 —— 因此，在我休假之前，我想与大家分享我对这个SDK的计划。

这个SDK叫做GLAD，全称是GC Latency Analysis and Diagnostics（GC延迟分析和诊断）。它本质上将基于我们今天在PerfView中为GC所做的分析和诊断工作，但我希望将其独立出来，做成一个SDK，以便每个人都能从中受益。它将在GitHub上开源 —— 我们将在那里继续开发GLAD；也欢迎其他人贡献代码。

我们将从将分析部分转换为这种形式开始GLAD的开发。诊断部分将在之后进行。两者的区别在于，分析部分会显示每个GC的信息，而诊断部分则会查看GC的直方图，并告诉你发现了哪些问题。

需要注意的是，我并不是一个API设计专家，所以如果你告诉我我的API设计很糟糕，我不会生气，但同时我也请求你帮助我，提出更好的建议 🙂

我非常感谢你的反馈，所以请随意评论（或者如果你只是很高兴看到它，你也可以这么说！）。我会在圣诞节前后查看所有的评论（似乎这是一个讨论这类话题的合适时机）。

动机
分析GC行为并不是一项简单的任务。历史上，我们在自己的工具PerfView中构建了这些知识，PerfView通过处理ETW跟踪数据来生成对GC行为的洞察。PerfView被微软内部的许多团队使用。然而，我们认识到，一些团队已经拥有或正在添加他们自己的ETW处理管道。显然，他们希望将GC暂停作为其ETW分析的一部分。然而，解释GC ETW事件对他们来说是一个挑战。例如，即使是计算GC暂停时间这样一个基本且关键的指标，由于后台GC（Background GC）引入的复杂性，也经常被错误地处理。因此，我们希望将这些代码公开，供任何希望将GC行为分析集成到他们自己的ETW处理中的人使用。

背景
对于那些不熟悉如何使用ETW事件进行分析的工作流程的人，让我以PerfView为例进行说明。

在你收集了跟踪数据后（你可以使用PerfView，正如我在ETW博客文章中提到的，或者你可以使用WPA，这是另一个由操作系统团队开发的微软工具，或者你选择的其他工具），你需要将这些事件从二进制形式转换为人类可读的形式。为了实现这一点，你需要知道事件的布局。GC ETW事件的布局在这里有文档记录。例如，GCStart版本1事件具有以下字段及其类型和解释：

GCStart_V1事件字段
字段名	类型	描述
Count	UInt32	GC的计数（从进程启动开始的第N次GC）。
Depth	UInt32	GC的代（0、1或2）。
Reason	UInt32	触发GC的原因（例如，分配、低内存等）。
Type	UInt32	GC的类型（例如，阻塞GC、后台GC等）。
**ClrInstanceID	UInt16	CLR实例的唯一标识符（用于多CLR实例的场景）。
GLAD SDK的目标
GLAD SDK的目标是简化GC行为的分析过程，使开发人员能够轻松地将GC暂停时间和其他相关指标集成到他们的性能监控和分析工具中。通过提供清晰的API和开源代码，我们希望帮助开发人员更好地理解他们的应用程序的GC行为，并优化其性能。

API设计思路
虽然我不是API设计专家，但我希望GLAD的API能够简单易用，同时提供足够的灵活性来处理各种场景。以下是我对API设计的一些初步想法：

初始化：提供一个简单的初始化方法，用于设置ETW事件的处理管道。

csharp
复制
GladAnalyzer.Initialize();
事件处理：允许用户注册回调函数来处理特定的GC事件。

csharp
复制
GladAnalyzer.OnGCStart += (gcStartEvent) => {
    Console.WriteLine($"GC Started: Generation {gcStartEvent.Depth}, Reason {gcStartEvent.Reason}");
};
暂停时间计算：自动计算并报告GC暂停时间，处理后台GC的复杂性。

csharp
复制
GladAnalyzer.OnGCPause += (pauseDuration) => {
    Console.WriteLine($"GC Pause Duration: {pauseDuration.TotalMilliseconds} ms");
};
诊断报告：生成诊断报告，指出潜在的GC性能问题。

csharp
复制
var diagnosticsReport = GladAnalyzer.GenerateDiagnosticsReport();
Console.WriteLine(diagnosticsReport);
总结
GLAD SDK的目标是让GC行为分析变得更加简单和可访问。通过开源和社区贡献，我们希望不断改进这个工具，使其成为每个开发人员在优化应用程序性能时的得力助手。如果你对这个项目感兴趣，或者有任何建议，请在评论区留言。我会在圣诞节前后查看所有的反馈。

祝你假期愉快！🎄



Count:         UInt32 (the GC index)

Depth:         UInt32 (which generation)

Reason:        UInt32 (why was this GC triggered)

Type:          UInt32 (is this a non concurrent GC, a foreground or a background GC?)

ClrInstanceID: UInt16 (you can ignore this for this discussion)

Also for each ETW, by default, there’s a bunch of information already available to you such as the timestamp when this event was fired.

We have a library that does exactly that for you – it’s called TraceEvent – it knows the event layouts (including a bunch of OS events, CLR events, asp.net events and etc) 
and as it encounters each event it will convert it to a type that you can consume in your code. So when it sees a GCStart event, 
it will construct a type that has exactly those fields that you can read; and it also gives you a chance to do some event processing. 
So let’s say I want to write in my log everytime I see a GC start and end with their timestamps and the process it happened in:

// ETWTraceEventSource is what knows how to convert a .etl file to

// the human readable event form.

ETWTraceEventSource source = new ETWTraceEventSource(“myTrace.etl”);


// Clr is a provider that TraceEvent knows about which represents,

// you guess it, the CLR ETW provider. It has a bunch of events defined

// like GCStart and GCStop.

source.Clr.GCStart += delegate(GCStartTraceData data)

{
    // Since GC is per process, we’d like note down the process ID as well.

    myLog.WriteLine(“Observed in process {0} GC#{1} started at {2}ms”,

                    data.ProcessID, data.Count, data.TimeStampRelativeMSec);

}

source.Clr.GCStop +=  delegate(GCEndTraceData data)

{

    // Since GC is per process, we’d like note down the process ID as well.

    myLog.WriteLine(“Observed in process {0} GC ended at {1}ms”,

                    data.ProcessID, data.TimeStampRelativeMSec);
};

(note that this code sample is based on the current TraceEvent)

GLAD essentially replaces the bold lines with much more interesting processing – it would provide rich info on each GC such as time managed threads were paused, promoted size, 
reasons why we decided to collect this generation, fragmentation, pinning, time for marking from different sources and etc.

Details

The SDK will consist of the following:

1) The implementation of a set of action delegates that correspond to each event where each delegate takes one parameter which is the event data (that’s what we’ll be processing for that event).

2) A list of GCInformation for each GC we processed.

Normally we don’t care about partial GCs, ie, a GC that we don’t have the full sequence of events for. There are 2 ways the user can recognize a complete GC:

1) via events – when we have processed a full sequence of events for a GC, we fire an event (not to be confused with an ETW event! This is just a c# event);

2) via checking for the isComplete field in GCInformation. The user can do this in the last ETW event in the sequence (which is the GCHeapStats event). 
After he invoked our delegate to process this event, isComplete is set to true unless we did not see the full sequence. 
Not seeing the full sequence usually happens at the beginning of the trace. Occasionally I’ve also seen that we miss an event in this sequence.

User could use this for real-time or post processing. During real-time processing, the user could choose to perform some action when he sees an “interesting” GC, eg, 
when a GC is just completed, he checks and sees that the duration is >1s, he then stops his ETW collection and saves the events to a file for someone to investigate.

Input

Users will provide the SDK with the event data to interpret. If you use TraceEvent you automatically get the event data; 
if you choose to implement your own you need to implement the interfaces that are defined in the contracts. So I would strongly encourage you to just use TraceEvent.

The following events are mandatory

Process Start/Stop/DCStart/DCStop events from the Kernel provider

GC is per process which means there’s per process state we’d like to keep. These events give us the process info such as their PIDs and names.

GC informational level events from CLR public/private providers

The following events are optional

GC verbose level events from CLR public/private providers

If these events are present, more analysis will light up.
GLAD本质上会用更有趣的处理方式替换掉那些粗体行 —— 它将为每次GC提供丰富的信息，例如托管线程暂停的时间、提升的大小、决定回收该代的原因、碎片化、固定（pinning）、来自不同源的标记时间等等。

细节
SDK将包括以下内容：

一组操作委托的实现：每个委托对应一个事件，每个委托接受一个参数，即事件数据（这是我们将为该事件处理的内容）。

每个已处理GC的GCInformation列表。

通常情况下，我们不关心部分GC，即我们没有完整事件序列的GC。用户可以通过以下两种方式识别完整的GC：

通过事件 —— 当我们处理完一个GC的完整事件序列时，我们会触发一个事件（不要与ETW事件混淆！这只是一个C#事件）；

通过检查GCInformation中的isComplete字段。用户可以在序列中的最后一个ETW事件（即GCHeapStats事件）中执行此操作。在他调用我们的委托处理此事件后，isComplete将被设置为true，除非我们没有看到完整的事件序列。没有看到完整序列通常发生在跟踪的开始部分。偶尔，我也看到我们在这个序列中遗漏了一个事件。

用户可以将此用于实时或后处理。在实时处理期间，用户可以选择在看到“有趣”的GC时执行某些操作，例如，当一个GC刚刚完成时，他检查并发现持续时间超过1秒，然后停止他的ETW收集并将事件保存到文件中以供进一步调查。

输入
用户将向SDK提供事件数据以进行解释。如果你使用TraceEvent，你会自动获取事件数据；如果你选择自己实现，则需要实现合同中定义的接口。因此，我强烈建议你直接使用TraceEvent。

以下事件是必需的：

来自Kernel提供程序的Process Start/Stop/DCStart/DCStop事件
GC是按进程进行的，这意味着我们希望保留每个进程的状态。这些事件为我们提供了进程信息，例如它们的PID和名称。

来自CLR公共/私有提供程序的GC信息级别事件

以下事件是可选的：

来自CLR公共/私有提供程序的GC详细级别事件
如果存在这些事件，将启用更多分析。


API

Contracts (gc-event-contracts.dll (name is subject to change))

The event layout definition

interface IGCStartTraceData

{

    int Count { get; }

    GCReason Reason { get; }

    int Depth { get; }

    // …

}

interface IGCEndTraceData

{

    int Count { get; }

    int Depth { get; }

    // …

}

Currently the layout definitions are in TraceEvent. In order to have both TraceEvent.dll (or some other tool that listens to ETW events) and the GLAD SDK refer to these event layouts, 
we need a common definition. TraceEvent.dll will implement getting the fields of events and GLAD will refer to these fields in order to calculate the GC information.

The event handlers in the parser

We need to expose the definition of event handlers to allow GLAD to hook up all the events it processes 
(we don’t want users to get a partially processed set which causes the GC information to not be filled in fully).

interface IGCEventParser

{

    event Action<IGCStartTraceData> GCStart;

    event Action<IGCEndTraceData> GCEnd;

    // other event handlers.

　   // this invokes the actual processing of the events

         bool Process();

}

These are in its own dll so TraceEvent.dll and glad.dll can both reference it.

Implementation (glad.dll (name is subject to change))

The following sections are for implementation and will be in glad.dll.

1) Delegates for ETW event processing

// GC*Data describes the ETW event layout.

delegate void OnGCStart(IGCStartTraceData data);

delegate void OnGCEnd(IGCEndTraceData data);

// delegates for other events.

class GCProcessor

{

    // user can overwrite this.   

    public virtual OnGCStartHandler(IGCStartTraceData data);

    public virtual OnGCEndHandler(IGCEndTraceData data);

    // other events.

}

2) The list of GCInformation that describes the processed info for all the GCs in a particular process.

GCInformation allows the user to attach additional info for each GC. This is useful internally for us to experiment with certain things before we publish it in GLAD. 
Eg, we are doing some additional processing, we can iterate on that some on our side; when it’s all tested we take it out of userData and publish it.

// This is the info for a particular GC

class GCInformation

{ 

    public bool isComplete;   

    public int gcIndex;

    public double durationMSec;

    // other GC fields

    // User might want to hang some info off of each GC.

    public object userData;

}

// This is the GCs for that particular process

class GCInformationPerProcess

{

    public int processID;

    public int currentGCIndex;

    public List<GCInformation> listGCInfo;

    // …

}

3) The class for the user to invoke GLAD’s processing.

class GCProcessor

{

    public virtual OnGCStartHandler(IGCStartTraceData data);

    public virtual OnGCEndHandler(IGCEndTraceData data);

    // event handers for other events.

　

    // hooks up all events that we process.

    public GCProcessor(IGCEventParser p)

    {

        p.GCStart += OnGCStartHandler;

        p.GCEnd += OnGCEndHandler;

        // other events.

    }

    // If you are not interested in processing any of the GC events, you

    // can simply get the results after processing.

    public List<GCInformationPerProcess> GetResults()

    {

        // return the list of processes with their GC info.

    }

    // If you are interested in additional processing, you can get the

    // current GC and do your processing there.

    public GCInformation GetCurrentGC(int ProcessID)

    {

        // returns the current GC in this process.

    }

}

4) Event to notify users of a GC sequence processing completion

// When we have processed the full sequence for a GC, we fire an event.

// sender would be of type GCInformationPerProcess.

delegate void GCCompletionEventHandler(object sender, EventArgs e);

class GCProcessor

{   

    public event EventHandler OnGCSequenceCompletedHandler;

}

User scenarios

1) User doesn’t do any processing of GC events on his own. But he’s interested in getting the GC completion notification via our event.

// ****in GLAD implementation****

// in the GCHeapStats ETW event processor

currentGC.isComplete = true;

OnGCSequenceCompletedHandler (GCInformationPerProcess, null);

// ****in user code****

void GCCompletionEventHandler(object sender, EventArgs e)

{

    GCInformationPerProcess gcProcess = sender as GCInformationPerProcess;

    GCInformation currentGC = gcProcess.listGCInfo[currentGCIndex];

    myLog.WriteLine(“Process {0}: GC {1} took {2}ms”, gcProcess.processID, currentGC.gcIndex, currentGC.durationMSec);

 

    if (current.durationMSec > 1000)

    {

        // stop collecting.

    }

}

// source is the event parser.

GCProcessor gcProcessor = new GCProcessor(source);

gcProcessor.OnGCSequenceCompletedHandler += GCCompletionEventHandler;

source.Process();

2) User wants to do some processing with selected GC events.

// ****in user code****

class MyGCProcessor : GCProcessor

{

    public MyGCProcessor (IGCEventParser p) : base (p)

    {

    }

    public override void OnGCStart(IGCStartTraceData data)

    {

        // Note for doing additional processing, you always need to call

        // base event handler first. This is important.

        base.OnGCStart(data);

        GCInformation gcInfo = GetCurrentGC(data.ProcessID);

        gcInfo.userData = new MyGCInformation();

        // do some additional processing here.

    }

}

https://devblogs.microsoft.com/dotnet/are-you-glad/