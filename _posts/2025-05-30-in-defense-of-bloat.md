---
layout: post
title: "In defense of bloat"
params:
    license: CC-BY-4.0
draft: true
date: 2025-05-30
permalink: /in-defense-of-bloat.html
excerpt: \"bloat\" as a concept is ahistorical bullshit.
---

A motif certain pundits in the Linux sphere will repeat
(in and amongst the *literal fascist rhetoric*)
is that software bloat cannot be allowed to exist.
It makes your computers slow,
it makes your programs have featuresets they don't need[^on-need],
it took your kids in the divorce...

I think what they're saying is **ahistorical bullshit**.

# The IBM PC kicked off the PC revolution, not the Commodore 64

At risk of inflicting psychic damage on some enthusiasts,
I'm sorry, the 'early' home computers were toys.
The C64, in the modern sense of the term,
does not qualify as a "computer".
While the same criticism could be levelled at
the IBM PC up to the 286/386 era,
the C64 simply never had any concept of persistence.
They also couldn't be expanded any further
than what was in the box in 1982.

I think part of this is the obvious fact that the
Commodore didn't ever have a hard drive,
but I think there's also another fact:
the IBM provided a lot of freedom to be wasteful.

The [C64's memory map](https://www.c64-wiki.com/wiki/Memory_Map),
all 64 kilobytes of it, is *packed*.
It's got memory-mapped IO intruding into the actual RAM,
which produces strange patterns like
"if you read that address range,
you're reading the BASIC ROM,
but if you *write* to it,
that's actually the underlying memory you're writing to".
Already, we can see a problem:
the limited address space
makes it harder to access the RAM you paid for.
Even the burned-in BASIC can tell you this
with all of its 38911 bytes free.
Part of this is overhead, of course, but
it's also the longest contiguous region of RAM.

By comparison,
the IBM PC has a whole megabyte of address space,
a truly "bloated" design decision for its time
(these things could ship with 16 kB!),
but its benefits are twofold:
peripherals could be shuffled off to a 384 kB
"upper memory area",
which would've left you with a truly luxurious 640k
of 'real' RAM address space to eventually work with.

Boy howdy, did they work with that RAM.

Lotus 1-2-3,
*the* prototypical killer app for the IBM PC,
needed 192k in its earliest release[^lotus123].
Would it have been possible to fit its features into 64k?
Maybe, they did make a PCjr version that fit in 128k,
but consider:

# Products, not art pieces

I don't like that this is the reality,
but we do need to be clear on this:
commercial projects live and die on timelines.
The demoscene is impressive, don't get me wrong,
but commercially, they're toast.
30 years to do
[8-bit sample playback](https://csdb.dk/release/?id=115651)?
Get Real.
In a mere *decade* after the C64's release,
Creative put out the Sound Blaster 16,
capable of all the CD-quality
sample playback one could ever desire.

Nobody waited for the institutional knowledge
around the C64 demoscene
to build up, just to see that effect come to fruition.
Instead, people kept pushing for hardware of the future
so they could stop being limited by the hardware
of their time.

This *is* bloat, yes.
This is requiring a larger hardware footprint
in anticipation that people will eventually
gain access to newer, better hardware.
While perhaps it's possible to over-apply this thinking
(modern AAA gaming invariably requiring hardware
a majority of players simply do not have),
you can't exactly look forward
when you're shackled to the present.

Certainly the people who wrote the Unix Hater's Handbook
were (and perhaps understandably so)
unkind to X11's system requirements,
but what kind of forward thinking could
Project Athena really have pulled off
had they been told to optimize for 8088s with Hercules cards?

Sierra games did not target CGA
because they knew it was the done thing by then.
Instead, they looked towards the future
by designing for these newfangled 16-color cards,
and gracefully degrading when the player's hardware
was *not* an EGA or Tandy.
And you know what?
By pushing the envelope,
they produced a demand for better hardware
that otherwise would've taken longer to materialize,
had it materialized at all.

# The problem is economic

I'd like to make it clear that demonizing devs is Bad™.
I think the blame has to be
laid at the feet of the fools well above them, ultimately;
managers are one thing,
but unreasonable bosses who demand ungodly deadlines,
force constant overtime,
and impose whatever the fuck "Agile" is --
it's abuse by any other name.
These bosses, ultimately,
[do not do any work](https://www.wheresyoured.at/the-era-of-the-business-idiot/);
they exist to extract labor from their peons,
and then have the audacity to be baffled
when nobody wants to work anymore.

Given this environment,
of course 'bloat' as a concept would fall by the wayside.
Of course the modern web ends up
[being downright unusable](https://danluu.com/slow-device/)
on an N4020 or a quad-core Cortex-A7.
Of course Cities Skylines 2 gets released and ends up
[being unplayable on anything short of an RTX 3080](https://gamersnexus.net/game-benchmarks-graphics-guides/terrible-optimization-cities-skylines-2-gpu-benchmarks-graphics)
[^cs2].

There's just no incentive to Do It Right.
I think any solution to bloat, *real* bloat[^real-bloat],
will need a change in the economics.
I don't have a solution here,
as much as I'd like to have one.
I don't know how you wedge in optimization into projects,
but it
[can be done](https://shkspr.mobi/blog/2021/01/the-unreasonable-effectiveness-of-simple-html/),
surely,
perhaps under the name of "digital accessibility" or
"nobody's gonna be able to play our game, dipshit!".

We'll have to.
Despite PassMark ratings on CPUs rising faster than ever[^strix-point],
[PC sales are slowing down](https://www.pcmag.com/news/pc-market-sees-slow-growth-as-ai-features-fail-to-ignite-sales),
as it turns out that [The Plateau](https://youtu.be/v8tjA8VyfvU)
gets ever-longer.
This is good, actually,
because it means that even decade-old machines
can have a decent time in the modern world.
Computing is more accessible than ever because we're all *gluttons*.

Of course, you have to cut the long tail off eventually,
but I think it would be a damn shame
if these computers had to go to landfill needlessly
just because we couldn't love them no more.

# Footnotes

[^on-need]: "need" here is sometimes coupled
    to a thought-out understanding of the purpose of programs
    but oftentimes it can just as easily mean
    "whatever I feel programs 'should' be doing",
    a truly vibe-based, unprincipled assessment
    from people who pride themselves on "objectivity".

[^lotus123]: I specifically referred to the version 1.0A manual.

[^real-bloat]: As opposed to the "bloat" that makes you uninstall vital parts
    of your OS on the basis that it's "taking up too many resources", despite
    the fact that you kinda *needed* that functionality.

    Plus, even if you don't break the OS doing this,
    I'd like to impart some wisdom:
    don't do work for free, kids.

[^cs2]: A performance tier only 6% of Steam users had at time of release!

[^strix-point]: Have you seen Strix Point? 50k out of a souped-up laptop chip, imagine that!
