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

https://devblogs.microsoft.com/dotnet/helping-customers-effectively/