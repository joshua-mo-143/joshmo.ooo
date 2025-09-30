+++
title = "I Hired My First Rust Junior - Here's What Happened"
description = "I just hired my first junior Rust dev - let's talk about it."
template = "blog-page.html"
+++

## Context (and the reason for the job role being open)
As someone who currently works as open source maintainer for their day job, it's pretty hard work (and I'm sure any fellow open source maintainers will agree). Up until about 2-3 months ago, [Rig](https://github.com/0xplaygrounds/rig) has previously had 4 people working on it. However due to financial reasons (VC-backed startup needs to make money), everyone else was moved to the commercial product to-be while I volunteered to single-handedly maintain the project and ensure the project could still grow and be reliably maintained - thus began my sole maintainership (and being a figurehead) of the library.

Between then and now, we've been published on websites like SurrealDB and the Solana developer docs, we've had several companies tell us that they've adopted us in production with many more public repos from companies like Nethermind visibly using Rig, and I am also planning to do some talks on Rig very soon. However, all of this stuff takes serious time investment to make happen through solid, consistent code contributions. Trying to handle all of this solo without a team is quite a serious mental toll and it's not long before thoughts turn to needing to quit programming and live on a farm.

Naturally, this doesn't go amiss by the rest of the team. When you're doing everything in public, there is nowhere you can hide. I was asked by management if I needed someone to help me handle some of the technical work, which I was only too happy to agree to. Thus began the recruitment process for hiring a Rust junior developer.

## The job description
The initial scope of the role itself was set to be quite small: a 3 month part time contract, paying $3500 a month. The person to be hired would initially help me with bug fixes and documentation (to assist with rapid on-ramping), and then eventually progress onto feature work. Of course, this is not a 100% rigid structure and whoever got the job wouldn't only be working in this order. However because bug fixes and docs are the quickest way to learn a codebase, it really makes a lot of sense (at least in my opinion).

As someone who works in a startup, the core qualities that I was looking for were that essentially they could write somewhat non-trivial Rust without needing hand-holding. They would also need to be able to work autonomously as this was a fully remote role. This perhaps goes without saying, but I will also need to actually like talking to the candidate - they will basically be my teammate and I'll be their primary point of contact, which means that being able to at the very least get along is crucial.

## The recruitment process
I decided that I would handle the entire hiring process primarily on my own. Not that we were planning to use a recruitment agency anyway, but I would prefer to have a very personal touch when it comes to hiring and therefore a first-hand experience of candidates. I would like to think that while I might not be the perfect manager as I've never been in a managerial role before, as I am the only person on my project I do know what kind of person I need for the role. This essentially means that I can be quite ruthless with ensuring that only candidates that I feel would be a great fit for the role get interviews.

To be honest I am not a fan of these long, drawn-out processes where you have 5 or 6 interview rounds: an initial screening round, a couple of technical rounds, some of the team will probably interview you individually or as a group for culture fit, and then maybe individual C-suite interviews. After all that, you might get hired. I think it's an inefficient use of time for everyone involved, especially for candidates - so I decided there would only be two interviews: me, and then whoever gets shortlisted will have an interview with myself and the CTO.

My first thought was to post [a simple job ad on Twitter.](https://x.com/joshmo_dev/status/1957914986648326427)

## What actually happened
The job ad ultimately ended up going viral on Twitter because it was quite simple and laid the details out very clearly: there will be only 2 interviews, candidates must bring a technical side-project to the interview, and all candidates who apply must have previously used Rust before.

A similar job ad which I posted on LinkedIn also experienced very high amounts of engagement.

This of course meant that I was inundated with private messages from people trying to apply. I tried to reply to all of them, but it was mentally a lot and I eventually just had to quote tweet the post (and editing in the case of LinkedIn) to say that the job posting was closed within literally about 24-48 hours of it being opened.

There were a few things I found interesting, which we will talk about below.
1) Many, **many** seniors applied for the role in spite of it being for juniors - Rust seniors included. I am not against just shooting your shot because essentially you never know what will happen, but I was mostly looking for someone who would grow into the role. As someone whose first tech job was specifically using Rust myself, I wanted to basically guarantee that we could get someone who was still early on in their tech career but could write more than trivial Rust code.
2) Many juniors applied who didn't have any Rust experience at all, despite the job description explicitly asking for it. This is more a sign of the job market just being complete crap and people basically just applying to whatever's available even if they don't have the skills for it. I get it.
3) Many other juniors who applied that actually knew Rust had very basic toy projects. Now I'm not saying that everyone should go out and write advanced side projects - because not every role needs that. However because Rig is a very generics/trait-heavy framework, ideally we needed someone who has worked with libraries before, as well as showing that they're self driven in some way (advanced side projects are one way to show this).
4) There were an absolute flood of Rust blockchain developers who applied. This was basically guaranteed to happen because it's a Rust role. My stance on blockchain is relatively neutral: however, many of these developers didn't have any side projects outside of crypto or were able to show that they could code outside of a web3/crypto context, so it was difficult for me to justify interviewing many of these developers. I am not saying they are unskilled - more that I need to see that they have some versatility.

Yes, that's 4 things and not three.

Of course, I also couldn't get around to everyone as I still had to do my own things while this was all going on. Unfortunately as I am one person with finite resources and time, this means things do slip through the cracks.

Additionally, some candidates didn't actually have a technical personal project but still showed a lot of promise. I mostly took this on a case-by-case basis, but we essentially did a whiteboarding task where I asked them to recreate the architecture of an AI co-pilot agent that can take background processing tasks (aka Claude Code at home). Nothing too extensive, but I feel that this was the best task that combined systems programming with LLM & Applied AI field knowledge.

## Shortlisting
Because there were so many candidates who applied (of which many were very, very good), it was very tough to actually set up my shortlist. There were a few things I noticed however in the successful shortlisted candidates that really, really stuck out to me:
- Quite a few of the candidates already had experience with functional or systems style programming. They had a very good grasp of Rust's type system and could reliably use sync/concurrency primitives `Arc<Mutex<T>>`, channels and more.
- Several applicants who have transitioned from other fields into tech had quite frankly impressive side-projects, some of which were even deployed in production. These projects also demonstrated production grade architecture (MVC, observability pipelines with tracing, generics and more) alongside good code structure.
- Some candidates had already made meaningful Rust OSS projects or were even maintainers of Rust OSS projects, which made evaluating their Rust skills very straightforward (and for which I am immensely thankful for as it saved much time).

Ultimately, the shortlist came down to three really strong candidates who combined solid technical skills with a clear ability to tackle real-world problems - ideally problems they actually cared about. While the CTO would make the final call, narrowing the field this way was incredibly valuable for me. It wasn’t just about who could write Rust today, but who had the potential to grow into the role and contribute meaningfully over the months to come. Of course, I also greatly enjoyed talking to all of the shortlisted candidates, so I don't think team compatbiility was really an issue there.

## Reflections
Hiring is difficult. While I probably still won't be considering a recruitment agency any time soon (even if the company was paying for it), the amount of effort that goes into having to write the job description correctly, ensure that candidates have been replied to and interviewing on top of regular duties is quite intense. Many of the small details that make up the candidate application experience matter way more when it comes to hiring than I initially realised.

Additionally, doing job rejections was unexpectedly much tougher than I iniitally anticipated. Because I ran the whole process myself and enjoyed pretty much every interview to some degree, it was difficult to reject candidates who were very suitable for the role but couldn't justify replacing anyone on the shortlist. Still however, I made sure to reply to everyone that I could. In today's job market, empathy isn't optional - it's the bare minimum when many candidates are struggling to find positions. There were several occasions when applicants asked me for an unpaid internship.

While being "the best at Rust" (out of the list of candidates) is what I was somewhat looking for initially, in the end it didn't matter really as much as having good systems level thinking and many other qualities that the more successful candidates had. Being good at Rust (or at least just not bad) does go a very long way. However because the role is for maintaining a library, we also wanted someone who was able to have a better sense of the bigger picture and how things fit together as well as how they'd potentially fit into the wider team should the scope expand.

Would I do this again? Probably. I very much enjoyed the process of being able to select candidates myself, interview and find more about them, as well as learning from them about what they're up to and perhaps learn a thing or two that I didn't know before. However, next time I will be ensuring my appointment calendar is much more balanced to avoid doing interviews right before I have to sleep.

## Wrapping up
As my first ever hiring adventure, I have been pleasantly surprised both by how capable candidates have been and additionally how much I have enjoyed talking to them. While I initially came into this looking for "the most skilled Rust junior that the job salary can buy", it eventually became much more about the individual's well-roundedness and external factors outside of Rust programming skills.

This has been a learning experience I'm very glad to have had - and I'm even more thankful that Rig now has another person to help keep the momentum going. While the work doesn't automatically become easier, it's much less isolating when you have team mates to help balance the load.
