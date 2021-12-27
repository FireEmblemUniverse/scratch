
# Thumb convention
I (Snek) am going to dump some thumb wizardry convention info here.
The lack of consistent convention in the FE(GBA) ROMhacking community is an ongoing problem that I hope to remediate here.
Disclaimer: Although I am experienced in the FE8 wizardry scene, I have little formal computer science education. My terminology is likely a little rough.

I'm going to expect that the reader has a decent understanding of thumb, and I refer the reader to [Teq's Assembly Guide for Dummies](https://feuniverse.us/t/gbafe-assembly-for-dummies-by-dummies/3563)

## Convention? Sounds boring...
It probably is unless you're me... which seems unlikely. Convention is cool and very useful!
While there are many things that you CAN do in thumb, that doesn't mean you SHOULD do them.
Convention constitutes a set of rules that people and compilers follow. With these rules, you can have solid expectations about how functions work and how to use them.
The convention that I will cover closely matches vanilla FEGBA plus some extensions that are useful for hacking.

## Things always to do and not to do

### Registers - not all registers are treated equally
We have 16 registers at our disposal. (I'm discounting the CPU status ones.) Well, because of convention, it's really 13... really 12 if you consider `ip` reserved for longcalling.
The register allocations are as follows:

| Register | Name | Purpose                                    |
|----------|------|--------------------------------------------|
| r0 - r3  |      | Lo scratch                                 |
| r4 - r7  |      | Lo nonscratch                              |
| r8 - r10 |      | Hi nonscratch                              |
| r11      | fp   | Frame pointer (functionally hi nonscratch) |
| r12      | ip   | Hi scratch (intraprocedural longcall)      |
| r13      | sp   | Stack pointer (reserved)                   |
| r14      | lr   | Link register (reserved kinda sorta)       |
| r15      | pc   | Program counter (reserved)                 |

The most important takeaways here are _Scratch_, _Nonscratch_, and _Reserved_.
 - Scratch: A subroutine is NOT expected to maintain this register.
 - Nonscratch: A subroutine IS expected to maintain this register.
 - Reserved: You should not try to modify these registers directly.
These terms should make a little more sense when we get to setting up and finishing up functions.

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
What if we want some _parameters_? Our convention dictates that parameters are passed in through r0, then r1, then r2, then r3, then in the stack.
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

#### Pushing hi registers and allocating stack space
What do you do if you run out of places to put variables? What if you've used up r4 - r7 but need more long term storage? Use hi registers!
Unfortunately, pushing hi registers is a little bit more of a headache because of the limited nature of the thumb set.
Instructions are only a short, so possible opcodes are fairly limited. Operations with hi registers are limited, so use r4 - r7 first.

There are no opcodes defined for pushing r8 and higher directly. To get around this, we can use an intermediate lower register:
```
FuncStart: @ Say you plan on using r8 and r9.
push { r4 - r7, lr } @ I'm assuming you plan on using r4 - r7 as well.
mov r4, r8 @ Move what is currently in r8 and r9 into lower registers that we've already pushed.
mov r5, r9
push { r4, r5 } @ Now push what was in r8 and r9 for retrieval later.
```

And what's this about the stack? Doesn't `push` already do stuff with the stack for me?
Yes it does, but there are many reasons why you may want to allocate stack space for yourself otherwise.
You might need more variables than your registers allow.
You might need an area of RAM for a working `struct` variable.
Maybe a subroutine you need has more than 4 parameters.
Whatever you want to do, here's how you allocate stack space:
```
FuncStart:
@ Finish pushing all of your registers.
sub sp, sp, #0x10 @ This will allocate 0x10 bytes of the stack for you.
```
The stack is weird because it grows backwards. Don't worry about it. The consequence of this is that we subtract from the stack pointer to give ourselves room to work with.

### Functions - finishing up
So you've set up your function and you've finished doing whatever your function does. Great! Now what? We have conventions that dictate exactly how to end a function.

1. Did you allocate any stack space?
	- `add sp, sp, #XX` where `XX` is how much space you allocated. From my example in setting up, this would be `add, sp, sp, #0x10`.
2. Did you push any hi registers?
	- `pop` into a lower register and `mov` each value back into the correct hi register. This is backwards of setting up hi registers.
3. Did you push any lo registers?
	- `pop` them back _except_ lr
4. Perform the appropriate return style I'm about to describe.

For example, if I want to set up my return with 2 high registers and 0x10 bytes stack allocation, I would do:
```
add sp, sp, #0x10
@ This following step is NOT popping what was in r4 and r5 back.
@ This is popping what was in r8 and r9 into r4 and r5. Recall that we can't pop directly into hi registers.
pop { r4, r5 } @ If I'm at the end of my function, I shouldn't need my nonscratch registers anymore.
mov r8, r4 @ This resets r8 to what it was before.
mov r9, r5 @ This resets r9 to what it was before.
pop { r4 - r7 } @ Suppose I pushed r4 - r7 at the start. Now I can pop them all back.
```

Finally, we handle the return itself.

*In all cases, your return value should be in r0!*
The only exception to returning a value not in r0 that is defined by convention is returning a `long long`, a 64 bit variable, which you should never really do.
I think something else dumb happens if you try to return a structure instead of a pointer to a structure but don't do that either.

If you find yourself needing to return more than one value, consider:
 - Bitpacking your return into one r0. You have 32 bits to work with. If you can return 2 shorts in r0, that's fine.
 - Don't return a value. Instead, have your function accept a pointer to RAM and fill it with things. Vanilla support routines do this, for example.
 - Refactor your code. It's also likely that there is a more elegant way to design your system.

These cases depend on whether you're returning something and whether you pushed lr in preparation for a subroutine.
#### If you did not push lr
All you need to do is:
```
bx lr
```
This case does not depend on whether you have a return value.
#### If you did push lr and you do not have a return value
`pop` what was in lr at the start of your function into r0, then `bx r0`
```
pop { r0 }
bx r0
```
### If you did push lr and you do have a return value
Same as the previous case, except `pop` into and `bx` with r1 so that you don't clobber r0.
```
pop { r1 }
bx r1
```

*Why is keeping these return cases consistent important? Why wouldn't I always return with r1, or r3 or something?*

The answer to this is evident if you've ever researched/debugged at the assembly level.
If you're unsure exactly what a function does, this convention leaves clues for a human to figure out the return signature.
If you're working in a debugger and you see what an unknown function returns with its return address in r0, you know it has no return value.
If you're glancing at another function's return and see that it uses r1 to return, you know for sure that it does have a return value.
One of the biggest benefits of consistent convention is that a skilled human can glean numerous clues about a function's behavior from otherwise indecipherable machine code. 

## Things you really really should and shouldn't do
There's a lot of things we've discussed that you absolutely need to keep straight for decent functions.
Now I'm going to go into a few things that aren't quite as important but should still be considered.

### Only push and pop when you need to
*But Sneeeeeekkkkkkk, it's easier to just always push r4 - r7 and r14 so I don't have to think about it!*
Fair enough, but consider:
 - Anything that loads and stores is a relatively slow process.
 I know for GBAFE hacking, speed is only terribly relevant in certain situations, so the next reasons are more important practically.
 - From my experience, if you're not thinking about something in the process of writing code, that's a problem. It's more valuable to be more explicit with everything you do.
 - It makes your code more readable. A reader would find it more obvious what registers are in use.

## Helpful methods - keep it consistent! 
wip

## An example function
wip
