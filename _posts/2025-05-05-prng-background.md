---
layout: post
title: "Some background on pseudorandom number generation, with respect to Linux"
date: 2025-05-05
params:
    license: CC-BY-4.0
permalink: /prng-background.html
excerpt: Not everyone knows what's up with (P)RNGs, so here's a primer.
---

<div class="altcontext" markdown="1">
This article was spun off from an infodump section of
[*What does "Saving 256 bits of creditable seed" mean?*](what-does-saving-creditable-seed-mean.html)
that never was.
I've tried to make it readable stand-alone,
but no guarantees.
Refer to that article for context.

[For those that got here from that article, here's a way back.](what-does-saving-creditable-seed-mean.html#the-program)
</div>

Not everyone knows what's up with (P)RNGs, so here's a primer.

The following is a synthesis of what I've picked up over time.
I wish I could cite every source I read relevant to this topic,
but I really have forgotten.

# Computers are deterministic

Computers, unless they're broken, are deterministic.
On its face, this should be obvious enough, but consider what this means for
anything on a computer that needs random numbers: they shouldn't be possible.
What do you mean, some input goes in,
and a *different* output from last time comes out?

So how do we cope? I've just told you
[Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_algorithm) and
[Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_Casino)
shouldn't be possible.

Well... computers don't have eyes or ears, but they do, usually, come with peripherals.
Desktops and laptops will have mice and keyboards,
and even servers probably come with network cards
that'll send or receive a packet from time to time.

# But there's an outside world

So, why am I telling you this?
It's great that computers can sense some kind of world outside of them, but
it's trivia without a second insight.

Without getting too deep into the weeds, all of these peripherals produce,
let's call them, "events".
They all generate an "event" for the computer to handle.
When I move my mouse, it produces an "event" that says that at time T,
the mouse is moving by so-and-so arbitrary units in the X direction, and
so-and-so arbitrary units in the Y direction.

What's important here is that they all produce numbers, and often at
ridiculous precision for what they actually need to do.
Even if they contain no other data, those events come with timestamps.
Computers these days have hardware timekeeping devices
("clocks") so precise that the smallest unit they can keep track of
is in the *micro*seconds.

<div class="panel-warning-half" markdown="1">
## **Lie-to-children**

What I've just described is a mishmash
of at least 2 ways the Linux kernel
collects entropy.

The Linux kernel can also get its entropy from interrupts, timer delays,
seek times on hard drives, and even dedicated hardware RNGs.
The kernel is also capable of extracting entropy from the CPU itself.[^haveged]

This diversity of entropy sources affords the kernel resistance
against attempts to manipulate its RNG,
since even if you seed it with data you
know (e.g. via a "faulty" hardware RNG),
there's still the other 5+ sources you need to account for, *including*
the very CPU it's running on.
</div>

These digits at the end ("least significant digits") are basically
impossible to control, and this makes them effectively
random. Of course, there's some statistical magic that needs
to be done to "clean" those values for proper randomness,
but you'd have to ask someone else for that.
I'm assuming that magic works; the whole world relies on it!

# Not much of an outside world, though...

So, you have all this randomness to play with. Great.
But that's not actually enough, because while these events produce
a reliable source of randomness, they also don't produce *much*
relative to what users ask `/dev/urandom` to shovel out the window.
Recall the last time you ran `dd if=/dev/random`
and made the `of=` parameter a disk.
You may have wanted
[ShredOS](https://github.com/PartialVolume/shredos.x86_64)
there, but no judgement.

Enter the Pseudo-Random Number Generator (PRNG).
The Lore™ on these is deep, but let's focus on the Gameplay.
A PRNG is 2 things:

* the last number it generated ("state")

* a "seed" value

For a fixed seed value, a PRNG produces some shuffling
(the standard term is a "cycle") of a range of numbers.
The length of that shuffling is called its "period",
and it's one of the factors that determines how good the PRNG is.
All else being equal, the longer it is, the better it is.

This is why, if you've played Minecraft,
you can replicate someone else's world if you know its seed.
The shuffling itself is random,
but you have the *exact* seed that produces *that* specific shuffling.

# Who says we can't synthesize one?

It'd be catastrophic if we all set the seed value to a fixed value;
imagine getting the same SSH private keys each time we all ran `ssh-keygen`.
Clearly, this isn't what we do.

Instead, let's make use of this conveniently-placed trickle
of "true" randomness.
Seeds aren't that much data;
the size that the Linux kernel uses is 256 bits, or 32 bytes.
The Linux kernel may not collect much entropy with the method I just described,
but it's definitely collecting enough to fill 32 bytes.

You might scoff at that, but consider what 256 bits would physically represent:

<div class="altcontext" markdown="1">
Imagine two rooms of 256 light switches. The first has them all turned off,
but the other has them turned on and off in a random configuration.
You know nothing else about it.
You can't see the second room at all.

The only thing you know is that a robotic voice will congratulate you
if you match the first room's switches with the second.
</div>

If you work through the problem,
you'll have 2^256 (or about 1 with 77 zeros after it)
configurations (more properly "permutations") to work through.
Good luck!

This is, simply put, **not possible**.
Even if those switches were 100 THz transistors,
the universe would end first.
If the PRNG is any good, there is no correlation
at all between the possible seed values;
there is no pattern in the shufflings
and there is no pattern *between* the shufflings.

But there's still a flaw here.
Filling those 256 bits with entropy
and then using that as the only seed means that,
in theory, it's possible to figure out what seed was being used.
If you planted 256 bits of all-0s into the
on-disk entropy stash that `seedrng`
or `systemd-random-seed.service` make use of,
that'd be a real disaster.
You'd be able to predict the contents of
the keys and certificates the machine will end up generating.

<div class="panel-info" markdown="1">
On a less serious note,
exploiting RNG seeding schemes that only seed once,
or seed in a manner that can be controlled,
are the basis for RNG manipulation in (older) games.
[The Smogon guides for RNG manipulation in Pokémon games](https://www.smogon.com/ingame/rng/)
are a real hoot.
</div>

# Who said we can't jump between them?

And now, finally, after these thousand words,
I can explain the Linux entropy pool.

The entropy pool, as far as I understand it,
is the seed the Linux RNG works off.
It is continually refreshed ("reseeded")
with new entropy as it comes in.
Even *if* you could figure out the seed,
Linux will just change it from under your feet
fast enough[^reseed-interval] that no exploit could take place.
Additionally, since the Linux RNG is used by
everything on your system from OpenSSL to
you wiping your disk with `dd`,
trying to actually "land" at any specific
"place" in the RNG's shuffling is so
improbable that it *is* impossible.

This makes it much, much harder to manipulate the RNG.
You now need total, single-step control (possibly even more than that!)
over the system, as well as its peripherals and its environment.
Barring that, you would need to find the seed within the span of the
reseed interval[^reseed-interval], *and then* figure out where the
RNG is in its cycle.

To put it bluntly: **you'd have to be God**
to manipulate the Linux kernel's RNG.

# Footnotes

[^haveged]: Nearly all machines in use today use
    pipelined, speculatively-executing, out-of-order, multi-core CPUs whose
    internal machinations are so complex
    that you would need to be omniscient to figure out what's going on.
    The days of predictable instruction timing, being able to "race the beam",
    are long over.

    This complexity can be exploited by the
    [HAVEGE algorithm](https://www.irisa.fr/caps/projects/hipsor/publications/havege-tomacs.pdf),
    which is available as a
    [userspace daemon](https://github.com/jirka-h/haveged)
    that seeds the RNG with its output.
    Linux kernels 5.4+ also contain a similar algorithm.

[^reseed-interval]: I think reseeds happen every minute, as of 6.14?
                    I can't tell what unit it's using.
