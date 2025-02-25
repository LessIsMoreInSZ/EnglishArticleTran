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

我一直在谈论使用 ETW（事件跟踪 for Windows）事件进行托管堆性能分析，因为 ETW 是一个非常强大的工具。它具有明确定义的格式，从内核模式到用户模式的各种组件都会生成 ETW 事件，这意味着你可以拥有能够解析这些事件格式并关联它们的工具。在微软的性能团队中，通过解释 ETW 事件，可以告诉你“这个长时间请求花费了 X 毫秒，其中 X0 毫秒是由于内核模式组件0，X1 毫秒是由于用户模式组件1 等等”。这非常有用。

然而，启用 ETW 事件需要管理员权限，这并不总是理想的。另外，由于 CoreCLR 的跨平台特性，我自己也开始将部分开发工作转移到 Linux 上，因此我开始更深入地研究其他事件机制，从 GC 和客户的角度来看。我之前听说过 EventSource/EventListener，但从未亲自使用过。这些机制已经在桌面 CLR 中存在多年，并且托管组件一直使用它们来触发事件。在 CoreCLR 2.2 中，我们使通过这种机制获取 CLR 运行时事件（例如 GC 事件）成为可能。我和我们的诊断专家 Noah Falk 谈话时，他提到如果我只关心 GC 事件，获取它们是非常简单的。对于 GC 自身的性能分析，我绝对还需要其他事件（主要是内核事件）。但对于客户来说，通常只需要获取信息级别的 GC 事件就足够了。事实证明，只需 10 行代码就可以在进程内、用托管代码获取 GC 事件——

csharp
深色版本
class SimpleEventListener : EventListener
{
    public SimpleEventListener() {}

    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        if (eventSource.Name.Equals("Microsoft-Windows-DotNETRuntime"))
            EnableEvents(eventSource, EventLevel.Informational, (EventKeywords)1);
    }
}
实际上，主要部分只是第 7 和第 8 行。

（我本想在标题中说“两行”，但我忍住了）

这两行表示启用 CLR 提供程序（Microsoft-Windows-DotNETRuntime）中的关键字 1（GC 的关键字），级别为 Informational。

你可以选择处理这些 GC 事件中的任何一个。作为一个简单的例子，如果我想在控制台打印每次 GC 及其持续时间，可以在 SimpleEventListener 类中添加另一个方法：

csharp
深色版本
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
            gcIndex, (double)(timeGCEnd - timeGCStart) / 10.0 / 1000.0);
    }
}
你可以打印 eventData.EventName 来查看其他 GC 事件。事件列表也在 MSDN 上有文档说明。

当然，在生产环境中你不应该打印到控制台。记录到文件会是一个改进，但既然可以动态启用和禁用这些事件，为什么止步于此呢？假设我希望在前 100 次 GC 后禁用 GC 事件，我可以在 OnEventSourceCreated 中保存事件源到 eventSourceDotNet，并在 OnEventWritten 处理 GCEnd 时执行以下操作：

csharp
深色版本
if (gcIndex >= 100)
    DisableEvents(eventSourceDotNet);
然后 GC 事件就会被禁用。性能工程师经常做的一件事是“每 Y 小时记录 X 分钟”，所以他们可以随时根据需要启用或禁用事件。

你还可以实现类似的功能，比如“如果我看到每分钟 > X 次第 2 代 GC，我将开始记录 GC 事件，并可能启用更多事件甚至更多提供程序以帮助诊断问题”。

当然，你不需要将任何内容保存到文本日志中，你可以选择将从这些事件中获取的信息写入缓冲区，并定期刷新到你选择的位置。随心所欲吧，世界是你的牡蛎 😊

当然还有其他事件源提供程序。你可以在 OnEventSourceCreated 中获取可用的提供程序：

csharp
深色版本
protected override void OnEventSourceCreated(EventSource eventSource)
{
    // 这将显示进程中所有提供程序。
    Console.WriteLine(eventSource.Name);
}
在我的最小测试中，仅仅分配字节数组并使用 SimpleEventListener 类时，我看到了 3 个提供程序：

深色版本
Microsoft-Windows-DotNETRuntime
System.Threading.Tasks.TplEventSource
System.Runtime
当然，如果你愿意，你也可以从 DotNETRuntime 提供程序中获取其他运行时事件——我在下面的完整示例中包含了一些其他关键字及其输出。注意，我不需要创建线程——如果需要，线程会自动为你创建，因此你不必担心它。而使用 ETW 时，你需要自己处理线程。

csharp
深色版本
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

        protected override void OnEventSourceCreated(EventSource eventSource)
        {
            Console.WriteLine(eventSource.Name);
            if (eventSource.Name.Equals("Microsoft-Windows-DotNETRuntime"))
            {
                EnableEvents(eventSource, EventLevel.Informational, (EventKeywords)keyword);
                eventSourceDotNet = eventSource;
            }
        }

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
                    gcIndex, (double)(timeGCEnd - timeGCStart) / 10.0 / 1000.0);

                if (gcIndex >= 5)
                    DisableEvents(eventSourceDotNet);
            }

            countTotalEvents++;
        }
    }
}
输出：

深色版本
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
代码模式