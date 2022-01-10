
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
#### If you did push lr and you do have a return value
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

### Use lyn!
[lyn](https://feuniverse.us/t/ea-asm-tool-lyn-elf2ea-if-you-will/2986) is a linker by StanH that allows you to assemble your thumb code in a slightly different way than the old fashioned way with `#incbin`ning DMP files.
lyn makes good use of the ELF object files to allow EA labels to be visible to your thumb code and your thumb labels to be visible to your EA buildfile.
Older code requires a hacky approach of manually "cooking" the literal pool. lyn handles this more elegantly.
Because of this, if your code is designed for DMP insertion, it's a headache for other people to modify and reimplement.

### Use consistent spacing and tabbing
If you've worked with thumb before, you've probably noticed that my spacing is weird. To be honest, it's that way because of notepad++ syntax highlighting being weird.
I think it's fine, and the important thing is that I've stuck with it for years.
If your style is consistent, it makes your code more readable!
Furthermore (especially in a small community of wizards), if you've read a lot of other people's code, you can tell who wrote what you're looking at which is helpful to know.

On the topic of tabbing, I find tabbing for thumb to be useful in a limited way. I tend to for easy logical flow clarification and loops.
Again (see the pattern?), however you prefer to do it is mostly arbitrary, except it's best to keep it consistent!

## Helpful methods - keep it consistent! 
wip

## An example function
I was recently asked to help an apprentice with a small battle calculation hack: Make thunder tomes deal additional damage (+1/2 user's power stat) to armored units.
I thought this would be a good opportunity to show what a workflow with good convention looks like. Here's how I start.
```
.thumb

.global ThunderBuff
.type ThunderBuff, %function
ThunderBuff: @ Called from the prebattle calc loop. r0 and r1 have battle structs to work with.
```
This function is called `ThunderBuff`, and the `.global` and `.type` lines are used by lyn to make `ThunderBuff` visible to Event Assembler.
I intend for this function to be called from the prebattle calc loop. Okay, that has two parameters that are battle struct pointers.
Let's start with a comment stating exactly how I intend to _use_ this function and what its _parameters_ are.
```
.thumb

.global ThunderBuff
.type ThunderBuff, %function
ThunderBuff: @ Called from the prebattle calc loop. r0 and r1 have battle structs to work with.
push { r4, r5 }
mov r4, r0 @ Battle unit 1.
mov r5, r1 @ Battle unit 2.

pop { r4, r5 }
bx lr
.ltorg
```
Next, I consider what I intend to do. I happen to know that (
[Teq doc](https://www.dropbox.com/sh/zymc1h221nnxpm9/AACrgal3LFRvbDKL-5qDxF3-a/Tequila/Teq%20Doq?dl=0&subfolder_nav_tracking=1)
is an excellent place for documentation for this kind of thing) that a battle unit pointer + 0x5A (short) has the current damage this unit deals.
I'll keep that in mind as my goal variable I want to edit. Furthermore, this function has no return value.
I don't intend on pushing `lr`, so I've chosen the appropriate return signature. I've also gone ahead and pushed `r4` and `r5` and put my parameters in place.

As a quick aside, many functions that are called from the prebattle loop call the parameters "attacker" and "defender," but that's not quite accurate.
How it works is that the prebattle loop is performed twice:
Once with `r0` = the _attacker's_ battle struct, `r1` = the _defender's_ battle struct;
Once with `r0` = the _defender's_ battle struct, `r1` = the _attacker's_ battle struct.
By consequence, our little function will work on both the attacker and defender.
If we wanted to make a function that _only_ applied to the attacker, then we would need to explicitly check for the parameters being oriented correctly.

Anyway, we have our basic function, but two more quick things before progressing.
First, `.ltorg` tells the assembler that it's okay to put a literal pool here. It's good practice to put that at the end of a function, even if it's your only function.
Secondly,
### Don't forget .thumb !
If you don't have `.thumb` at the top of your file, then the assembler will think you're writing ARM! Unless you're weird (like Stan) you're probably not trying to write ARM.

```
.thumb

.macro blh to, reg
    ldr \reg, =\to
    mov lr, \reg
    .short 0xF800
.endm

.equ GetItemIndex, 0x80174EC

.global ThunderBuff
.type ThunderBuff, %function
ThunderBuff: @ Called from the prebattle calc loop. r0 and r1 have battle structs to work with.
push { r4, r5, lr }
mov r4, r0 @ Battle unit 1.
mov r5, r1 @ Battle unit 2.

@ First, check to see if our unit is using a thunder tome.
mov r1, #0x4A
ldrh r0, [ r4, r1 ] @ Equipped weapon.
blh GetItemIndex, r1

pop { r4, r5 }
pop { r0 }
bx r0
.ltorg
```
Woah a lot's happened. What happened? I changed my mind about needing to call a function.
I've decided that I need to call `GetItemIndex` in order to turn my equipped item/durability short into an item ID. Here's what I did:
 - I declared my `blh` macro. If you need to longcall with `blh`, you'll need this macro!
 - I declared what `GetItemIndex` is with `.equ`. Unless you do this, lyn will think that `GetItemIndex` is declared as a label at the EA level (which is likely isn't).
 - I pushed `lr`. This is necessary if you call a function.
 - I changed my return signature to account for `lr` being pushed. Note that I still don't have a return value. I just need to `bx` with `r0` instead of with `lr` directly.
 - I used `blh` to call `GetItemIndex` with `r1`. `blh` needs a free register, and I consistently use the next free scratch register.
 This isn't a hard convention, but whatever you do, do it consistently!

```
@ First, check to see if our unit is using a thunder tome.
mov r1, #0x4A
ldrh r0, [ r4, r1 ] @ Equipped weapon.
blh GetItemIndex, r1
@ Is this item ID a thunder tome?
ldr r1, =gThunderList @ 0-terminated list of item IDs we consider thunder tomes.
ThunderLoop:
	ldrb r2, [ r1 ]
	cmp r2, #0x00
	beq End @ This item ID isn't in the list, end.
	cmp r1, r2
	bne ThunderLoop @ If these item IDs aren't equivalent, try finding another.
```
I haven't changed anything outside of this block - all I've added is this loop. Your function probably has a condition of some sort. Comment as appropriately as you see fit.
I hope that these comments make it clear what I'm trying to do here. I'm assuming there's a list written in an EA script somewhere in this buildfile looking something like
```
gThunderList:
BYTE Thunder Bolting 0x0
```
Because of the magic of lyn, `gThunderList` is visible within our assembly script! I'm checking to see if the equipped item ID is in this list.
If not, end and do nothing. If so, continue doing things.

```
@ Is the enemy an armored unit?
ldr r0, [ r5, #0x04 ] @ Pointer to ROM class data.
ldrb r0, [ r0, #0x04 ] @ Class ID.
ldr r1, =gArmorList @ Same idea as before, but we're checking for an armored class ID.
ArmorLoop:
	ldrb r2, [ r1 ]
	cmp r2, #0x00
	beq End @ This class ID isn't in the list, end.
	add r1, r1, #0x01
	cmp r0, r2
	bne ArmorLoop
```
I've now added this loop in order to check for the unit in `r5` being armored. It's the same idea as the thunder condition.
Notice how I've chosen to format the code in a similar way. This shows that the appearance of the code reflects its function. Overall, this improves readability.

```
@ Add half pow to the r4 battle unit.
ldrb r0, [ r4, #0x14 ] @ Power.
lsr r0, r0, #0x01 @ Divide power by 2.
mov r2, #0x5A
ldrh r1, [ r4, r2 ] @ Current attack.
add r1, r0, r1 @ Add half power to attack.
strh r1, [ r4, r2 ] @ Store the new attack.
```
Finally, I've added this final block that executes the effect, adding half power to attack.
It's good to leave comments with piecewise calculations like this, especially with bitshifts which are particularly confusing.

With everything said and done, my whole function looks like
```
.thumb

.macro blh to, reg
    ldr \reg, =\to
    mov lr, \reg
    .short 0xF800
.endm

.equ GetItemIndex, 0x80174EC

.global ThunderBuff
.type ThunderBuff, %function
ThunderBuff: @ Called from the prebattle calc loop. r0 and r1 have battle structs to work with.
push { r4, r5, lr }
mov r4, r0 @ Battle unit 1.
mov r5, r1 @ Battle unit 2.

@ First, check to see if our unit is using a thunder tome.
mov r1, #0x4A
ldrh r0, [ r4, r1 ] @ Equipped weapon.
blh GetItemIndex, r1
@ Is this item ID a thunder tome?
ldr r1, =gThunderList @ 0-terminated list of item IDs we consider thunder tomes.
ThunderLoop:
	ldrb r2, [ r1 ]
	cmp r2, #0x00
	beq End @ This item ID isn't in the list, end.
	add r1, r1, #0x01
	cmp r0, r2
	bne ThunderLoop @ If these item IDs aren't equivalent, try finding another.

@ Is the enemy an armored unit?
ldr r0, [ r5, #0x04 ] @ Pointer to ROM class data.
ldrb r0, [ r0, #0x04 ] @ Class ID.
ldr r1, =gArmorList @ Same idea as before, but we're checking for an armored class ID.
ArmorLoop:
	ldrb r2, [ r1 ]
	cmp r2, #0x00
	beq End @ This class ID isn't in the list, end.
	add r1, r1, #0x01
	cmp r0, r2
	bne ArmorLoop

@ Add half pow to the r4 battle unit.
ldrb r0, [ r4, #0x14 ] @ Power.
lsr r0, r0, #0x01 @ Divide power by 2.
mov r2, #0x5A
ldrh r1, [ r4, r2 ] @ Current attack.
add r1, r0, r1 @ Add half power to attack.
strh r1, [ r4, r2 ] @ Store the new attack.

End:
pop { r4, r5 }
pop { r0 }
bx r0
.ltorg
```
