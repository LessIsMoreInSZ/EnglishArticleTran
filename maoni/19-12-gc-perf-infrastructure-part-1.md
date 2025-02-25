<h1>GC Perf Infrastructure – Part 1</h1>

We open sourced our new GC Perf Infrastructure! It’s now part of the dotnet performance repo. I’ve been meaning to write about it ‘cause some curious minds had been asking when they could use it after I blogged about it last time but didn’t get around to it till now.

First of all, let me point out that the target audience of this infra, aside from the obvious (ie, those who make performance changes to the GC), are folks need to do in-depth analysis of GC/managed memory performance and/or to build automation around it. So it assumes you already have a fair amount of knowledge what to look for with the analysis.

Secondly, there are a lot of moving parts in the infra and since it’s still under development I wouldn’t be surprised if you hit problems when you try to use it. Please be patient with us as we work through the issues! We don’t have a whole lot of resources so we may not be able to get to them right away. And of course if you want to contribute it would be most appreciated. I know many people who are reading this are passionate about perf analysis and have done a ton of work to build/improve perf analysis for .NET, whether in your own tooling or other people’s. And contributing to perf analysis is a fantastic way to learn about GC tuning if you are looking to start somewhere. So I would strongly encourage you to contribute!

Topology

We discussed whether we wanted to open source this in its own repo and the conclusion we wouldn’t mostly just due to logistics reasons so this became part of the perf repo under the “src/benchmarks/gc” directory (which I’ll refer to as the root directory). It doesn’t depend on anything outside of this directory which means you don’t need to build anything outside of it if you just want to use the GC perf infra part.

The readme.md in the root directory describes the general workflow and basic usage. More documentation can be found in the docs directory.

There are 2 major components of the infra –

Running perf benchmarks

This runs our own perf benchmarks – this is for folks who need to actually make perf changes to the GC. It provides the following functionalities –

Specifying different commandline args to generate different perf characteristics in the tests, eg, different surv ratios for SOH/LOH and different pinning ratios;
Specifying builds to compare against;
Specifying different environments, eg, different env vars to specify GC configs, running in containers or high memory load situations;
Specifying different options to collect traces with, eg, GCCollectOnly or ThreadTime.
You specify all these in what we call a bench file (it’s a .yaml file but really could be anything – we just chose .yaml). We also provide configurations for the basic perf scenarios so when you make changes those should be run to make sure things don’t regress.

You don’t have to run our tests – you could run whatever you like as long as you can specify it as a commandline program, and still take advantage of the rest of what we provide like running in a container.

This is documented in the readme and I will be talking about this in more detail in one of the future blog entries.

Source for this is in the exec dir.

Analyzing perf

This can be used without the running part at all. If you already collected perf traces, you can use this to analyze them. I’d imagine more folks would be interested in this than the running part so I’ll devote more content to analysis. In the last GC perf infra post I already talked about things you could do using Jupyter Notebook (I’ll be showing more examples with the actual code in the upcoming blog entries). This time I’ll focus on actually setting things up and using the commands we provide. Feel free to try it out now that it’s out there.

Source for this is in the analysis dir.

Analysis setup

After you clone the dotnet performance repo, you’ll see the readme in the gc infra root dir. Setup is detailed in that doc. If you just want the analysis piece you don’t need to do all of the setup steps there. The only steps you need are –

Install python. 3.7 is the minimal required version and recommended version. 3.8 has problems with Jupyter Notebook. I wanted to point this out because 3.8 is the latest release version on python’s page.
Install the python libraries needed – you can install this via “py -m pip install -r src/requirements.txt” as the readme says and if no errors occur, great; but you might get errors with pythonnet which is mandatory for analysis. In fact installing pythonnet can be so troublesome that we devoted a whole doc just for it. I hope one day there are enough good c# charting libraries and c# works in Jupyter Notebook inside VSCode so we no longer need pythonnet.
Build the c# analysis library by running “dotnet publish” in the src\analysis\managed-lib dir.
Specify what to analyze

Let’s say you’ve collected an ETW trace (this can be from .NET or .NET Core) and want to analyze it, you’ll need to tell the infra which process is of interest to you (on Linux you collect the events for the process of interest with dotnet-trace but since the infra works on both Windows and Linux this is the same step you’d perform). Specifying the process to analyze means simply writing a .yaml file that we call the “test status file”. From the readme, the test status file you write just for analysis only needs these 3 lines –

success: true

trace_file_name: x.etl # A relative path. Should generally match the name of this file.

process_id: 1234 # If you don’t know this, use the print-processes command for a list


You might wonder why you need to specify the “success: true” line at all – this is simply because the infra can also be used to analyze the results of running tests with it and when you run lots of tests and analyze their results in automation we’d look for this line and only analyze the ones that succeeded.

You may already know the PID of the process you want to analyze via other tools like PerfView but we aim to have the infra used standalone without having to run other tools so there’s a command that prints out the PIDs of processes a trace contains.

We really wanted to have the infra provide meaningful built-in help so when you wonder how to do something you could generally find it in its help. To get the list of all commands simply ask for the top level help in the root dir –

C:\perf\src\benchmarks\gc>py . help

Read README.md first. For help with an individual command, use py . command-name --help. (You can also pass --help --hidden to see hidden arguments.)

run commands

[omitted]

analysis commands

Commands for analyzing test results (trace files). To compare a small number of configs, use diff. To compare many, use chart-configs. For detailed analysis of a single trace, use analyze-single or chart-individual-gcs.

analyze-single: Given a single trace, print run metrics and optionally metrics for individual GCs.

analyze-single-gc: Print detailed info about a single GC within a single trace.

[more output omitted and did some formatting of the output]

 

(I apologize for the formatting – it amazes me that that we don’t seem to have a decent html editing program for blogging and writing a blog mostly consists of manually writing html ourselves which is really painful)

As the top level help says you can get help with specific commands. So we’ll follow that suggestion and do

C:\perf\src\benchmarks\gc>py . help print-processes

Print all process PIDs and names from a trace file.

arg name	arg type	description
–name-regex	any string	Regular expression used to filter processes by their name
–hide-threads	true or false	Don’t show threads for each process
[more output omitted; I also did some formatting to get rid of some columns so the lines are not too long]

As an example, I purposefully chose a test that I know is unsuitable to be run with Server GC ‘cause it only has one thread so I’m expecting to see some heap imbalance. I know the imbalance will occur when we mark older generation objects holding onto young gen objects so I’ll use the chart-individual-gcs command to show me how long each heap took to mark those.

C:\perf\src\benchmarks\gc>py . chart-individual-gcs C:\traces\fragment\fragment.yaml –x-single-gc-metric Index –y-single-heap-metrics MarkOlderMSec

This will show 8 heaps. Consider passing --show-n-heaps.

markold-time

Sure enough one of the heaps always takes significantly longer to mark young gen objects referenced by older gen objects, and to make sure it’s not because of some other factors I also looked at how much is promoted per heap –


C:\perf\src\benchmarks\gc>py . chart-individual-gcs C:\traces\fragment\fragment.yaml –x-single-gc-metric Index –y-single-heap-metrics MarkOlderPromotedMB

This will show 8 heaps. Consider passing --show-n-heaps.

markold-promoted This confirms the theory – it’s because we marked significantly more with one heap which caused that heap to spend significantly longer in marking.

This trace was taken with the latest version of the desktop CLR. In the current version of coreclr we are able to handle this situation better but I’ll save that for another day since today I wanted to focus on tooling.

There’s an example.md that shows examples of using some of the commands. Note that the join analysis is not checked in just yet – the PR is out and I wanted to spend more time on the CR before merging it.

https://devblogs.microsoft.com/dotnet/gc-perf-infrastructure-part-1/

我们开源了新的 GC 性能基础设施！它现在已经成为 dotnet performance 仓库的一部分。我一直想写一篇关于它的文章，因为在上次我发布博客后，一些好奇的人问我什么时候可以使用它，但我一直没时间写，直到现在。

首先，我想指出这个基础设施的目标受众，除了显而易见的（即那些需要对 GC 进行性能改进的人），还包括那些需要深入分析 GC/托管内存性能和/或围绕其构建自动化的人员。因此，它假设你已经具备相当多的分析知识。

其次，这个基础设施有很多组成部分，并且由于它仍在开发中，所以如果你在使用时遇到问题，请不要感到惊讶。请耐心等待我们解决这些问题！我们的资源有限，可能无法立即处理所有问题。当然，如果你愿意贡献代码，我们将非常感激。我知道许多正在阅读这篇文章的人都对性能分析充满热情，并为 .NET 的性能分析做了大量工作，无论是通过自己的工具还是其他人的工具。如果你正在寻找一个切入点来学习 GC 调优，那么为性能分析做出贡献是一个极好的方式。因此，我强烈鼓励你参与贡献！

架构
我们讨论过是否要在单独的仓库中开源这个基础设施，但由于后勤原因，最终决定将其作为性能仓库的一部分，位于 src/benchmarks/gc 目录下（我将称其为根目录）。它不依赖于该目录之外的任何内容，这意味着如果你只想使用 GC 性能基础设施部分，不需要构建其他内容。

根目录中的 readme.md 描述了通用的工作流程和基本用法。更多文档可以在 docs 目录中找到。

这个基础设施主要有两个组成部分：

1. 运行性能基准测试
这部分运行我们自己的性能基准测试——这是为那些需要实际对 GC 进行性能改进的人准备的。它提供了以下功能：

指定不同的命令行参数以生成不同的性能特征，例如 SOH/LOH 的不同存活率和不同的固定比例。
指定要比较的构建版本。
指定不同的环境，例如通过环境变量指定 GC 配置、在容器中运行或高内存负载情况下运行。
指定不同的选项以收集跟踪数据，例如 GCCollectOnly 或 ThreadTime。
你可以在我们称为“bench 文件”（这是一个 .yaml 文件，但实际上可以是任何格式——我们只是选择了 .yaml）中指定所有这些内容。我们还为基本的性能场景提供了配置，因此当你进行更改时，应该运行这些配置以确保性能不会退化。

你不必运行我们的测试——只要你能将其指定为命令行程序，你可以运行任何你喜欢的内容，并仍然利用我们提供的其他功能，例如在容器中运行。

这部分的源代码位于 exec 目录中。

2. 分析性能
这部分可以独立于运行部分使用。如果你已经收集了性能跟踪数据，你可以使用它来分析这些数据。我认为更多的人会对分析部分感兴趣，所以我将投入更多内容来讨论分析。在上一篇关于 GC 性能基础设施的文章中，我已经谈到了如何使用 Jupyter Notebook（在未来的博客文章中，我会展示更多带有实际代码的例子）。这次我将重点介绍如何设置和使用我们提供的命令。既然它已经开源，欢迎尝试。

这部分的源代码位于 analysis 目录中。

分析设置
克隆 dotnet performance 仓库后，你会在 GC 基础设施的根目录中看到 readme 文件。设置细节在该文档中有详细说明。如果你只需要分析部分，不需要完成所有的设置步骤。你需要的唯一步骤是：

安装 Python。最低要求版本是 3.7，推荐版本也是 3.7。Python 3.8 在 Jupyter Notebook 中存在问题。我想指出这一点，因为 Python 3.8 是 Python 官方页面上的最新发布版本。
安装所需的 Python 库——可以通过 py -m pip install -r src/requirements.txt 安装，如果没有任何错误发生，那就很好；但你可能会在安装 pythonnet 时遇到问题，这是分析所必需的。事实上，安装 pythonnet 可能非常麻烦，所以我们专门为它编写了一个文档。我希望有一天有足够的优秀的 C# 图表库，并且 C# 能够在 VSCode 中的 Jupyter Notebook 中运行，这样我们就不再需要 pythonnet。
在 src\analysis\managed-lib 目录中运行 dotnet publish 来构建 C# 分析库。
指定要分析的内容
假设你已经收集了一个 ETW 跟踪文件（这可以来自 .NET 或 .NET Core），并且想要分析它，你需要告诉基础设施你感兴趣的进程是什么（在 Linux 上，你可以使用 dotnet-trace 收集目标进程的事件，但由于基础设施同时适用于 Windows 和 Linux，这是一样的步骤）。指定要分析的进程只需编写一个 .yaml 文件，我们称之为“测试状态文件”。根据 readme 的说明，仅用于分析的测试状态文件只需要以下三行：

yaml
深色版本
success: true

trace_file_name: x.etl # 相对路径。通常应与该文件名匹配。

process_id: 1234 # 如果你不知道这个值，可以使用 print-processes 命令列出进程。
你可能会疑惑为什么需要指定 success: true 这一行——这是因为基础设施还可以用于分析运行测试的结果，当你运行大量测试并自动化分析它们的结果时，我们会查找这一行，并只分析成功的测试。

你可能已经通过其他工具（如 PerfView）知道了要分析的进程的 PID，但我们希望基础设施能够独立使用，而不必运行其他工具，因此有一个命令可以列出跟踪文件中包含的进程的 PID。

我们真的很希望基础设施提供有意义的内置帮助，因此当你想知道如何做某事时，通常可以在其帮助中找到答案。要获取所有命令的列表，只需在根目录中请求顶级帮助：

bash
深色版本
C:\perf\src\benchmarks\gc>py . help
正如顶级帮助所说，你可以获取特定命令的帮助。因此，我们将按照建议执行以下操作：

bash
深色版本
C:\perf\src\benchmarks\gc>py . help print-processes
输出示例：

plaintext
深色版本
从跟踪文件中打印所有进程的 PID 和名称。

参数名称 参数类型 描述
--name-regex 任意字符串 用于按名称过滤进程的正则表达式
--hide-threads true 或 false 不显示每个进程的线程
作为一个例子，我故意选择了一个不适合使用 Server GC 的测试，因为它只有一个线程，所以我预计会看到堆不平衡的情况。我知道这种不平衡会在我们标记旧代对象引用年轻代对象时发生，因此我将使用 chart-individual-gcs 命令来显示每个堆标记这些对象所需的时间。

bash
深色版本
C:\perf\src\benchmarks\gc>py . chart-individual-gcs C:\traces\fragment\fragment.yaml --x-single-gc-metric Index --y-single-heap-metrics MarkOlderMSec
这将显示 8 个堆。考虑传递 --show-n-heaps。

markold-time

果然，其中一个堆总是花费显著更长的时间来标记年轻代对象被旧代对象引用的情况。为了确保这不是由于其他因素，我还查看了每个堆的提升量：

bash
深色版本
C:\perf\src\benchmarks\gc>py . chart-individual-gcs C:\traces\fragment\fragment.yaml --x-single-gc-metric Index --y-single-heap-metrics MarkOlderPromotedMB
这确认了我的理论——因为我们用一个堆标记了显著更多的对象，导致该堆在标记过程中花费了显著更长的时间。

这个跟踪文件是使用最新版本的桌面 CLR 获取的。在当前版本的 CoreCLR 中，我们能够更好地处理这种情况，但今天我想专注于工具，所以我会留到另一天再讨论。

有一个 example.md 文件展示了使用一些命令的示例。注意，联合分析尚未提交——PR 已经发出，我想花更多时间在代码审查上，然后再合并它。