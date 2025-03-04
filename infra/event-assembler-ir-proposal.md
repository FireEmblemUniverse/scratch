# EA Intermediate Format

This document outlines and proposes a new intermediate representation for
Event Assembler installers that is intended to be the halfway point between
direct binary output and raw EA source.

## Motivation

Event Assembler has grown a lot beyond its original goal of being an assembler
for *events* (specifically, scripting for FE chapters). Specifically, ever since
the advent of `#incext`, it's become both a build system (technically just a
command runner; it doesn't do any kind of short-circuiting or dependency
resolution) and general-purpose freespace allocator.

The latter functionality has caused some problems with distributing EA "modules"
with tools that don't also reproduce the command-running functionality.
FEbuilder sidesteps this by bundling EA itself internally, but has difficulty
tracking the changes made by any given EA installer, let alone with the "insert
EA" button. This means that basic functionality like "undo" and "is this patch
installed" need to use buggy heuristics or other workarounds.

## Overview

The work done by most EA installer modules can be summed up relatively
concisely:

- Compute a block of bytes and write them to some fixed offsets.
- Compute more bytes and write them to the current freespace head.

The actual process of computing the bytes to write can be arbitrarily
complicated between raws lookup, macro expansion, calling out to external
tools and so on, but the only one that can't *in theory* be done ahead of time
is label expansion. This means EA patches could be distributed almost entirely
in binary form, with carve-outs for specific pointers.

## Technical Details

### Basic structure

A prebuilt EA patch will take the form of one or more named *blocks*. A block
is specified by an optional write-to location and a sequence of *events*,
where an event is either a chunk (corresponding to some bytes written to the
current write head) or a label. A chunk is either a sequence of bytes or a
pointer.

```rust
struct Patch = P(Vec<(String, Block)>);

enum BlockKind {
  Located(u32),
  Free,
}

struct Block = B(BlockKind, Vec<Event>);

enum Event {
  Chunk(Chunk),
  Label(String),
}

enum Chunk {
  Bytes(Vec<u8>),
  Pointer(String),
}
```

When applied, `Pointer` chunks should be expanded to a GBA pointer (little
endian, ROM offset 0800:0000) corresponding to the address of the named block.

### Variables

We can relatively easily add support for basic definitions by allowing a chunk
to be a variable with a size. These could correspond to configuration options
usually expressed via simple EA definitions (e.g. `#define NumSkills 255`).

### User-visible changes

EA would be extended with an ability to output in "patch mode". Currently,
if no initial `ORG` is specified, all implementations of Event Assembler will
set the initial write head to offset 0. In patch mode, writes with an empty
write head stack would become `Free` chunks, while each `ORG` would initialize
a new `Located` chunk.

When in patch mode, definitions can be specified as "deferred" (syntax TBD).
Only simple definitions (no parameters) can be deferred. When encountering a
deferred symbol during macro expansion, it should become a `Variable` of the
appropriate size. Performing any operation other than directly writing a
deferred value is an assembly-time error (e.g. `2 * deferred`).

#### Deferred conditionals

If a deferred symbol is branched on via `#ifdef`, execution should be forked,
producing separate `Block`s.

We have two options for relating these blocks to each other:

1. Turn `Patch` from a flat list to a tree replicating the conditional structure
   of the original source, or output `Event`s corresponding to the presence of
   an `#ifdef`.
2. Output some external metadata stating which `Block`s should or should not
   be assembled.

(CR cam: what to do about errors? `#ifdef foo; do_thing; #else ERROR #endif`
isn't too uncommon)

### Other notes

The design of "deferred" values was inspired by "staged programming",
particularly [LMS](https://scala-lms.github.io/). The system described in this
document is much less powerful than a proper multi-stage system, but going
further down that direction would be an interesting path.
