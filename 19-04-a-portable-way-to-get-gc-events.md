<h1>A portable way to get GC events in process and no admin privilege with 10 lines of code (and ability to dynamically enable/disable events)</h1>

I’ve been talking about doing managed heap performance analysis with ETW events for ages because ETW is just such a powerful tool. It has a well defined format so many components, from kernel modes to user mode ones, all emit ETW events which means you can have tools that just know how to parse the event format and correlate them. At Microsoft perf teams have analysis that tells you “this long request took X ms and X0 ms was due to a kmode component0, X1 ms was due to a umode component1 and etc” by interpreting ETW events. This is immensely useful.

However, you do need admin privilege to turn on ETW events which is not always desirable. Also I myself am moving part of my development to Linux due to the cross-plat nature of coreclr so I started to look at other eventing mechanisms more closely, from both GC’s and customers’ POV. I had heard of EventSource/EventListener before but had never used it myself. These had existed on desktop CLR for years and managed components were using them to fire events. In coreclr 2.2 we made it possible to obtain the CLR runtime events (eg. the GC events) via this mechanism. And I was talking to our diagnostics expert Noah Falk who mentioned it’s super simple to get the GC events if that’s all I cared about. Now, for GC’s own perf analysis, I absolutely do care about other events (mostly kernel events). But for customers, it’s usually sufficient to get just the informational level GC events. It turned out it’s literally 10 lines of code to just get the GC events, in process, in managed code –


```
class SimpleEventListener : EventListener
{
    public SimpleEventListener() {}

    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name.Equals("Microsoft-Windows-DotNETRuntime"))
            EnableEvents(eventSource, EventLevel.Informational, (EventKeywords)1);
    }
}
```

view rawGetGCEvents.cs hosted with ❤ by GitHub
And really the main part is just line 7 and 8.

(I was tempted to say “2 lines” in the title but I resisted)

These 2 lines say to enable keyword 1 (which is the keyword for GC), level Informational in the CLR provider (Microsoft-Windows-DotNETRuntime).

And you can choose to process any of these GC events. As a simple example, if I want to print out to the console each GC with its duration, I can add another method in my SimpleEventListener class:

```
long timeGCStart = 0;

protected override void OnEventWritten(EventWrittenEventArgs eventData)
{
    if (eventData.EventName.Contains("GCStart"))
    {
        timeGCStart = eventData.TimeStamp.Ticks;
    }
    else if (eventData.EventName.Contains("GCEnd"))
    {
        long timeGCEnd = eventData.TimeStamp.Ticks;
        long gcIndex = long.Parse(eventData.Payload[0].ToString());
        Console.WriteLine("GC#{0} took {1:f3}ms", 
            gcIndex, (double) (timeGCEnd - timeGCStart)/10.0/1000.0);
    }
}
```

view rawProcessGCEvents.cs hosted with ❤ by GitHub

You could print out eventData.EventName to see other GC events. The list is also documented on MSDN.

Of course on production you do not want to print to console. It would be an improvement to log to a file but why stop there when you could do things like dynamically enabling and disabling these events easily? Let’s say I want to disable GC events after the first 100 GCs, I can save the event source in OnEventSourceCreated to eventSourceDotNet and in OnEventWritten where I’m handling the GCEnd I can do:

```
        if (gcIndex >= 100)
            DisableEvents(eventSourceDotNet);
```
Then GC events are disabled. A very common thing perf folks do is “let’s log for X mins every Y hours” so they can use this to enable/disable whenever they need to.

You could also do stuff like “if I see > X gen2 GCs/minute, I will start logging GC events and perhaps even enable more events or even more providers to help me diagnose problems”.

Of course you don’t need to save anything to text logs, you can for example choose to write whatever info you get from these events to a buffer that gets flushed periodically to a location of your choice. Do whatever you like, the world is your oyster 

There are of course other event source providers. You can get the available ones in OnEventSourceCreated –

```
{
    // This shows all providers in your process.
    Console.WriteLine(eventSource.Name);
}
```
protected override void OnEventSourceCreated(EventSource eventSource)

In my minimal test that just allocates byte arrays and with the SimpleEventListener class I see 3 providers:

Copy
Microsoft-Windows-DotNETRuntime
System.Threading.Tasks.TplEventSource
System.Runtime
And of course you could get other runtime events from the DotNETRuntime provider if you like – I’ve included some other keywords in the full example below along with the output. Note that I did not need to create a thread – a thread is created for you if needed so you don’t need to worry about it. With ETW you’d need to handle that yourself.

```
using System;
using System.Diagnostics;
using System.Diagnostics.Tracing;
using System.IO;

namespace RuntimeEvents
{
    public class Program
    {
        enum ClrRuntimeEventKeywords
        {
            GC = 0x1,
            GCHandle = 0x2,
            Loader = 0x8,
            Jit = 0x10,
            Contention = 0x4000,
            Exceptions = 0x8000
        }

        // It takes one arg which is the keyword you want to enable
        // in the dotnet provider.
        // 
        // 0 means no events
        // 1 means GC
        // other possible keywords are listed above.
        static void Main(string[] args)
        {
            int keyword = int.Parse(args[0]);

            Console.WriteLine("KEYWORD= " + keyword);
            SimpleEventListener l = null;
            if (keyword != 0)
            {
                SimpleEventListener.keyword = keyword;
                l = new SimpleEventListener();
            }

            Stopwatch sw = new Stopwatch();

            sw.Start();
            Allocate();
            sw.Stop();
            Console.WriteLine("took {0}ms", sw.ElapsedMilliseconds);

            if (l != null)
            {
                Console.WriteLine("Total Count= " + l.countTotalEvents);
            }
        }
        static void Allocate()
        {
            for (int i = 0; i < 300_000_000; i++)
            {
                int[] x = new int[100];
            }
        }
    }

    class SimpleEventListener : EventListener
    {
        public ulong countTotalEvents = 0;
        public static int keyword;

        long timeGCStart = 0;
        EventSource eventSourceDotNet;

        public SimpleEventListener() {}
        
        // Called whenever an EventSource is created.
        protected override void OnEventSourceCreated(EventSource eventSource)
        {
            Console.WriteLine(eventSource.Name);
            if (eventSource.Name.Equals("Microsoft-Windows-DotNETRuntime"))
            {
                EnableEvents(eventSource, EventLevel.Informational, (EventKeywords)keyword);
                eventSourceDotNet = eventSource;
            }
        }
        // Called whenever an event is written.
        protected override void OnEventWritten(EventWrittenEventArgs eventData)
        {
            // Write the contents of the event to the console.
            if (eventData.EventName.Contains("GCStart"))
            {
                timeGCStart = eventData.TimeStamp.Ticks;
            }
            else if (eventData.EventName.Contains("GCEnd"))
            {
                long timeGCEnd = eventData.TimeStamp.Ticks;
                long gcIndex = long.Parse(eventData.Payload[0].ToString());
                Console.WriteLine("GC#{0} took {1:f3}ms", 
                    gcIndex, (double) (timeGCEnd - timeGCStart)/10.0/1000.0);

                if (gcIndex >= 5)
                    DisableEvents(eventSourceDotNet);
            }

            countTotalEvents++;
        }
    }
}
view rawfullGCEvents.cs hosted with ❤ by GitHub
Output

Copy
F:\coreclr-event\bin\tests\Windows_NT.x64.Release\Tests\Core_Root>corerun Collect0.exe 1
KEYWORD= 1
Microsoft-Windows-DotNETRuntime
System.Threading.Tasks.TplEventSource
System.Runtime
GC#1 took 0.677ms
GC#2 took 0.313ms
GC#3 took 0.030ms
GC#4 took 0.021ms
GC#5 took 0.018ms
took 6661ms
Total Count= 70
```

https://devblogs.microsoft.com/dotnet/a-portable-way-to-get-gc-events-in-process-and-no-admin-privilege-with-10-lines-of-code-and-ability-to-dynamically-enable-disable-events/