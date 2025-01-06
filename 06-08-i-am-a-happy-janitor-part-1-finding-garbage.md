Indeed what I do is very much like the job of the janitors – like the ones who clean your building, or janitors you see at a food court, or yourself when you are taking care of garbage at your house. Doubtful you say? Let me prove it to you.

Finding garbage

What do you do when you are having lunch with your friends at a food court and suddenly you need to go somewhere for a short while in the middle of it (like, you asked for hot sauce but were given not-so-hot-sauce and you went back to the cashier to tell him how it’s so not up to your standard)? Then the nice janitor guy comes along and he sees your plate and tries to take it (you were so hungry you already ate most of your food before your remembered to use the sauce…). What happens now?

One of your friends says to the janitor guy, “my friend is not finished yet.”. The janitor then goes away without taking your plate.

Same principle for the GC, the difference is GC’s friends tell it what’s not garbage, ie, what’s live (seriously..who wants garbage…). GC’s friends include:

· JIT

· EE stack walker

· Handle table

· Finalize queue

What happens is the following:

GC says: I need to start a collection on gen X…My super friends please report to me what is live now for gen X.

JIT says: Roger that. I will take care of the live roots on managed stack frames. When I jitted methods I had already built up a table that tracked which locals were alive at which point so I will report the ones that are currently live to you. I am quite aggressive at determining if something should be dead or not in retail builds – locals could be treated as dead as soon as I determine it’s not in use anymore (not guaranteed). However in debuggable builds I am much more lenient – I will let all locals live till the end of the scope to make debugging more convenient.

EE stack walker says: I can take care of the locals on the unmanaged frames when managed frames call into the EE (for lock implementation and whatnot).

Finalize queue says: Indeed, the almighty me brought back some dead objects back to life so I am telling you about these objects.

Handle table says: Reporting handles in use in gen X and all its youngest generations now.

GC waits patiently while the super friends are reporting what’s live.

The only time GC needs to participate in finding what’s live is when GC needs to find out if there are objects in older generations that are referring to objects in younger generations. For example if GC is collecting gen 1, it will need to go find out if there are any gen2 objects that are referring to gen1 and gen0.


原文 https://devblogs.microsoft.com/dotnet/i-am-a-happy-janitor-part-1-finding-garbage/
作者 Maoni

事实上，我所做的工作很像看门人——就像那些打扫大楼的人，或者你在美食广场看到的看门人，或者你在家里处理垃圾时的自己。你说怀疑吗？让我证明给你看。

发现垃圾

当你和朋友在美食广场吃午饭时，突然你需要去某个地方吃一会儿（比如，你要了辣酱，但却得到了不那么辣的酱，你回到收银员那里告诉他这怎么不符合你的标准），
你会怎么做？然后，好心的看门人走过来，他看到你的盘子，想把它拿走（你太饿了，在你想起要加酱汁之前，你已经吃完了大部分的食物……）现在发生了什么？

你的一个朋友对看门人说：“我的朋友还没完工呢。”然后，看门人没有拿走你的盘子就走了。

同样的原则也适用于GC，不同的是GC的朋友告诉它什么不是垃圾，也就是说，什么是活的(说真的…谁想要垃圾…)。GC的朋友包括：

·JIT

·EE堆栈步行者

·手柄台

·完成队列

具体情况如下：

GC说：我需要在X世代开始收集，我的超级朋友们请告诉我X世代现在有什么活动。

JIT说：收到。我将处理托管堆栈框架上的活动根。当我抛弃方法时，我已经建立了一个表，它跟踪了哪些局部变量在什么时候是活的，所以我将向您报告当前是活的。
我非常积极地决定零售建筑中某些东西是否应该死亡-只要我确定它不再使用（不保证），本地建筑就可以被视为死亡。然而，在可调试的构建中，我要宽容得多——我会让所有的局部变量活到作用域结束，以使调试更方便。

EE stack walker说：当托管帧调用到EE时，我可以处理非托管帧上的本地（用于锁实现等）。

Finalize队列说：的确，全能的我使一些死去的物体起死回生，所以我告诉你这些物体。

手柄表显示：报告手柄现在在X世代和所有最年轻的世代中使用。

GC耐心地等着超级朋友们报道现场情况。

GC唯一需要参与查找活动的情况是，当GC需要查找老一代中的对象是否引用了年轻一代中的对象时。例如，如果GC正在收集第1代，它将需要找出是否有任何引用第1代和第0代的第2代对象。