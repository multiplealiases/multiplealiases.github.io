---
layout: post
title: (Technically) installing Gentoo on a rooted Android phone
date: 2024-07-27
params:
    license: CC-BY-4.0
permalink: /technicaly-installing-gentoo-on-rooted-android.html
---

# (Technically) installing Gentoo on a rooted Android phone

It's possible, but why would you? It's just a normal Gentoo install, but worse.
This is Diogenes holding up a de-feathered chicken and claiming "behold, a man!".

Not much to say here, just download and untar a stage3 in,
say, a directory under the Termux home directory, then:

```
mount --bind /dev gentoo/dev   # your phone might not have devtmpfs
mount -t devpts devpts gentoo/dev/pts
mount -t proc proc gentoo/proc
mount -t sysfs sys gentoo/sys
mount -t tmpfs shm gentoo/dev/shm
```

chroot, probably using the following command to clear the env
(derived from Chapter 7.4 of LFS 12.1):

```
sudo chroot "gentoo/" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PATH=/usr/bin:/usr/sbin     \
    /bin/bash --login
```

Your kernel will likely be lacking in namespace support; `ls /proc/self/ns`
to see what you have. This annoys `emerge`, which does sandboxing using namespaces.

In this case, use aggressively insecure FEATURES to appease `emerge` in `make.conf`:

```
FEATURES="-sandbox -ipc-sandbox -network-sandbox -pid-sandbox -usersandbox -userfetch"
```
