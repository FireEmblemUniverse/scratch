
# Thumb convention
I (Snek) am going to dump some thumb wizardry convention info here.
The lack of consistent convention in the FE(GBA) ROMhacking community is an ongoing problem that I hope to remediate here.
Disclaimer: Although I am experienced in the FE8 wizardry scene, I have little formal computer science education. My terminology is likely a little rough.

## Convention? Sounds boring...
It probably is unless you're me... which seems unlikely. Convention is cool and very useful!
While there are many things that you CAN do in thumb, that doesn't mean you SHOULD do them.
Convention constitutes a set of rules that people and compilers follow. With these rules, you can have solid expectations about how functions work and how to use them.
The convention that I will cover closely matches vanilla FEGBA plus some extenstions that are useful for hacking.

## Things always to do and not to do

### Registers - not all registers are treated equally
We have 16 registers at our disposal. (I'm discounting the CPU status ones.) Well, because of convention, it's really 13... really 12 if you consider `ip` reserved for longcalling.
The register allocations are as follows:

| Register | Name | Purpose                                    |
|----------|------|--------------------------------------------|
| r0 - r3  |      | Lo scratch                                 |
| r4 - r7  |      | Lo nonscratch                              |
| r8 - r10 |      | Hi nonscratch                              |
| r11      | fp   | Frame pointer (Functionally hi nonscratch) |
| r12      | ip   | Hi scratch (intraprocedural longcall)      |
| r13      | sp   | Stack pointer (reserved)                   |
| r14      | lr   | Link register (reserved kinda sorta)       |
| r15      | pc   | Program counter (reserved)                 |



## Things you really really should do
wip

## Helpful methods - keep it consistent! 
wip
