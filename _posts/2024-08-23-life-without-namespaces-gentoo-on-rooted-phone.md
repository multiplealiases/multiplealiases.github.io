---
layout: post
title: "Life Without Namespaces: Running a Gentoo Chroot on a Rooted Android Phone"
date: 2024-08-23
params:
    license: CC-BY-4.0
permalink: /life-without-namespaces-gentoo-on-rooted-phone.html
---

It's not quite as awful as I thought it was! But it's still not great.

# Setting up the Termux environment

Install Termux; either the F-Droid version or the GitHub Releases versions, 
but not the Google Play one. I'm going to assume you're doing this all in 
Termux, because it provides a couple of nice features that would be annoying to 
reimplement in the raw `mksh`[^1] + ToyBox environment that Android gives you.

Since you're rooted, install
[DozeOff](https://f-droid.org/packages/com.dosse.dozeoff/) to get rid of Doze
and the Phantom Process Killer, which would otherwise interfere with your very
important work. Install
[Termux:Boot](https://f-droid.org/packages/com.termux.boot/) and open it once to
allow it to start up on boot.

Install `termux-services`[^2] (`pkg install termux-services`), then enable at
least the ssh daemon (port 8022, key auth only; make your machines' SSH keys
into QR codes or something).

```console
$ pkg install openssh
$ sv-enable sshd
```

Actually, I think it might enable all possible services by default. If this is a
concern, `ls $PREFIX/var/service` to see what you have, then `sv-disable` the
ones you don't want.

Enable it at boot, for good measure.

```sh
cat << EOF > $HOME/.termux/boot/termux-services
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
. $PREFIX/etc/profile
EOF
chmod +x $HOME/.termux/boot/termux-services
```

If you don't want to keep typing on a phone, this is when you `ssh` into your
phone from one of your keyboarded devices. Consider also installing a terminal
multiplexer; I use `byobu`, but `tmux` and `screen` exist too. How you'll SSH
into your phone is up to you; I like Tailscale, but alternatives (including
setting up WireGuard manually on a VPS) exist.

# Setting up the Gentoo chroot

First, download and place down an OpenRC stage3 in a convenient location. I used
`/data/local/gentoo`. `cd` into that location and untar it with the following
command:

```console
# tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

Mount the appropriate virtual kernel filesystems and `chroot` into it.

I suggest using this script[^3] for convenience.

```sh
#!/data/data/com.termux/files/usr/bin/sh

# Copyright (c) 2024 multiplealiases
# SPDX-License-Identifier: MIT

unset LD_PRELOAD
prefix=/data/local/gentoo

die() {
    # shellcheck disable=SC2059
    printf "$@"
    exit 1
}

if [ "$(id -u)" != 0 ]
then
        die "Must be root!\n"
fi

checkmount() {
    src="$1"
    dst="$2"
    shift 2
    if ! 2>&1 1>/dev/null findmnt -M "$dst"
    then
        mount "$src" "$dst" "$@"
    fi
}
checkmount /dev "$prefix"/dev --bind --mkdir
checkmount devpts "$prefix"/dev/pts -t devpts --mkdir
checkmount shm "$prefix"/dev/shm -t tmpfs --mkdir
checkmount proc "$prefix"/proc -t proc --mkdir
checkmount sys "$prefix"/sys -t sysfs --mkdir

chroot "$prefix" env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PATH=/usr/bin:/usr/sbin     \
    "$@"
```

Save this as, say, `gentoo-prefix-chroot`, mark it executable, copy it to
`$PREFIX/bin`, then run `sudo gentoo-prefix-chroot /bin/bash --login`.

## Welcome to chroot!

You have entered the chroot.

Astute readers will have noticed that there's no `/etc/resolv.conf`. Even more
astute readers will notice Android doesn't have an `/etc/resolv.conf`. Let's fix
that. Replace with your favorite nameservers; these entries are for Quad9.

```console
# cat << EOF > /etc/resolv.conf
nameserver 9.9.9.9
nameserver 149.112.112.112
EOF
```

`ping` your favorite website to check that DNS works.

## Appeasing Portage

If you try installing anything right now, you'll see a hundred and one errors.
Disable the sandboxing features in `/etc/portage/make.conf`, since you're 
probably lacking in namespaces.

```
FEATURES="${FEATURES} -sandbox -ipc-sandbox -network-sandbox -pid-sandbox -usersandbox -userfetch"
```

You might also want to add `getbinpkg` to save some time.

## Setting up the "init"

The stage3 tarball (hopefully OpenRC, since that's what I used) includes an init
system, but not a very useful one in this context. We don't have PID namespaces,
so we can't fake a boot. OpenRC, by default, sets up udev, dmesg and virtual
kernel filesystems like it owns the place.

This might shock you, but this is not where the story ends! OpenRC can be used
as a service manager without being PID 1; `grep rc_sys /etc/rc.conf -B30` to
look at what's available.

Let's set it up for use in a prefix instead. First off, set `rc_sys` to
`prefix`, like so:

```diff
--- a/etc/rc.conf   1970-01-01 00:00:00.000000000 +0000
+++ b/etc/rc.conf   1970-01-01 00:21:52.000000000 +0000
@@ -179,7 +179,7 @@
 # "vserver"        - Linux vserver
 # "xen0"           - Xen0 Domain (Linux and NetBSD)
 # "xenU"           - XenU Domain (Linux and NetBSD)
-#rc_sys=""
+rc_sys="prefix"

 # if  you use openrc-init, which is currently only available on Linux,
 # this is the default runlevel to activate after "sysinit" and "boot"
```

There's no magic in this; the `depend()` function of OpenRC's `/etc/init.d/`
scripts will include some directives to OpenRC. Here's `/etc/init.d/agetty`'s:

```sh
depend() {
        after local
        keyword -prefix
        provide getty
}
```

The thing that tells OpenRC to not enable a given service by default for an
`rc_sys` setting is `keyword`. Here, it's saying "not for `prefix`", because
it'd be odd for a prefix to have `agetty` started.

A few more services will remain, which are all counterproductive for a chroot,
especially `udev`, which'll cause the "boot" to fail, or worse, interfere with
Android's udev. Delete those.

```console
# rc-service del kmod-static-nodes
# rc-service del udev
# rc-service del udev-trigger
```

Enable `sshd` if, for some reason, you'd like to ssh straight into this prefix.

```console
# emerge net-misc/openssh
# rc-service add sshd
```

### "Booting" the chroot

`termux-services` services run as the Termux user, so we need to execute
something of a hack to get the chroot working.

```sh
#!/data/data/com.termux/files/usr/bin/sh
# Copyright (c) 2024 multiplealiases
# SPDX-License-Identifier: MIT

prefix="/data/local/gentoo"
die() {
    # shellcheck disable=SC2059
    printf "$@"
    exit 1
}
if sudo $PREFIX/bin/gentoo-prefix-chroot /sbin/openrc sysinit
then
        printf 'Prefix at %s "booted".\n' "$prefix"
        sleep infinity
else
        die 'Prefix at %s failed to "boot".\n' "$prefix"
fi
```

`mkdir -p $PREFIX/var/service/gentoo-prefix-init/`, copy this script to
`$PREFIX/var/service/gentoo-prefix-init/run`, and mark it executable.

Don't enable it quite yet, we'll need the most scuffed possible `finish` script
for it. Place this in `$PREFIX/var/service/gentoo-prefix-init/finish`.

```sh
#!/data/data/com.termux/files/usr/bin/sh
sudo gentoo-prefix-chroot find /etc/init.d/ -executable -type f -exec {} stop \;
```

It technically does the job of stopping the prefix's services, though I despise
that it does.

Now run `sv-enable gentoo-prefix-init` and `sv restart gentoo-prefix-init` to
restart the service manager, tearing down the earlier set of enabled services
and starting the current set. I'm sorry I don't have a better solution.

## Bonus: Old kernels?

Some kernels will be so old that they can't build certain packages; mostly
graphical stuff (glib, I'm staring at you with your
[pidfd_open(2) misdetection](https://bugs.gentoo.org/927664)). In this case
you'll have to partially admit defeat and run a binhost.

If you're running aarch64 Gentoo on a machine with a modern kernel with matching
USE and CFLAGS, you can go straight to `quickpkg`-ing `@world` and, assuming
your machine runs `sshd` and you can access it from your phone, place down a
`binrepos.conf` entry that looks like

```
[yourmachine]
priority = 9999
sync-uri = ssh://user@host/var/cache/binpkgs
```

(if you use Tailscale, keep in mind that DNS resolution of machine hostnames
won't work. In this case, use the IPs or figure out the correct
`/etc/resolv.conf` somehow)

and `emerge --getbinpkgonly --pretend some-package`, and assuming it's worked
out, you'll see it'll detect your binpkgs. Ideally you'd want to make sure that
your phone is also syncing its ebuild repos from your binpkg host, too, so it
won't try and (gasp!) _build_ packages out of sync with your binary package
host.

### Don't use qemu-system-{ARCH}/TCG, worst mistake of my life

Alas, I am not this lucky. I don't even have _good_[^4] aarch64 hardware to play
with, other than the OnePlus 6 being experimented on. I don't wanna boot
postmarketOS on it, 'cause it'd stop being a useful phone to me at that point --
also I'd have namespaces and notably not be on Android, defeating the purpose of
this guide.

For reasons I don't really want to recall, `crossdev` did not give me a workable
aarch64-unknown-linux-gnu environment. It'd fail to compile GCC for reasons I do
not comprehend, and that's a problem because a package I wanted has it as a dep.

I do not understand cross-compilation. How about emulated native compilation?
Surely that's more reliable, though slower, right?

Well, in an act of desperation (and failure to realize that the `plugins` USE
flag on Qemu was entirely unnecessary, allowing me to build it with
`static-user`), I used `qemu-system-aarch64` and tried installing binutils.[^5]

**1 hour, 10 minutes.**

That's how long it took.

I need you to understand something. The Snapdragon 845 as used in the OnePlus 6
may not be fast by desktop standards, but it pulls off a binutils in 6 minutes.
The Ryzen 7 7730U in the machine I'm typing on does a binutils in 1¼ minutes.
It's not _that_ fast, but it's not this _geologically_ slow, either.

### qemu-user isn't that great either

Alright, then, `qemu-aarch64` time, after I figured out I did not need `plugins`
in its USE flags. Just unpack a similar arm64 stage3,
`cp /usr/bin/qemu-aarch64 ./usr/bin`, `systemd-nspawn`, right? Mostly. Per
[the Gentoo wiki](https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Compiling_with_QEMU_user_chroot#Chrooting),
your FEATURES have to look like

```
FEATURES="-pid-sandbox -network-sandbox buildpkg"
```

I don't have anything dramatic to say here, 1 SBU was 20 minutes, which isn't
great, either, but it's 5 times faster than qemu-system/TCG.

I then made an entry in the phone's `binrepos.conf` with `sync-uri`
`ssh://user@host:/mnt/qemu-aarch64/var/cache/binpkgs`, and it Just Worked.

# Conclusion?

I don't know. I'm thinking of getting graphical stuff 
[(especially hardware-accelerated)](https://github.com/cheadrian/termux-chroot-proot-wine-box86_64/) 
up and running at some point, hence the talk of 'graphical stuff'
in the bonus section.

For funsies, here's the VIDEO_CARDS I'm using. A lot of those aren't 
necessary, but it doesn't hurt to have *all* of them at once.


```
VIDEO_CARDS="virgl zink qxl panfrost freedreno exynos"
```

# Footnotes

[^1]: 
    Yeah, Android uses mksh. In any `adb shell`, try 
    `strings /bin/sh | grep VERSION`
    and you'll probably see something along the lines of
    `KSH_VERSION=@(#)MIRBSD KSH R59 2020/10/31 Android`

[^2]:
    Fun fact: that's `runit` you're using! That's the second use of
    runit-as-service-manager I've seen, the first being GitLab.

[^3]:
    I don't want to waste vertical space quoting the same license repeatedly, so
    here's a single copy of the license you can go and include.
```
# Copyright (c) 2024 multiplealiases
#
# Permission is hereby granted, free of charge, to any person obtaining
# a copy of this software and associated documentation files (the
# “Software”), to deal in the Software without restriction, including
# without limitation the rights to use, copy, modify, merge, publish,
# distribute, sublicense, and/or sell copies of the Software, and to
# permit persons to whom the Software is furnished to do so, subject to
# the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND,
# EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
# NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
# DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
# OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR
# THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```

[^4]: Raspberry Pis (Raspberries Pi?) don't count.

[^5]:
    Timing it to find a nonstandard variant of the
    [Standard Build Unit](https://www.linuxfromscratch.org/lfs/view/11.1/chapter04/aboutsbus.html)
    where you time `emerge binutils` instead of 
    `./configure && make && make install` 
    and it's as multicore as your hardware allows.
