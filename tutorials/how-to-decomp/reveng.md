# Reverse Engineering

Cool, we know how to look at vanilla code. But that's not all that useful when
90% of projects these days are using the SkillSystem(s) patch, and are
otherwise so covered by patches written in 2017 that the decomp won't be
helpful, right...?

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
times. I also don't really find debuggers to be particularly fun to use, even
though I've had to get good at them over my tenure.

And yet, in this hobby, the current reality is that there's a lot of assembly
flying around. There are efforts to "modernize" and rewrite old functionality
in C, but my money is that the use of assembly in GBAFE hacking will probably
outlive my presence in this community. And so, it pays to know how to draw
parallels from someone else's assembly to high-level C.
