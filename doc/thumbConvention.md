
# Thumb convention
I (Snek) am going to dump some thumb wizardry convention info here.
The lack of consistent convention in the FE(GBA) ROMhacking community is an ongoing problem that I hope to remediate here.
Disclaimer: Although I am experienced in the FE8 wizardry scene, I have little formal computer science education. My terminology is likely a little rough.

I'm going to expect that the reader has a decent understanding of thumb, and I refer the reader to [Teq's Assembly Guide for Dummies](https://feuniverse.us/t/gbafe-assembly-for-dummies-by-dummies/3563)

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

The most important takeaways here are _Scratch_, _Nonscratch_, and _Reserved_.
 - Scratch: A subroutine is NOT expected to maintain this register.
 - Nonscratch: A subroutine IS expected to maintain this register.
 - Reserved: You should not try to modify these registers directly.
These terms should make a little more sense when we get to sett ing up and finishing up functions.

### Functions - setting up
By "functions," I mean _standard_ functions. That is, a function that a compiler would generate. These functions have clear entry points and clear returns.
I'll cover hack methods like hooks later.

If we imagine we're at the start of a function, we can suppose a few things.
 - We... probably have a parent function? Yes, someone has called us to do something. It expects some things of us (think scratch registers).
 - It might have given us some information to work with. Parameters!
 - It might expect we give it something back. A return value!
 
It sounds like we need an agreement between the function we're starting and our parent function on how to handle these. This is where convention comes in.
And that same convention will dictate how we deal with subroutines of our own. In that sense, our convention is recursive in nature.

#### Parameters
Let's start our function:
```
FuncStart:
``` 
What if we want some _parameters_? Our convention dictates that paramaters are passed in through r0, then r1, then r2, then r3, then in the stack.
If our function has one parameter, say... a unit struct pointer...
```
FuncStart:
@ Right now, r0 has the unit pointer given to us by the parent function.
```
If we want to use another parameter, that will be in r1. The next would be in r2, but what if we have more than 4?
```
FuncStart:
@ Our first 4 parameters are held right now in r0-r3. Any parameters beyond the fourth are held in the stack.
...
ldr r0, [ sp ] @ If I want to access the 5th parameter (suppose I haven't shifted the stack!), then I would use this.
ldr r0, [ sp, #0x04 ] @ This is the 6th parameter, etc.
```

#### Pushing lo nonscratch registers
This presents a problem, however. If our parameters are in r0-r3, registers that we want for arithmetic and intermediate operations, they're in the way!
To solve this, we (generally) put our parameters in non-reserved higher registers and the stack.
But... those registers are not scratch. That is, our parent function expects us to put them back the way they we found them. To do this, we use `push` and `pop`.

If you plan on using a nonscratch register, you need to push it at the start of your function.
```
FuncStart: @ Suppose I have one parameter. Recall that by convention, it will always be in r0.
push { r4 } @ This puts r4 onto the stack and saves it for us. We can get it back at the end of our function.
mov r4, r0 @ Now that r4 is saved elsewhere, we can use it too for our parameter.
```
Now here's the beauty of this convention. Because r4 is nonscratch, if _we_ want to call a subroutine, we expect the subroutine not to change r4. Convenient!
Popping registers back into place will be covered later.

#### Preparing for subroutines
Sepaking of subroutines, there's also something we need to do if we plan on calling one.
At the start of our function, r14 contains our return point for when we're finished.
We're gonna need that; however, a subroutine will need to change r14 to remember _its_ return point.
If we need to save that register for later, we `push` it like so:
```
FuncStart:
push { lr } @ Recall that lr is r14. push { r14 } works too.
```
If we want to push more registers, we can do
```
FuncStart:
push { r4 - r6, lr } @ For example, this pushes r4, r5, r6, and r14.
```

#### Hi registers

### Functions - finishing up
wip

## Things you really really should do
wip

## Helpful methods - keep it consistent! 
wip
