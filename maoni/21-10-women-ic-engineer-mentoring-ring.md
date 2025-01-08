<h1>Women IC (Individual Contributor) engineer mentoring ring</h1>

During this fiscal year I ran a women IC mentoring ring in the Developer Division at Microsoft. It was part of the women's mentoring ring program in our division. I've always felt a little sad when I looked around and saw very few women ICs at very senior levels. Most women who advanced to those levels became managers. This was what prompted me to suggest such a mentoring ring to the organizers of the women's mentoring ring program. I'm happy to report that the ring remains one of the most requested so it will keep going for next fiscal year (I will however be leading a different mentoring ring just because we tend to change up the mentors in each ring from year to year).

As we are discussing next fiscal year's mentoring program, I came across the notes from the last one and wanted to share some of the discussions we had (that can be shared publicly) as I think these are generally applicable and could help other women (or men) too. These were a collective set of wisdom from everyone in my mentoring ring, many of them were suggested by mentees, not myself 🙂

# Time management

"There are so many PRs on my team, I want to review a lot of them so I can learn from them, but I don't have enough time!"

This was a pretty common question from especially more junior engineers and engineers who work on teams with a very diverse set of technology, eg, an API/SDK team. We talked about the following –

Spend your time wisely

You have to choose what you spend your time on. It's always good to be curious and to want to learn more but you have to decide what's relevant enough for you to spend time on. Know who the experts for areas relevant to you and ask them for their wisdom! On a healthy team, experienced folks are absolutely supposed to help new comers – it's part of their job.

<h2>Spend your time efficiently</h2>

If you believe there's something reasonable to mention in a PR but is not mentioned in the description, it's perfectly reasonable to ask to have that kind of info put in. For example, if a PR introduces a new pattern in the code base that's widely used in that PR, it's a reasonable thing to ask the author to describe that new pattern in the PR. Especially for APIs, asking for tests (or sometimes if you can, enforcing tests) to go with the implementation is a great to help you understand how to use the API. This can help you tremendously to understand the PR instead of having to figure out everything on your own.

"How to achieve better work life balance, especially for women who are mothers?"

Being clear about what other folks can expect from you is very helpful, eg, “I don’t work on weekends or at night”, “I don’t answer emails outside business hours unless it’s absolutely urgent, in which case text me”, or “the baby is here and may start crying and I’ll need to attend to him/her during the meeting”.
Give clear description of what work will be accomplished during a timeframe, eg, “it takes X hours/weeks to get to functional, no stress, no perf, no doc and this is with no distraction/interruption”.
Be strategic about your time management. For example, with kids you might need to scatter your work hours out around the kids’ activities.
When discussing work items with your manager, be clear about what to cut if you need to add something.
Adapt to changes, eg, if you WFH when your coworkers work from the office, be more responsive to their issues to not be the bottleneck, and request that they keep important conversations accessible to you (eg, in Teams channels instead of only chatting amongst themselves).
At the end of the day, it’s about your choices, do what is comfortable for you. More importantly, stick to your choices/priority list when you make them. For example, if you decide having kids is a priority, it means be ok with not putting in as much time for work when the kids are little. It doesn’t make much sense to compare how many hours you are putting in with your coworkers who don’t have kids and are willing to put in more hours, since spending time on your kids is also a fulfilling part of your life.
Communication skills
"How to voice my opinions in a conflicting situation instead of remaining quiet"

We've all been in conflicting situations before. Women tend to have more trouble voice opinions in such situations. This was actually brought up in the context of conflicting situations when reviewing PRs. The following was suggested –

Determine if it’s something worth spending time on and/or how much time you should spend it

Is it a trivial issue? Is it subjective? Does the other person have stronger opinions on it than you? If the answer to any of those is yes, that would be a factor that makes it less worthwhile for you to spend a lot of time on.
Recognize that it's perfectly valid to have emotions; also important to recognize whether these emotions have any long last effect. If it does that makes it a problem more worth solving.
If you need to interact with the person you are talking to long term (eg, your teammate) it also makes it more worthwhile to sort it out.
Recognize you don’t have to accept all comments on a PR.
When it does get emotional

Much easier to talk in person (or on Teams) if you have such options, than continue the discussion in email/GH. If we are talking about your teammates, you certainly have the options to talk to them face to face.
Take some time to cool down before chatting with the person.
Make the other person take some responsibility!

In case of fairly subjective matters, ask the other person to take responsibility to prove their way is more desirable. For example, if the person claims something should be the new pattern, ask them to make it a pattern and convince others on the team it should be this way, especially if they are more senior than you
"How to explain something clearly (eg, what you do; your idea; your suggestion) to others that may not be intimately familiar with it already?"

When applicable use code snippets to help explain.
Do a demo for folks
Use pictures! Get a tablet. I use this one which works very well with the whiteboard app and super easy to set up
Send out info before the meeting. Some people would read it and others may not. If the doc isn’t very long you could even dedicate the first few minutes of the meeting to make sure everyone has read it (briefly).
A good meme/video/analogy with an everyday item could make your explanation much easier to understand. I have an example of explaining the GC tuning with food courts here.
Share your explanation with one person to see how they think of it before sharing with more people
Ask the person if the thing you just explained made sense – this doesn’t work 100% of the time as some people would say it does when it doesn’t but there’re definitely people who will say “no” and asking this question gives them a perfect opportunity to ask questions.
"I'm junior on my team, how can I possibly change the culture of the team?"

One trick is to find a more senior person who's willing to mentor and advocate for you – when you ask nicely, usually folks will say yes. Asking someone for 15 mins of their time for a meeting will almost always yield a "yes". Be sure to be respectful of their time – one key thing I always make sure of is to have topics prepared before the mentoring session so I don't waste the other person's time.

"English is not my first language and I don't feel confident when speaking it. And when my teammates who share the same first language with me always talk to me in that language, what should/can I do?"

(This question obviously assumes English is the common language used at your work place, replace it with whatever language is at your work place)

If you are not talking about work it's totally okay to talk in whatever language you are comfortable with but if you are talking about work, really force yourself to get into a habit of speaking English. This is not just to practice your English; you are also appearing a little rude to your coworkers who don't understand that language (it's pretty easy to figure out whether you are talking about work – you hear English words here and there 🙂). Indeed it can be awkward to force yourself and your coworkers who want to talk to you about work in an language that's not English but it creates a much larger long-term benefit.

"Do I need to appear confident all the time?"

Not at all! It's totally fine to not be confident in topics you aren't knowledgeable about. Be okay with asking naive questions – if you are really worried, you could prefix your questions with "I'm not familiar with this so my questions are probably gonna be naive".

Productivity tools
Use SharpLab to see codegen for C# code

Use godbolt to see codegen for C++ code

Use ILSpy to see source for a .NET assembly

Use MermaidJS if you want to draw professional looking charts with simple lines of text. An example is here which was done with some simple mermaidjs script with a mermaidjs extension in VS code:

Copy
graph TD
Copy
        A(obj0/1/2 in gen0) -->|gen0 GC| B(obj0 dies<br>obj1/2 in gen1)
        B -->|more allocation| C(obj3/4 in gen0<br>obj1/2 in gen1)
        C -->|gen0 GC| D(obj3 dies<br>obj1/2/4 in gen1)
        C -->|gen1 GC| E(obj1/3 dies<br>obj4 in gen1<br>obj2 in gen2)
        C -->|gen2 GC| F(obj1/3 dies<br>obj4 in gen1 obj2 in gen2)
    classDef nodeStyle fill:#eee,stroke:#555;
    class A,B,C,D,E,F nodeStyle;
    linkStyle 0,2,3,4 color:blue;
    linkStyle 1 color:teal;
Use the Docs Authoring Pack extension in VS code to help with doc work

Use the Code spell checker extension in VS code to check for spelling

Use the OpenAPI editor extension in VS code to make looking at swagger files (REST API description) easy

 

https://devblogs.microsoft.com/dotnet/women-ic-engineer-mentoring-ring/