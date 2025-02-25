<h1>Helping Customers Effectively</h1>


I have to put a disclaimer here since this is not the usual type of blog posts I write. I’m by no means a master at communication. This is just what I thought that seemed to work well. YMMV of course. But I’d be very happy if they help you in some way ‘cause many of us work with customers. And I welcome your thoughts and suggestions.

I have talked to customers a lot since I started working on the GC. This is in the context of helping customers to handle and resolve issues they are seeing.

1. The intention of helping should be genuine

I am naturally the kind of person who wants to help with coming up with a solution when I see a problematic situation. But an equally important reason in the professional context is that I think that it’s actually a bit misleading to say “help customers” ‘cause it makes it sound like “I’m the giver and the customer is the receiver”. In reality the end result of helping customers is largely to help ourselves ‘cause we wouldn’t be here if we didn’t have customers 😀. If you think about this, the empathy for customers will come naturally and you will genuinely want to help.

I’m sure we all have had this experience – when you sense someone doesn’t actually want to help you, it’s never a pleasant experience.

2. Respect others and assume the best intentions

Respect is a 2-way street so this is really a suggestion for all parties involved.

I have definitely come across my share of customers who simply didn’t make it easy for themselves to be helped. I’ve worked with people who were hostile and/or demeaning and/or didn’t care to make any effort to try any solutions and only continued to complain. But most folks I’ve worked with were very reasonable. Some were so accommodating and that I’m really grateful that I have customers like them. By default I always treat someone new I talk to with respect. And I never felt the need to change this default. I always assume that there’s a good reason why they have or have not done something.

When I ask someone for help, I’m always appreciative – this doesn’t just mean to say “I appreciate your help”, much more importantly, having a clear description of the issue and showing them what you have already tried shows that you have respect for their time. It makes people much less interested to help you when they need to pull the info out of you. We’ve probably all seen the cases where someone is like “It’s broken. Can you look at it?” with no description of what they are doing, what version of the product they are using, whether they have collected dumps/traces and whether they have done any analysis at all and when you ask them they give minimal answers.

I’m actually totally fine with it when people come to me but haven’t done much investigation on their own, as long as they are willing to do it if it helps to resolve the issue. I understand that people have different experiences – there are hardcore perf folks who do memory analysis on a daily basis – honestly I’m impressed with the level of investigation and understanding from some customers I’ve worked with and deeply appreciative of them. And there are folks who just haven’t had the need to learn about memory and that’s fine (in a way this is a good sign – it means GC and our libraries work well enough that they don’t need to care). Sometimes this actually shows a problem on our side – it may mean our doc is too hard to discover or our explanation is simply insufficient. Over the years I’ve put in a lot of effort in improving the tooling and the documentation so it’s easier for the customers to discover and diagnose problems on their own.

If I have any doubts about the feasibility of my suggestions I would end the conversation with “Does this sound reasonable to you?” which provides a clear opportunity for them to voice their concerns. I myself find that very helpful when other people say it to me so I’d like to believe it’s helpful when I say it to my customers.

3. Understand the problem before suggesting solutions

This sounds obvious but you’d be surprised how often people would jump ahead of themselves and talk about solutions without much understanding of the problem they are trying to solve. It’s interesting ‘cause I’ve seen this on both sides. Some of us would start spelling out various solutions that may be useful or just enumerate solutions they’ve tried and worked before; and some customers would want suggestions before they explain what the issues they are facing actually are.

Understanding the problem often means getting data or sometimes even trying out a prototype in order to get data. And I completely understand that may not be feasible so there’s definitely a time and place that calls for “hey, I’ve worked on this for a while and have seen many cases, and here are the possible causes/solutions you should try”. But I’ve definitely seen this happen way before it’s concluded that’s not feasible or without even thinking about the feasibility at all.

When I request info from folks, I explain why I’m requesting it, ie, what kind of things it would help me figure out so we can make forward progress. This helps with justifying the time they’ll need to spend to get the info.

4. Convey applicable information

When I describe the solution/workaround, I include 2 pieces of info – the diagnostics I did, in a way that’s relevant to the issue and the action items for the customer and/or for us.

I’ve seen devs write long emails that mentioned a lot of info when responding to an issue, yet it’s not obvious how each piece of info is related to the issue at hand and more importantly, not obvious what the action items are. Most of the info may not be important for the customer to understand at all to resolve the issue and especially when folks are under time pressure (which means they are busier than usual), this doesn’t exactly help with the situation.

I have a coworker who does this exceptionally well – he would describe the root cause and the fix/workaround at the beginning of his email in a concise yet informative way, and specifically say “the rest of the email is the details that you only need to go read if you are curious”. And the rest of his email describes things in very much detail.

–—–

Of course, as mentioned in the beginning, this is all in the context of helping to resolve issues. If you are helping customers in a different context, eg. “I’m writing a blog post to tell my customers some info about my product that they may not need in their daily work but just ‘cause they are a curious bunch” then of course some of this does not apply.

https://devblogs.microsoft.com/dotnet/helping-customers-effectively/在此，我需要声明一下：这并不是我通常写的那种博文。我绝非沟通方面的专家。以下内容只是我认为行之有效的方法，当然，效果因人而异（YMMV）。但如果这些内容能在某种程度上对你有所帮助，我会非常高兴，因为我们的许多工作都涉及与客户打交道。同时，我也欢迎你的想法和建议。

自从我开始从事垃圾回收（GC）相关的工作以来，我与客户进行了大量的交流。这种交流的背景是为了帮助客户处理和解决他们遇到的问题。

---

### 1. 帮助客户的意图应该是真诚的

我天生就是那种在看到问题时想要提供解决方案的人。但在职业背景下，同样重要的一个原因是，“帮助客户”这种说法其实有些误导性，因为它听起来像是“我是给予者，客户是接受者”。实际上，帮助客户的最终结果很大程度上是在帮助我们自己——如果没有客户，我们就不会在这里 😊。如果你这样想，对客户的同理心就会自然而然地产生，你也会真心想要帮助他们。

我相信我们都曾有过这样的经历——当你感觉到某人并不真的想帮助你时，那绝对不是一种愉快的体验。

---

### 2. 尊重他人并假设对方有最好的意图

尊重是双向的，因此这是一个对所有相关方的建议。

我确实遇到过一些客户，他们并没有让自己容易被帮助。我曾与一些态度敌对、贬低他人或不愿尝试任何解决方案、只是一味抱怨的人合作过。但大多数我接触过的人都非常合理。有些人甚至非常配合，让我非常感激能有这样的客户。默认情况下，我总是以尊重的态度对待新认识的人，并且从未觉得有必要改变这一默认立场。我总是假设他们做或没做某事都有充分的理由。

当我向别人寻求帮助时，我总是心存感激——这不仅意味着说一句“我感谢你的帮助”，更重要的是，清晰描述问题并展示我已经尝试过的内容，表明你尊重他们的时间。当人们需要从你那里费力获取信息时，他们会变得不那么愿意帮助你。我们可能都见过这样的情况：有人简单地说“它坏了，你能看看吗？”却没有描述他们在做什么、使用的产品版本是什么、是否收集了转储/跟踪信息，或者是否进行过任何分析。当你询问他们时，他们给出的回答也非常简短。

实际上，如果人们来找我求助但尚未自行进行深入调查，只要他们愿意为了解决问题而去做，我是完全可以接受的。我理解人们的背景不同——有些人在日常工作中深入研究性能问题并进行内存分析，我对一些客户表现出的调查水平和理解深度感到印象深刻，并深表感激。也有一些人从未需要学习关于内存的知识，这也没问题（某种程度上来说，这是一个好现象——这意味着 GC 和我们的库工作得很好，以至于他们无需关心）。有时，这种情况实际上反映了我们自身的问题——可能是我们的文档太难找到，或者我们的解释不够充分。多年来，我一直在努力改进工具和文档，以便客户能够更容易地发现和诊断问题。

如果我对自己的建议可行性有任何怀疑，我会以“这对你来说合理吗？”结束对话，为他们提供明确的机会表达担忧。我自己在别人对我说这句话时也觉得很有帮助，所以我相信这对我的客户也有帮助。

---

### 3. 在提出解决方案之前先了解问题

这听起来显而易见，但你会惊讶于有多少人会在没有深入了解问题的情况下跳过步骤，直接谈论解决方案。有趣的是，我在双方身上都见过这种情况。有些人会开始列出各种可能有用的解决方案，或者列举以前尝试过并奏效的解决方案；而有些客户则会在解释他们面临的问题之前就想得到建议。

了解问题通常意味着获取数据，有时甚至需要尝试一个原型来获取数据。我完全理解这可能不可行，因此在某些时候和场合，可能会出现“嘿，我已经在这个领域工作了一段时间，看过很多案例，这是你应该尝试的可能原因/解决方案”的情况。但我确实见过这种情况发生在结论得出之前，甚至根本没有考虑过可行性。

当我向别人请求信息时，我会解释为什么要请求这些信息，即这些信息将如何帮助我找出问题所在，从而推动进展。这有助于证明他们花费时间获取信息的合理性。

---

### 4. 传达适用的信息

当我描述解决方案或变通方法时，我会包含两部分信息：相关的诊断过程以及客户的行动项或我们的行动项。

我见过开发人员写很长的邮件，在回应问题时提到大量信息，但每条信息与当前问题的相关性并不明显，更重要的是，行动项也不清楚。大多数信息对客户解决该问题来说可能根本不重要，尤其是在人们时间紧迫（这意味着他们比平时更忙）的情况下，这并不能改善局面。

我有一位同事在这方面做得特别出色——他会在邮件开头简洁明了地描述根本原因和修复/变通方法，并明确表示“邮件的其余部分是细节，只有在你好奇时才需要阅读”。而邮件的其余部分则详细描述了相关内容。

---

当然，正如开头提到的，以上内容都是在帮助客户解决问题的背景下讨论的。如果你在其他背景下帮助客户，例如“我正在写一篇博客，告诉我的客户一些他们日常工作中可能不需要但因为他们好奇而提供的产品信息”，那么上述部分内容可能并不适用。