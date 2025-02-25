<h1>The history of the GC configs</h1>

Recently, Nick from Stack Overflow tweeted about his experience of using the .NET Core GC configs – he seemed quite happy with them (minus the fact they are not documented well which is something I’m talking to our doc folks about). I thought it’d be fun to tell you about the history of the GC configs ‘cause it’s almost the weekend and I want to contribute to your fun weekend reading.

I started working on the GC toward the end of .NET 2.0. At that time we had really, really few public GC configs. “Public” in this context means “officially supported”. I’m talking just gcServer and gcConcurrent, which you could specify as application configs and I think the retain VM one which was exposed as a startup flag STARTUP_HOARD_GC_VM, not an app config. Those of you who only worked with .NET Core may not have come across “application configs” – that’s a concept that only exists on .NET, not .NET Core. If you have an .exe called a.exe, and you have a file called a.exe.config in the same directory, then .NET will look at things you specify in this file under the runtime element for things like gcServer or some other non GC configs.

At that time the official ways to configure the CLR were:

App configs, under the runtime element
Startup flags when you load the CLR via Hosting API, strictly speaking, you could use some of the hosting APIs to customize other aspects of GC like providing memory for the GC instead of having GC acquire memory via the OS VirtualAlloc/VirtualFree APIs but I will not get into those – the only customer of those was SQLCLR AFAIK.
(There were actually also other ways like the machine config but I will also not get into those as they have the same idea as the app configs)

Of course there have always been the (now (fairly) famous) COMPlus environment variables. We used them only for internal testing – it’s easy to specify env vars and our testing framework read them, set them and unset them as needed. There were actually not many of those either – one example was GCSegmentSize that was heavily used in testing (to test the expand heap scenario) but not officially supported so we never documented them as app configs.

I was told that env vars were not a good customer facing way to config things because people tend to set them and forget to unset them and then later they wonder why they were seeing some unexpected behavior. And I did see that happen with some internal teams so this seemed like a reasonable reason.

Startup flags are just a hosting API thing and hosting API was something few people heard of and way fewer used. You could say things like you want to start the runtime with Server GC and domain neutral. It’s a native API and most of our customers refused to use it when they were recommended to try. Today I’m aware of only one team who’s actively using it – not surprisingly many people on that team used to work on SQLCLR 😛

For things you could specify as app configs you could also specify them with env vars or even registry values because on .NET our internal API to read these configs always check all 3 places. While we had a different kind of attitude toward configs you could specific via app config, which were considered officially supported, implementation wise this was great because devs didn’t need to worry about which place the config would be read from – they knew if they added a new config in clrconfigvalues.h it could be specified via any of the 3 ways automatically.

During the .NET 4.x timeframe We needed to add public configs for things like CPU group (we started seeing machines with > 64 procs) or creating objects of >2gb due to customer requests. Very few customers used these configs. So they could be thought of as special case configs, in other words, the majority of the scenarios were run with no configs aside from the gcServer/gcConcurrent ones.

I was pretty wary of adding new public configs. Adding internal ones was one thing but actually telling folks about them means we’d basically be saying we are supporting them forever – in the older versions of .NET the compatibility bar was ultra high. And tooling was of course not as advanced then so perf analysis was harder to do (most of the GC configs were for perf).

For a long time folks used the 2 major flavors of the GC, Server and Workstation, mostly according to the way they were designed. But you know how the rest of this story goes – folks didn’t exactly use them “as designed originally” anymore. And as the GC diagnostic space also advanced customers were able to debug and understand GC perf better and also used .NET on larger, more stressful and more diverse scenarios. So there was more and more desire from them to do more configuration on their own.

Good thing was Microsoft internally had plenty of customers who had very stressful workloads that called for configuration so I was able to test on these stressful real world scenarios. Around the time of .NET 4.6 I started adding configs more aggressively. One of our 1st party customers was running a scenario with many managed processes. They had configed some to use Server GC and others to use Workstation. But there was nothing inbetween. This was when configs like GCHeapCount/GCNoAffinitize/GCHeapAffinitizeMask were added.

Around that time we also open sourced coreclr. The distinction of “officially supported configs” vs internal only configs was still there – in theory that line had become a little blurry because our customers could see what internal configs we had 🙂 but it also took time for Core adoption so I wasn’t aware of really anyone who was using internal only configs. Also we changed the way config values were read – we no longer had the “one API reads them all” so today on Core where the “official configs” are specified via the runtimeconfig.json, you’d need to use a different API and specify the name in the json and the name for the env var if you want to read from both.

My development was still on CLR mostly, just because we had very few workloads on CoreCLR at that time and being able to try things on large/stressful workloads was a very valuable thing. Around this time I added a few more configs for various scenarios – notable ones are GCLOHTheshold and GCHighMemPercent. A team had their product running in a process coexisting with a much large process on the same machine which had a lot of memory. So the default memory load that GC considered as “high memory load situation”, which was 90%, worked well for the much larger process but not for them. When there’s 10% physical memory left that was still a huge amount for their process so I added this for them to specify a higher value (they specified 97 or 98) which meant their process didn’t need to do full compacting GCs nearly as often.

Core 3.0 was when I unified the source between .NET and .NET Core so all the configs (“internal” or not) from .NET were made available on Core as well. The json way is obviously the official way to specify a config but it appeared specifying configs via env vars was becoming more common, especially with folks who work on scenarios with high perf requirements. I know quite a few internal and external customers use them (and have yet to hear any incidents that involved setting an env var in an undesirable fashion). A few more GC configs were added during Core 3.0 – GCHeapHardLimit, GCLargePages, GCHeapAffinitizeRanges and etc.

One thing that took folks (who used env vars) by surprise was the number you specific for a config in an env var format is interpreted as a hex number, not decimal. As far as why it was this way, it completely predates my time on the runtime team… since everyone remembered this for sure after they used it wrong the first time 😛 and it was an internal only thing, no one bothered to change it.

I am still of the opinion that the officially supported configs should not require you to have internal GC knowledge. Of course internal is up for interpretation – some people might view anything beyond gcServer as internal knowledge. I’m interpreting “not having internal GC knowledge” in this context as “only having general perf knowledge to influence the GC”. For example, GCHeapHardLimit tells the GC how much memory it’s allowed to use; GCHeapCount tells the GC how many cores it’s allowed to use. Memory/CPU usage are general perf knowledge that one already needs to have if they work on perf. GCLOHThreshold is actually violating this policy somewhat so it’s something we’d like to dynamically tune in GC instead of having users specify a number. But that’s work we haven’t done yet.

I don’t want to have configs that would need users to config things like “if this generation’s free list ratio or survival rate is > some threshold I would choose this particular GC to handle collections on that generation; but use this other GC to collect other generations”. That to me is definitely “requiring GC internal knowledge”.

So there you have it – the history of the GC configs in .NET/.NET Core.

https://devblogs.microsoft.com/dotnet/the-history-of-the-gc-configs/

最近，Stack Overflow 的 Nick 在推特上分享了他对使用 .NET Core GC 配置的经验——他似乎对这些配置感到很满意（除了它们的文档不够完善这一点，我正在和我们的文档团队讨论这个问题）。我想趁周末快到的时候，给大家讲讲 GC 配置的历史，为你的周末阅读增添一点乐趣。

我在 .NET 2.0 即将结束时开始从事 GC 的工作。当时，我们真正公开的 GC 配置非常少。“公开”在这里的意思是“官方支持”。我当时只记得有 gcServer 和 gcConcurrent，你可以通过应用程序配置文件指定它们。还有一个 retain VM，它是通过启动标志 STARTUP_HOARD_GC_VM 暴露的，而不是应用程序配置文件。如果你只用过 .NET Core，可能没有接触过“应用程序配置文件”——这是 .NET 中的概念，而不是 .NET Core 的。如果你有一个名为 a.exe 的可执行文件，并且在同一个目录下有一个名为 a.exe.config 的文件，那么 .NET 会在该文件的 <runtime> 元素下查找你指定的内容，例如 gcServer 或其他非 GC 配置。

当时的官方方式来配置 CLR 包括：

应用程序配置文件中的 <runtime> 元素。
使用托管 API 加载 CLR 时的启动标志。严格来说，某些托管 API 可以用于自定义 GC 的其他方面，例如为 GC 提供内存而不是让 GC 通过操作系统调用 VirtualAlloc/VirtualFree 来获取内存，但我不打算深入这些内容——据我所知，SQLCLR 是唯一使用这些功能的客户。 （实际上还有一些其他方式，比如机器配置文件，但它们与应用程序配置文件的概念相同，我也不再赘述。）
当然，一直以来还有那些现在（相对）著名的 COMPlus 环境变量。我们只在内部测试中使用它们——设置环境变量很方便，我们的测试框架会读取、设置和取消它们。其实这些变量的数量也不多——一个例子是 GCSegmentSize，它在测试中被广泛使用（用于测试堆扩展场景），但由于不是正式支持的，我们从未将其作为应用程序配置文档化。

有人告诉我，环境变量不是一个面向客户的好配置方式，因为人们倾向于设置它们后忘记取消，后来又不明白为什么会出现一些意外行为。我也确实看到一些内部团队发生过这种情况，所以这似乎是一个合理的理由。

启动标志只是一个托管 API 的功能，而托管 API 是很少有人听说或使用的。你可以通过它指定类似“使用 Server GC 启动运行时并启用域中立”这样的选项。这是一个本地 API，当用户被建议尝试时，大多数客户都拒绝使用它。今天我知道只有一个团队还在积极使用它——不出意外，这个团队的很多人曾经从事 SQLCLR 工作 😛

对于可以通过应用程序配置文件指定的内容，你也可以通过环境变量甚至注册表值来指定，因为在 .NET 中，我们的内部 API 在读取这些配置时总是检查这三个位置。虽然我们在态度上对通过应用程序配置文件指定的配置有所不同，这些被视为官方支持的配置，但从实现角度来看，这非常好，因为开发人员不必担心配置从哪个地方读取——他们知道如果在 clrconfigvalues.h 中添加一个新的配置项，就可以通过这三种方式自动指定。

在 .NET 4.x 的时期，我们需要根据客户需求添加公共配置，例如 CPU 组（我们开始看到具有超过 64 个处理器的机器）或创建大于 2GB 的对象。很少有客户使用这些配置，因此可以将它们视为特殊情况的配置，换句话说，大多数场景仍然仅使用 gcServer 和 gcConcurrent 这两个配置。

我对添加新的公共配置一直很谨慎。添加内部配置是一回事，但告诉人们它们的存在意味着我们基本上承诺永久支持它们——在旧版本的 .NET 中，兼容性标准非常高。而且当时的工具也没有那么先进，性能分析更难进行（大多数 GC 配置都是为了性能优化）。

很长一段时间里，人们主要根据设计初衷使用 GC 的两种主要模式：Server 和 Workstation。但你知道这个故事接下来的发展——人们不再完全按照最初的设计使用它们。随着 GC 诊断领域的进步，客户能够更好地调试和理解 GC 性能，并且在更大、更具压力和更多样化的场景中使用 .NET。因此，他们对自行配置的需求越来越多。

幸运的是，微软内部有许多客户拥有非常紧张的工作负载，这让我能够在这些真实的高压场景中进行测试。大约在 .NET 4.6 的时候，我开始更积极地添加配置。我们的一个第一方客户在一个场景中运行了许多托管进程。他们配置了一些进程使用 Server GC，另一些使用 Workstation GC，但两者之间没有任何中间状态。这就是 GCHeapCount、GCNoAffinitize 和 GCHeapAffinitizeMask 等配置被添加的原因。

大约在同一时间，我们开源了 coreclr。尽管“官方支持的配置”与仅供内部使用的配置之间的区别仍然存在，但理论上这条界限已经变得有些模糊，因为客户可以看到我们有哪些内部配置 🙂 不过由于 Core 的采用需要时间，我当时并没有意识到有人在使用仅供内部的配置。此外，我们改变了读取配置值的方式——我们不再有“一个 API 读取所有”的机制，因此在 Core 中，官方配置通过 runtimeconfig.json 指定时，你需要使用不同的 API，并在 JSON 和环境变量中分别指定名称，如果你想同时从两者读取。

我的开发工作仍然主要集中在 CLR 上，因为在那时 CoreCLR 的工作负载很少，能够在大型/紧张的工作负载上尝试新东西是非常有价值的事情。大约在这个时候，我为各种场景添加了几个新的配置——值得注意的是 GCLOHThreshold 和 GCHighMemPercent。一个团队的产品运行在一个进程中，该进程与同一台机器上的另一个更大的进程共存，而那个更大的进程占用了很多内存。因此，GC 认为“高内存负载情况”的默认阈值（90%）对较大的进程适用，但对他们的进程不适用。当只剩下 10% 的物理内存时，这对他们的进程来说仍然是一个巨大的数量，所以我为他们添加了这个配置，允许他们指定更高的值（他们指定了 97 或 98），这意味着他们的进程不需要像以前那样频繁地进行全量压缩 GC。

Core 3.0 是我统一 .NET 和 .NET Core 源代码的时间点，因此 .NET 中的所有配置（无论是“内部”还是“外部”）都在 Core 中可用。JSON 方式显然是指定配置的官方方式，但通过环境变量指定配置的方式正变得越来越普遍，尤其是在处理高性能需求场景的人群中。我知道许多内部和外部客户都在使用这种方式（并且尚未听说过任何因错误设置环境变量而导致的问题）。在 Core 3.0 期间，还添加了一些新的 GC 配置，例如 GCHeapHardLimit、GCLargePages 和 GCHeapAffinitizeRanges 等。

让使用环境变量的人感到惊讶的一件事是，在环境变量格式中为配置指定的数字会被解释为十六进制数，而不是十进制数。至于为什么会这样，这完全超出了我加入运行时团队之前的时间……由于每个人在第一次使用错误后都会记住这一点 😛 并且它只是内部使用的，所以没有人费心去改变它。

我仍然认为，官方支持的配置不应该要求用户具备内部 GC 知识。当然，“内部”是可以解释的——有些人可能认为任何超出 gcServer 的内容都属于内部知识。在此上下文中，我将“不具备内部 GC 知识”解释为“只需具备影响 GC 的一般性能知识即可”。例如，GCHeapHardLimit 告诉 GC 它可以使用的内存量；GCHeapCount 告诉 GC 它可以使用的内核数。内存/CPU 使用是通用的性能知识，如果你从事性能优化工作，已经需要具备这些知识。GCLOHThreshold 实际上稍微违反了这一政策，所以我们希望在 GC 内部动态调整它，而不是让用户指定一个数字。但这是一项尚未完成的工作。

我不想让用户配置类似“如果这一代的空闲列表比例或存活率 > 某个阈值，我会选择特定的 GC 来处理该代的收集；但对其他代使用另一种 GC”这样的内容。对我来说，这显然是“要求具备 GC 内部知识”。

所以这就是 .NET/.NET Core 中 GC 配置的历史。