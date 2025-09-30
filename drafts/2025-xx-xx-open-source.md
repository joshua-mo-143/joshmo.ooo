+++
title = "You Don't Want To Maintain Open Source"
description = "A think piece (and rant!) about open source."
template = "blog-page.html"
+++

In my (just over) two and a half years of being in tech, I have been pretty much entirely working in open source Rust. One of my first experiences that really struck me was at [Shuttle](https://shuttle.dev), reading some of the comments from one of the earlier releases on HackerNews. There were multiple comments saying the engineers must really suck because there was a large amount of unwraps in the code. Looking back at the codebase, there were multiple unwraps but it didn't really seem like the unwraps were possible to trigger, so I queried it internally.

The response from one of the engineers?

"If they don't like it, they can push a PR to fix it."

That, to me, kind of epitomises the experience of open source. I know how frustrating it can be to have a bug in a library that "should've been fixed by now" or that the maintainers just seem a bit slack on merging for some reason. I also know that as a maintainer, the act of maintaining open-source frameworks and libraries itself is a massive burden.

## Why open source matters
Open source makes up much of our current "software" infrastructure: free open source libraries like ffmpeg, Linux (of course), many programming languages and more underpin the vast tech economy that society itself is pretty much reliant on. Without it, it's highly likely that the software industry would be highly monopolised by a handful of companies. Perhaps that is not so different to our present situation where we have several tech giants, but at least if you want to make software at home you don't have to buy a license just to do that.

## The problem with open source
Despite open source maintainers working on perhaps some of the most important software projects of our time (Linux, anyone?), many maintainers are also extremely underpaid. It is not uncommon for many maintainers to simply stop working on projects because they can't afford to do it, and there are a significant number of developers who simply won't work on it because they're not paid to do so. And as for me being someone who is privileged enough to be paid to essentially be an open source maintainer, I can't say I blame them.

Additionally, despite their utter reliance on open source packages, many companies do not give back to the maintainers at all. Even worse, many employees at said companies rely upon said packages to carry out some business task by a deadline - so when they encounter a bug, of course they will ask the maintainer to please fix it because they need it for XYZ project instead of trying to add a PR themselves. This puts additional strain onto the maintainer, who is probably already swamped with tickets.

Many projects also have a very low bus factor. That is to say, if the maintainer were to go missing, the project stalls - sometimes permanently. There are [over six million projects](https://opensourcesecurity.io/2025/08-oss-one-person/) with one maintainer, so this is not just an isolated incident. In many cases, I've seen many maintainers almost single handedly maintain the packages underpinning entire ecosystems, [David Tolnay](https://github.com/dtolnay) and [Taiki Endo](https://github.com/taiki-e) being two such examples within the Rust ecosystem itself.

Then there's the community. I have been very fortunate to be in the Rust ecosystem where I would say nearly if not all maintainers are generally treated with a great level of consideration (at least in my experience, they seem to be). However, it would seem in larger projects that there are some users who quite seriously do not care about the maintainers of the project and will be outright toxic, simply because their issue won't get solved. This can really alienate maintainers from their communities because it brings about the feeling that the community you've cultivated does not appreciate the work you do - causing a chain of people who continually become more fed up of each other with each interaction.

## Advice for would-be maintainers
Through all of this, if you still truly want to be an open source maintainer I have some good advice for you that has served me quite well over the time that I have been working in this space:
- Don't take it personally. Eventually, you'll have that one guy who yells at you and tells you that your code looks like crap. It's not really your fault - they're just lashing out.
- Lower the barrier to entry. Make the instructions clear and easy to use. If your project takes the equivalent of a machine spirit summoning ritual to work, it's going to be very difficult to get new contributors in (good quality ones are already rare as it is!).
- Take time for yourself - by the time you're already feeling burnt out, it's too late in the process. What's more important: a bug fix today, or being able to keep the library alive and maintained?

## Conclusion
Open source can be incredible. Watching strangers use your code, seeing it pop up in real companies - that part feels amazing. But here’s the other side: it will drain you if you let it. The world won’t shower you with gratitude, and the people who depend on your work the most are often the ones who give back the least.

So go into it with your eyes open. Don’t expect applause. Don’t expect stability. And definitely don’t expect to keep your sanity if you try to be everything to everyone. Maintain open source because you want to, not because you’re chasing validation — otherwise, it will chew you up and spit you out.
