---
layout: post
title: What does "Saving 256 bits of creditable seed" mean?
date: 2025-05-03
permalink: /what-does-saving-creditable-seed-mean.html
---

This is a tiny mystery, but one nobody has bothered to document.
I've tried to solve it to the best of my abilities.

You may have seen the following message at boot.

```
Saving 256 bits of creditable seed
```

I've seen it on Alpine, but other OpenRC systems
(and other inits? I haven't checked) will apply.

For completeness, the equivalent service on systemd is
[`systemd-random-seed.service`](https://www.man7.org/linux/man-pages/man8/systemd-random-seed.service.8.html).
You may have seen this service under its description,
"Load/Save OS Random Seed".

# It's not the Linux kernel

I've searched! It's not the Linux kernel that does this.
Try `grep -R creditable`, `grep -R 'creditable seed'`, `grep -R 'seed'`,
it's not going to tell you it's there.

To save you some time, it's an OpenRC service and program.

# The service

The service responsible is called `seedrng`. It's a pretty straightforward service:

```sh
seedrng_with_options()
{
        set --
        [ -n "${seed_dir}" ] && set -- "$@" --seed-dir "${seed_dir}"
        yesno "${skip_credit}" && set -- "$@" --skip-credit
        seedrng "$@"
}

start()
{
        ebegin "Seeding random number generator"
        seedrng_with_options
        eend $? "Error seeding random number generator"
        return 0
}
```

So it starts up, runs `seedrng_with_options`,
massages the configuration into command-line args to pass,
and runs a program called `seedrng`.

# The program

`seedrng`'s purpose is obvious enough
if you have the background knowledge:
it's a utility to seed the Linux kernel random number generator.
In specific, it handles both *creating* a seed file for next boot,
and for using that seed file to seed the RNG on boot.

[random(4)](https://www.man7.org/linux/man-pages/man4/random.4.html)
(or `man 4 random` on your Linux machine)
describes the rationale for this setup:

> When a Linux system starts up without much operator interaction,
> the entropy pool may be in a fairly predictable state. This
> reduces the actual amount of noise in the entropy pool below the
> estimate. In order to counteract this effect, it helps to carry
> entropy pool information across shut-downs and start-ups.

So this line right here is the famous "creditable" line:

```c
	einfo("Saving %zu bits of %s seed for next boot", new_seed_len * 8, new_seed_creditable ? "creditable" : "non-creditable");
```

# What does "creditable" mean?

Let's cut straight to the code.

Seed file read into memory, `seedrng` constructs a **req**uest
struct in the format that the Linux kernel wants.

```c
static int seed_rng(uint8_t *seed, size_t len, bool credit)
{
	struct {
		int entropy_count;
		int buf_size;
		uint8_t buffer[MAX_SEED_LEN];
	} req = {
		.entropy_count = credit ? len * 8 : 0,
		.buf_size = len
	};
```

It checks that the to-be-written seed
won't overflow the fixed-length buffer made for it,
then if it won't, it copies the contents of `seed` into `req.buffer`.

```c
	if (len > sizeof(req.buffer)) {
		errno = EFBIG;
		return -1;
	}
	memcpy(req.buffer, seed, len);
```

It then yoinks `/dev/urandom`'s file descriptor,
runs this funny little ioctl on it,
using that `req` struct as input.

```c
	random_fd = open("/dev/urandom", O_RDONLY);
	if (random_fd < 0)
		return -1;
	ret = ioctl(random_fd, RNDADDENTROPY, &req);
```

<div class="panel-info" markdown="1">

### **What's an ioctl?**

An [ioctl(2)](https://www.man7.org/linux/man-pages/man2/ioctl.2.html)
is a very generic mechanism, but in essence it's a means of
sending a request to a device driver[^device-driver]
(via a device node) to Do Stuff with it.

The first argument is a file descriptor to a device node,
and the second is the moral equivalent of a (per-device driver)
enum representing the operation you would like to perform on it.
What the rest of the arguments mean,
what their types are,
and how many there are,
depend on what operation is being performed in the first place.
It's the price of genericity.

It's something like calling a method on an object,
but your objects are device nodes.

For instance, the `ioctl(random_fd, RNDADDENTROPY, &req);`
we just saw is effectively saying "I would like to call the
`RNDADDENTROPY` "method" on `/dev/urandom`, using `&req` as my argument".
</div>

And with the nitty-gritty described, let's step up a layer of abstraction
to disappoint you and say I *don't* know what crediting does.

`RNDADDENTROPY` there is an operation that adds additional entropy to
the entropy pool, using that `req` struct. It does this by loading
`.buffer` (the actual data) into the entropy pool, incrementing (or decrementing!)
the kernel entropy count by the amount given in `.entropy_count`.

This, thus, is what "creditable" seems to mean: non-creditable entropy
does not directly add, or "credit" into the entropy pool's count.
Non-creditable entropy adds to the entropy pool,
but doesn't change its count.

What I don't know is when to choose between crediting and non-crediting,
but here's what I know so far:

The Linux kernel, as of version 5.6, does not block reads
from `/dev/random` no matter how fast you read it...
but there's exactly one case where this is not true:
before the kernel RNG's entropy pool is initialized,
an event that happens only once.
You start the machine, Linux boots,
and it initializes the entropy pool at some point.
Once it's done, you can't *un*initialize it short of
turning the machine off.

Unless the Linux kernel is more foolish than I thought, it surely won't
believe the value of `.entropy_count` at face value, will it?

So what's going on here?
What's the purpose of crediting?
Is it merely a signal to the kernel that it's allowed to ignore if it
wishes, or is it something the kernel will die on a hill for, regardless
of what data was actually in `.buffer`?

With that said, here's what I can say with some confidence.

# Conclusion

```
Saving 256 bits of creditable seed
```

is saying the truth very tersely:
it's saving 256 bits of entropy harvested from `/dev/urandom`
so that the next time the system boots,
it can seed the kernel RNG with it,
"crediting" it into the kernel's "bank account" of entropy,
so it can get the entropy pool initialized earlier.

While this is usually not a problem on desktops or laptops, it's
an important consideration on hosts with little outside entropy
(e.g. virtual machines, headless).
[A lack of entropy can stall out the startup of essential daemons like sshd](https://daniel-lange.com/archives/152-Openssh-taking-minutes-to-become-available,-booting-takes-half-an-hour-...-because-your-server-waits-for-a-few-bytes-of-randomness.html), [^daniel]
after all.

# Footnotes

[^daniel]: I consider this article dubious:
           it was from Dec 2018 and was last updated April 2020,
           and it's unclear to me whether the problems described here have
           already been fixed or not.[^systemd]
           It has, literally, been 5+ years.

[^systemd]: If crediting matters to you, and you're on systemd, consider
            looking at the `SYSTEMD_RANDOM_SEED_CREDIT` environment
            variable.
            Details in [systemd-random-seed.service(8)](https://www.man7.org/linux/man-pages/man8/systemd-random-seed.service.8.html).

[^device-driver]: To be more pedantic, "device driver" is an arbitrary concept that
                  more-or-less means
                  "whatever you can convince the kernel to load
                  that shows up as a device node somewhere".
                  You can send ioctls to things that aren't hardware,
                  not even emulations of hardware, but purely
                  internal concepts with no tangible representation in hardware.
