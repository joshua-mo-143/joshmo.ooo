+++
title = "On Betting on Yourself"
description = "Some reflections on taking control of your own narrative and betting on yourself."
template = "blog-page.html"
tags = ["miscellaneous"]

[extra]
thumb = "on-betting-on-yourself-thumb.png"
+++
## The initial bet
Making the initial bet doesn't start with a grand gesture, or moment. It starts when you believe in something so strongly that you will chase it with everything you have.

For me, that started as a teenager. I loved music and wanted to get into songwriting and all that kind of stuff. I even eschewed going to university so I could get a music diploma... although at the time when talking to my parents, I don't think they thought it was a good idea. And even though it was pretty obvious I was not getting much success with it, it was something that I felt very strongly about. 

Even though I stopped doing music, I continued to have things that I believed very strongly in and continually pushed my own limits. At this time, it was primarily through video gaming. I became a semi-hardcore speedrunner in an MMO, leading many, many raids as well as stepping in to help lead other peoples' raid groups and constantly sharpening my own skills. At the time I didn't have a job or anything, so this was the only thing I really had.

At the time, I did it because I genuinely wanted to be one of the best players in the world. In reality though, while it did work to a degree and I was known to be one of the better players on North American servers I was also known to be extremely toxic. After a while, I really burned out of the game and I think this was the initial catalyst for me to basically get a job and become a contributing member to society.

Fast forward a few years, the very first "bet" I made professionally was that I could break into tech. The odds were stacked against me given that I worked in a warehouse job, but I believed in my ability to just do things and find stuff out. Anyway, fast forward 8 months and I was extremely lucky to have found Shuttle - a Rust native deployment platform. They initially hired me part time, then hired me full time after 6 months.
## Holding conviction
Due to my work at Shuttle initially being part-time, I didn't really have enough money to live off of from just doing that. So to pick up the rest of the slack, I initially also had a full-time admin job at the same time. However because learning Rust was also becoming a full-time job initially, I basically had to manage two full time jobs at the same time. 

I'm not going to lie, it was tough. There were a lot of days where it was pretty obvious to my partner if not myself that I was becoming extremely burnt out. I was basically having panic attacks almost every other day. At the same time though, there was something in me that just told me to keep going because I knew that my place (and more importantly, my people!) was in tech and even if I was running on empty, I just had to keep going and trying. 

Fortunately, 3-4 months into working for Shuttle after we knew it was going to work out I quit my full-time admin job so I could work purely on developer relations stuff. And it worked out - insanely well, in fact. If you'd known me purely from the articles I wrote, you would probably have not known at all that I was basically a junior. 

[This was the moment when I knew it'd work.](https://www.youtube.com/watch?v=Bh_tNehUV3k) Admittedly I did make a number of points that were... well, somewhat incorrect and unintentionally clickbaity - but when it's on your personal blog, it's a bit different from posting it on the company website. Needless to say I had zero idea about the traction it'd receive until my coworker posted about it and tagged me. That was an absolute watershed moment for me and was the moment I realised that I could actually really do this full time. 

In talking about Shuttle, I'm reminded of a recent conversation I had with an ex-colleague. By normal company standards, the expectations placed on me were such that I was pretty much being expected to output a team's worth of work. Any normal engineer would've probably quit already or negotiated their role scope down. 

However, it was obvious that the work was gaining a lot of traction in the Rust community which made the work highly visible, and more importantly extremely valuable for me as my name was being attached to it - which meant that the right choice for me at the time was to keep doing it. I could have negotiated the scope down, but my impact would have almost certainly been reduced - which meant if I wanted to keep accelerating my impact, I had to keep going. After a while though, it became much easier to just start pumping out articles so it's not as crazy as you might think.

## Pushing the frontier forward
Sometimes, betting on yourself isn't just about "can I work hard enough to make this happen?". At some point when you've been in the industry for a while, betting on yourself is also partially making bets on career trajectory by optimising for applications with companies using technology you want to work with. And in this case, it was AI.

I'll be honest, I was pretty skeptical of using AI at Shuttle. In fact, I pretty much initially flat out refused to use AI for the articles because the LLMs just weren't good enough yet and there was a big part of me that believed that the personal touch is almost the entire point of an article. If you're reading something that a machine's spat out and it's not an educational article, why would you read it? There's no taste nor love injected into the writing process.

Initially, my interest in LLMs was piqued because someone from Qdrant reached out to me at the time to join their ambassador program because I'd re-published an article in an effort to try and improve the SEO score. This was kind of around the time when Retrieval Augmented Generation pipelines were a big thing in improving LLM outputs.

I'm going to be honest, I had almost zero idea of how to use vector search for advanced use cases outside of semantic search. However, I accepted because even if I wasn't very knowledgeable at the time, joining the program would mean I'd have access to people who could help me get to where I needed to (being knowledgeable about RAG as well as using LLMs). That was a huge win in my book and I am extremely thankful for Qdrant. Around this time, I learned that LLMs are in fact pretty powerful when used correctly.

Fast forward to maybe very late 2024, I was introduced to Playgrounds who own Rig. We did a collaboration together where I wrote about how to write a Rig agent and host it on Shuttle. While I think the framework at the time was a bit clumsy, there was quite a lot to like about it and they got enough right that I thought it had quite serious potential.

In early 2025, they basically offered me a job after seeing my side projects and I was brought on to help co-maintain Rig and do some technical marketing for it. At the time I joined a team of 2 other engineers plus an intern at the time who I believe is now a full time engineer. After a few months though, we needed to pick a course of action as there was a new internal project brewing which needed more manpower (and would generate actual revenue). We had two options: either someone could maintain the framework full time, or we could have the engineers do a tour of duty. 

Me being the person that I am, obviously chose to ask to maintain the framework full time. It was a bit of a moonshot at the time given that AI still very much had (and still does have!) a negative stigma within the wider Rust community, but I made a prediction that Rust and AI would become a thing for a lot of companies who need robust services that can also talk to LLMs. 

It took a lot of time for people to come around to the idea that you could reasonably use Rust for AI, let alone Rust for web development. There was a lot of work to be done on teaching people why Rust is a good use case for it and what the tradeoffs are. There was a part of me that also sort of wondered if I might've made a bad choice. Regardless, a lot of my conviction went into simply trusting the process. 

A few months into me constantly pushing PRs up as soon as the CI run passed, we had our first visible external prod user - and I was extremely happy about that, because it validated all the months of work done up until that point. To making Rig not just a usable library, but a competent one that people will actually like using. 

Fast forward to today, there are a huge number of people using Rig daily if not weekly, Rig itself is in an extremely healthy spot and I also hired a part-time junior who has since become an extremely effective hire. She made up for a lot of the weaknesses I personally had as an engineer in that while I was good at shipping quickly and doing most conventional things in Rust, she had a lot of the more esoteric functional programming knowledge that made some of the things that currently exist in Rig even possible at all.
## What I learned from betting on myself

### The cost of moving quickly
Throughout my journey, there has been a pattern of basically spending a lot of late nights working and learning/studying the field so I could grow quickly as an engineer. Now I'm not saying that everyone should be doing this, because if you value work-life balance it's quite frankly a bad idea. However, done correctly moving quickly can have huge upside - if you're intentional about it.

### Ego can be a useful tool for growth
Ego, when used correctly, can be a highly useful tool for growth. However, it has to be used in a specific way - it's extremely easy to let your ego get ahead of you and now suddenly you've gotten in an argument with your coworker because you refuse to be wrong. In this case, I would like to propose using ego as a tool for motivation, self-awareness and setting high (but reachable!) standards for your own work.

The goal here is primarily to realise that you can always do better - and that doing your best work doesn't mean always doing it by yourself. You can do most of the work by yourself - but there **will** come a point where you'll need someone to help you because you lack experience and it is much more likely to come sooner rather than later. Ultimately, there's only so many times you can say "fuck it, I'll do it myself" before you realise there's too much for you to do and you need additional help.

### Learn when to slow down and when to speed up
Admittedly, this is a very difficult line to tread when you're in a startup because crazy things are happening all the time. However, I would highly suggest picking your battles on when things don't need to move quickly and when things do need to move quickly - and executing on that. 

As previously mentioned, moving quickly all the time will burn you out extremely quickly if you're not careful. On the other hand, not moving quickly enough can be a death trap. Knowing what things you need to move up or down on the priority level really can help manage expectations, as well as burnout. My key note for this is primarily that asking for clarity on how high priority something can really help - and after a while you tend to get a good sense of what can be moved up or down the priority ladder.

## Conclusion
Writing this article has been an extremely helpful exercise on where I stand in my current tech journey and helped me reflect on the things I have learned in these 3 years that I've been active in the tech community.

Three and a half years ago, I'd left my job working in a warehouse to gamble on getting into tech. If there's one thing to take away, it's that your bet doesn't have to make sense to everyone else - it just needs to be personally meaningful enough for you to go all out for it.
