Author: CT075

This document lists out several issues with the existing EA buildfile pipeline.
Most things here are not a problem of best practices, but rather with the
buildfile infrastructure as a whole, that are unlikely to be solved without
some significant rewrites or new tools. I'm not super up-to-date on the
differences between vanilla Event Assembler and ColorzCore, so definitely let
me know if any of these problems have been addressed.

This is meant to be a survey of the state of the art, some notes on why the
status quo can be problematic, and some suggestions on how we might begin
addressing them.

# Intermediate artifact generation

(otherwise known as "the `incext` problem", but a bit more general)

It's well-known that from-scratch project builds blow up extremely quickly as
you start adding assets. This has two main causes:

- Most assets are inserted via `#incext`, which spawns a secondary process to
    produce a binary blob, which is embedded into the ROM directly. Secondary
    to this is `#incextevent`, which produces EA output (as a string) instead,
    which needs to be parsed, etc. This can lead to a lot of churn while the
    OS spins up your new program runs that are disposed almost immediately.

- Certain assets, like text, take a long time to be processed in the first
    place, with only all-or-nothing work sharing between runs. This has lead to
    patterns like maintaining a separate `Makehack_small` that doesn't rebuild
    these artifacts at all, relying on a previous invocation of `Makehack_full`
    to have created a `text.dmp` that can be reused.

    A consequence of this is that `Makehack_full` *must* be re-run every time
    the text changes, making small changes (like typo fixes, etc) take a
    frustratingly long time to rebuild. Furthermore, there's a sharp edge in
    that a careless user can mistakenly run the "fast" makefile after a quick
    text change, confusingly not picking up the change at all!

A few different solutions to these problems have been proposed.

## Caching

### External caching

By far the most commonly-suggested solution to this problem is to pre-build
artifacts by running the `incext` or `incextevent` commands before invoking EA
at all. This skips the problem entirely by reducing the work done by EA itself
to a simple `incbin`, which is comparatively cheap.

This is a popular solution, because it's lightweight and requires little to no
adjustment from EA itself, only adjusting the buildfiles to use `incbin`
instead of `incext`.

Unfortunately, this solution is not ideal in a few different ways. Firstly, it
requires human intervention to rebuild the right part of the cache each time an
asset changes, which can be a tall order in a large project with lots of moving
parts. This can be alleviated by maintaining a `rebuild_cache` script, but this
only shifts the burden to remembering to run the script at all. Choosing to
*always* run the script by enshrining it as part of the build process causes us
to lose any speed gains (we're still spending time to run all the processes,
it's just now happening at the start of the build instead of mid-build).

This can be especially nasty when we consider that `incext` invocations can use
information that is only known at build-time, e.g.

```
#include some_other_file.event
Label:

#incext Program Label
```

This will pass the numerical value of `Label` as a pointer to `Program`, but
that value depends on the contents of `some_other_file.event`! Replacing this
`incext` with an equivalent `incbin` is deceptively complicated, as we now have
to know the value of `Label` in order to know whether to rebuild the cache, but
that requires running a build!

We're saved from an infinite loop in this situation by the fact that EA will
only substitute labels that are declared before the `incext` in question. That
is, in this situation

```
#incext Program Label

Label:
```

`Label` is passed as a literal string, as otherwise the value of `Label` would
depend on itself.

A third consideration is that I don't know of many people who rely on this
feature, so it may be acceptable to ignore it entirely.

### Make

Of course, programs exist to automate this kind of artifact caching, these are
called build systems, the most commonly-used of which is the venerable `make`.

Loosely speaking, the main advantage of `make` is allow you to declare
*dependencies*, and only rebuild things if a dependency changes. For example,
we could make `text.dmp` depend on `text/*.txt`, which means that we won't
rebuild `text.dmp` unless a file ending in `.txt` in the `text` folder changes.
This is nice, because we now get `Makehack_full` and `Makehack_quick` for free,
and don't need to remember for ourselves which one to use -- `make` will do
"the right thing" automatically.

This is already a huge improvement over the existing batch scripts, but hasn't
been widely adopted for a few reasons. Chief among them is that `make` tends to
do strange things on Windows boxes, and often breaks entirely on files that
have spaces in their names. Secondly, the Makefile language is complicated and
has many gotchas, and so using make becomes unmanageable as projects get
larger, especially if we want to distribute Makefiles for use with popular
sub-components, like the Skill System (see [Recursive Make considered Harmful](https://accu.org/journals/overload/14/71/miller_2004/)
for some details as to why this doesn't scale well).

A secondary issue is that, as is a common trend with most "off the shelf"
solutions, a human needs to ensure that the lists of dependencies are correct.
This can be tricky when taking into account things like the EA standard
library, generated event files (which can, themselves, introduce more
dependencies if you generate an `incbin` or even a regular `include`) and so
on.

These are all solvable in theory using pure make, but such a system will be
inscrutable and unmaintainable, and we should probably avoid them. For example,
we could run a script to find all the files included by a given `.event` file
(although this would not work with `incextevent`!) and produce a Makefile, but
an even better system would be to have EA *itself* feed such information to
another program at build time, so we always have the most up-to-date
information.

## Splitting the build into smaller pieces

With the current system, builds are pretty much "all or nothing" -- while there
is some possibility of caching intermediate pieces produced by tools like
`png2ea` or `lyn`, there will always be one single process that needs to read
the entire project, compute sizes and labels, and actually Write To The ROM.

To some extent, there is no way around this. At the end of the day, we need to
somehow determine that `MyLabel` ends up at offset `0xFABBED`, and to do that
we need complete info. However, we often spend a lot of work re-computing
pointers and offsets that are unchanged from the last build, and re-checking
sizes of binary blobs that havne't changed. You could imagine a system in which
we can just run EA on "the skill system", and get some kind of "bundle" that
contains all the relevant label data, graphics, etc, with the fixed and
relative offsets all in one file (something like the `.sym` file that gas
produces, but probably with more information). Maybe we could have `lyn` for
*event* sources, with our own intermediate format.

This is probably the least important issue to iterate on, largely because it
will probably be the most difficult to integrate into existing systems and is
very easy to get wrong.

# Bloat

A constant complaint heard amongst the skill system maintainers is that the
repository contains projects that are not part of the skill system itself, but
depend on it (or, conversely, that features of the skill system depend on).

# Expressiveness
