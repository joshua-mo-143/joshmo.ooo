+++
title = "Good tooling matters."
description = "My thoughts on good tooling and how tooling affects productivity"
template = "blog-page.html"
tags = ["miscellaneous"]

[extra]
thumb = "good-tooling-matters-thumb.png"
+++

Like every tradesman's toolbox, software engineers also have a toolbox that must be fit for purpose. If you don't have good tools, you can definitely get the job done - but it will be much more difficult to replace. If you have low-quality tools, you'll probably need to replace any breakages much more often. And of course, nobody's trying to hammer a nail with a pair of pliers unless you have a point to prove.

In my opinion, good tooling should do a few things:
- It should feel natural to use
- It should be comfortable to use
- It shouldn't get in the way of your other tools

The obvious parallel to this is the code editor. A good code editor, at minimum, should provide an interface that's intuiitive to use and gets out of the way where required. But an excellent text editor should have extra value-adds to help you become (even) more productive as you learn a given IDE and get used to the workflow. Vim is an obvious example here. With Vim motions, you can type much quicker by using shortcuts to be able to get the non-thinking part of software engineering over and done with. How much this benefits you depends primarily on your typing speed and willingness to learn it, of course, but it's there. It also helps if you've already thought of the whole design for a project so you can write it all up quickly (or "blazingly fast").

Of course, the best tooling is generally hotly contested. However, this doesn't mean the best tooling is automatically what makes you the most productive. And that's okay! If you need to ship something and get it done fast, at the end of the day you're always going to use your most productive tech stack. However, learning new tools in the hope that they can make you more productive is never a bad thing.

Having good programming language tooling, in my opinion, is doubly important. At the very least, it should be easy to use and it should be obvious what things do. A good example of this is Cargo - Rust's package management, test runner, compiler (wrapping around rustc) as well as built-in aliasing for Cargo commands is an extremely solid piece of kit. There are other things that improve on it, of course: `cargo-nextest` does an extremely good job of improving on the Rust testing experience. However, if you were to only use `cargo` and nothing else, there is not a huge amount you would be left wanting. The stuff that's left can generally be found as packages on Crates.io. 

After having used Python for almost 2 months professionally now, I can comfortably say that I don't feel like the tooling in Python is that great. This was mostly informed by me getting back into Python and re-learning the ecosystem: having to manage different Python versions and pick a package manager, then dealing with issues regarding the various linters (mypy/basedpyright to be specific) and untyped libraries, and so on and so forth. Does this mean Python is absolutely useless? No, of course not. It's still a useful language for scripting and has remained so for very, very many people. However, the inconsistent tooling when trying to use Python for larger projects has cost me quite a few hours of productivity and I can't really reclaim that time back.


