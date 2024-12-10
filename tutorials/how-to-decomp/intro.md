# Levelling up in wizardry with the decomp

With the [fe8u decompilation
project](https://github.com/FireEmblemUniverse/fireemblem8u) rapidly approaching
100% coverage, it's becoming more and more important that any aspiring wizards
be comfortable with using it. As with any large codebase, however, it can be
hard to navigate and intimidating to get started with. This guide aims to be a
starting point by giving some strategies for navigating and thinking about the
decomp.

Now, there is no shortage of very comprehensive documentation floating all over
this site, and you can skip a lot of the busywork with prior knowledge of those.
If FEBuilder has the answer, then by all means, check there first. For
completeness, however, I'll be documenting the process "from scratch".

This is **not** an informational reference; don't expect to walk away knowing
exactly what functions are in `src/bmcontainer.c` or what order the battle
calculations get performed in. This is also **not** a step-by-step tutorial.
There *is* no universal way to "use the decomp" -- it will depend heavily on
what you're actually trying to do. There are many ways to "use" the decomp
beyond what is outlined here, most of which I'm not qualified to write about.

What this text **does** aim to do is provide tools for breaking down the
decomp into smaller, more manageable pieces. The reality is, navigating a large
codebase requires trial and error, backtracking, and more than a few leaps of
faith. Instead of memorizing tiny details (which I don't find to be all that
helpful anyway), the focus is on strategy - what questions to ask and how to
connect the dots to reach the answers for yourself.

**Audience check**: This text is intended for readers with some experience
writing small- to moderately-complex custom features in assembly or C. Basic
familiarity with C syntax (specifically, you should be able to read C) is
strongly recommended. You do **not** need to have looked at the decomp before or
have any familiarity with its structure.

In the interest of not just dumping advice in a vacuum, I've also made sure to
include some case studies where I was using the decomp myself, with as much of
my scratch work, false starts and dead ends included as I can remember.
