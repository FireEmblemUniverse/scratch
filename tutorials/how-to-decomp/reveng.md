# Reverse Engineering

Cool, we know how to look at vanilla code. Unfortunately, there's a lot of
projects running around using SkillSystem(s), and half the code we'd want to
look up is hooked into by assembly patches written in 2016. What to do?

As it turns out, the decomp is *great* for understanding other people's custom
code, even if that code is written in assembly!

Before I go any further, I want to reiterate that this text is **not** intended
to be a general introduction to assembly, or even a guide on how to use the
decomp to work with any particular project.

If either of those are what you're looking for, I'd instead recommend:

- [GBAFE Assembly For Dummies](https://feuniverse.us/t/gbafe-assembly-for-dummies-by-dummies/3563) by Tequila
- [The Skill System and You](https://feuniverse.us/t/the-skill-system-and-you-maximizing-your-usage-of-fe8s-most-prolific-bundle-of-wizardry/8232) by Sme

Moving on, I have a confession. I *hate* dealing with assembly. Reading it
gives me a headache, and I find writing it to be frustrating at the best of
times. I also don't find debuggers to be particularly fun to use, though they
genuinely can make your life easier once you reach some cursory familiarity.

Unfortunately, the current reality is that there's a lot of assembly
flying around. There are efforts to "modernize" and rewrite old functionality
in C, but my money is that the use of assembly in GBAFE hacking will probably
outlive my presence in this community. So it pays to know how to use insights
from high-level C to understand someone else's hand-written ASM.
