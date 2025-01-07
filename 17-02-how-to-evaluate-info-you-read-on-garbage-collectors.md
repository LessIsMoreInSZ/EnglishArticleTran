<h1>How to evaluate info you read on garbage collectors</h1>

Just a word before I actually start this blog entry – I apologize for approving some of the comments so late – it appears that our blogs’ policy has changed and 
it would make some comments as pending without obvious reasons to me.

Also as one of the ways to support the community I was thinking I could have a specific time on a regular cadence (could be an hour every month to start with)
 when I would answer questions on the .NET GC. I’m thinking perhaps gitter is the best forum but I am not sure – I don’t know how popular it is with .NET developers 
 (that care about memory performance) so if you have any suggestions please post them in comments.

From time to time I get someone that tells me “Hey Maoni, I just read this GC article that says their GC is so great! I think it’s better than your GC!”. 
Then I read the article and found out that it said nothing that actually indicated their GC is better than ours, in fact, 
it’s very often the case it didn’t even say nearly enough to make an assessment on that GC’s performance, much less to do a comparison with another GC. 
So I thought I’d write something that helps to explain how to evaluate information you read on GCs.

First of all, comparing GC performance is definitely non trivial work. When you run even a micro benchmark, it can easily involve enough from the non GC part of a runtime. 
So you are actually not comparing just the GCs unless you isolate it down to the parts that’s related to GC perf. 
I remember one time we had someone who showed us a comparison on a benchmark between our runtime and another runtime and it was spending much more time in GC in our runtime. 
It turned out it was due to our codegen making a decision that caused us to survive a lot more in our runtime so GC had to work a lot harder.

When you compare something that’s more than a micro benchmark, eg, a web server macro benchmark we are just talking comparing frameworks at this point, 
it’s a long way from drawing any conclusion on the GC performance alone. 
One framework could have a completely different allocation and survival pattern from the other which means one can impose a very different amount of work on GC from the other. 
For example, one framework could choose to allocate much fewer objects or tend to survive much fewer because of the programming patterns in that framework 
so it simply doesn’t need a stellar GC to manage memory.

Even when you have isolated it to a point that you are mostly comparing the GC performance, as anyone who’s even slightly familiar with GCs, 
you know there are tons of tradeoffs to make in a GC. Without actually understand the architecture, the mechanisms and the policy of a GC,
 it’s far from complete to run a few tests to determine what a GC’s performance is. For example, 
 one GC could have made a choice to optimize for generational behavior while another may not have (could be because the framework simply does not exhibit generational behavior).

When you understand the architecture of a GC you know its limits so you can make a statement about what it’s capable of in theory. 
Of course 2 GCs with the same architecture can still have dramatically different performance on the same exact test 
(let’s say you’ve isolated enough to make such a test) because how optimized the implementation is – on the hot path, 
one could be using a more clever way to implement something while the other falls short.

At this point I hope you realize how careful you need to be when you are not doing experiments yourself and simply reading some information written on a GC.

If a GC article explains how it does something (like an architectural doc, or a blog entry that explains how the GC handles a specific scenario), 
it’s totally fine to read the description. But without some knowledge on how effective/innovative/sophisticated this allows a GC to be, 
it’s impossible to say whether you should believe the author’s claim in how good the GC is, especially when you want to do a comparison. 
I’ve seen GC articles that made the most mundane stuff sound like it’s something that sets their GC apart from others GCs, 
when in reality it’s something that most other GCs of the same genre do or did at some point and already improved upon.

If a GC article makes a claim about the performance, this is where you really need to be careful when choosing whether you should believe the claim or not. 
A good article would tell you the workload, the machine config and the GC config that were used to measure the performance, and relevant performance data. 
If the workload is simply enough (eg, it doesn’t use layers of libraries) or you have enough understanding about the workload to know how much work GC is doing, 
you can get an idea about GC perf in that framework. And if you port this workload to a different framework and isolate it enough to compare just the GC performance, 
you can then make a statement about GC performance comparison.

On the other hand, an article that simply makes claims like “our GC’s latency is less than 20 milliseconds” without describing what was run to get that number is meaningless. 
It could simply be a false statement (as in, you could write a test that simply makes that GC have longer pauses); 
or it could be the case that the framework the GC lives in simply never (or in 99% of the case) needs to handle situations that would make it have pauses more than 20ms. 
Without knowing which case it is you cannot make any decision about this GC which in turn means you can’t compare it with another GC.

And even when you are reading numbers printed out by some GCs you need to be careful how you interpret those numbers. 
Some GCs would indicate which numbers are for STW (Stop The World). Obviously during STW, all your threads are stopped. 
But STW is not the only way your threads will get interrupted by a GC. Some GCs choose to aggressively do mutator assist 
(meaning the user threads will take on some GC work before it actually gets to allocate) so while a thread is doing the mutator assist work it’s the same as being stopped. 
I’ve seen some very long mutator assistant pauses in some GCs that are much longer than their STW pauses. 
Obviously in that case if you are counting on that thread to finish a request that request’s going to take a long time.

In order to accurately assess latency it’s best to measure it yourself. In our GC micro benchmarks we would measure around a request 
(which usually does one or more allocations and some assignments) for latency. Since it’s a GC benchmark, 
usually the latency is due to GC pauses but sometimes it may also be due to other factors like the thread isn’t getting scheduled.

Some of our first party customers implement a diagnostics pipeline that measures the latency of each request and for long requests
 they can attribute the latency to specific tasks that happened during that request – network IO, disk IO, GC or something else.
  Depending on what OS/framework you work with, you might have the option to implement this too.

https://devblogs.microsoft.com/dotnet/how-to-evaluate-info-you-read-on-garbage-collectors/
