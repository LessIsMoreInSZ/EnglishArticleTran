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