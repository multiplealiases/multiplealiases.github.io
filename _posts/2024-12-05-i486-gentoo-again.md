---
layout: post
title: "486 Gentoo, again"
date: 2024-12-05
permalink: /i486-gentoo-again.html
---

I've discovered some new things!
One of them stops me from going much further, though.

# 86Box seems bugged

I've been using 86Box 4.2.1 to emulate a 486,
but it crashes a lot and has mystifying bugs:

* Changing ISOs crashes the emulator.

* Networking is entirely borked;
  the network cards show up with all-0xFF MAC addresses,
  as well as `NETDEV WATCHDOG` errors related to timed-out transmissions.

* Running mkfs.ext4 from within the liveCD
  causes it to die with a fatal `glibc` error:

  ```
  Fatal glibc error: malloc.c:4376 (_int_malloc): assertion failed: (unsigned long) (size) >= (unsigned long) (nb)
  Aborted (core dumped)
  ```

Since I don't have a real 486 on hand,
this looks to be the end of the line for me.

# The Gentoo install CD boots, but SIGILLs violently during initramfs

As of this writing, the Gentoo install CD has been updated to
[build the kernel for i486](https://github.com/gentoo/releng/commit/033a719e870cdf5b63573db9b609a43518ea30e1),
but it seems early userspace isn't too happy with 486s.

```
[  319.424232] Run /init as init process
[  319.427565]   with arguments:
[  319.430899]     /init
[  319.434232]     dokeymap
[  319.437565]     cdroot
[  319.444232]   with environment:
[  319.447565]     HOME=/
[  319.450899]     TERM=linux
[  319.454232]     BOOT_IMAGE=/boot/gentoo
[  319.464232] invalid opcode: 0000 [#7] PREEMPT SMP
[  319.464232] CPU: 0 PID: 1 Comm: init Tainted: G      D            6.6.57-gentoo-dist #1
[  319.464232] EIP: exit_to_user_mode_prepare+0x3e/0x1a0
[  319.464232] Code: f7 c3 0e 30 02 00 0f 85 b0 00 00 00 f6 c7 08 75 53 f7 c3 00 00 40 00 75 58 80 e7 40 75 5e 3e 8d 74 26 00 64 8b 0d 00 c9 37 cc <0f> 31 c1 c9 05 31 c1 64 89 0d 00 c9 37 cc 8b 35 30 00 3b cc 85 f6
[  319.464232] EAX: c11a4600 EBX: 00004000 ECX: 00000000 EDX: 00000000
[  319.464232] ESI: c11a4600 EDI: c11abfb4 EBP: c11abf84 ESP: c11abf78
[  319.464232] DS: 007b ES: 007b FS: 00d8 GS: 0000 SS: 0068 EFLAGS: 00000046
[  319.464232] CR0: 80050033 CR2: bfa83d5b CR3: 01017000 CR4: 00000000
[  319.464232] DR0: 00000000 DR1: 00000000 DR2: 00000000 DR3: 00000000
[  319.464232] DR6: 00000000 DR7: 00000400
```

This leads very quickly to a kernel panic.

```
[  319.464232] note: init[1] exited with irqs disabled
[  319.470899] Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
[  319.470899] Kernel Offset: disabled
[  319.470899] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b ]---
```

# You don't need the patch if you're using an x86 toolchain

The CET instructions are in x86_64 libgcc.

```console
$ objdump --disassemble /usr/lib/gcc/x86_64-pc-linux-gnu/14/libgcc.a | grep endbr
   0:   f3 0f 1e fa             endbr64
   0:   f3 0f 1e fa             endbr64
   0:   f3 0f 1e fa             endbr64
   0:   f3 0f 1e fa             endbr64
   0:   f3 0f 1e fa             endbr64
   0:   f3 0f 1e fa             endbr64
(...)
```

These get unconditionally included even with `-m32 -march=i486`, but
there's a fix: use an x86 toolchain.

If you just do not use an x86_64 toolchain, you will not encounter CET,
and you will not need to rebuild your compiler with `--disable-cet` or
use that patch I talked about last article.

The easiest way to find yourself an x86 toolchain is just to use an x86
Gentoo stage3, which leads into my next point:

# You can use the Gentoo tooling to build stuff

This is pretty simple.
You can use an x86 stage3 to build the kernel and initramfs
using an appropriately-configured `installkernel`.
Note that you must build the kernel from source,
as `gentoo-kernel-bin` is built for i686 and higher.

What I did was to make my USE look like this:

```
sys-kernel/installkernel dracut
sys-fs/lvm2 lvm
```


installed the following package list in the chroot:

```
app-arch/lz4
sys-fs/lvm2
sys-fs/squashfs-tools
sys-kernel/dracut
sys-kernel/gentoo-sources
sys-kernel/installkernel
```


and configured Dracut like so:

```
# dmsquash-live allows the initramfs
# to mount squashfs as live systems
dracutmodules+=" dmsquash-live "

# you need iso9660 and squashfs support,
# either built-in or in the initramfs
add_drivers+=" iso9660 squashfs "

# lz4 is quicker to decompress than xz
compress="lz4"
```

Navigate to `/usr/src/linux`, configure the kernel as you wish, and
`make && make modules_install && make install`.
installkernel will do all the magic,
and you should see a kernel and initramfs in `/boot`:

```
$ ls boot/
initramfs-6.6.58-gentoo-r1.img  kernel-6.6.58-gentoo-r1
```

But where do you place these files?

You could do it properly, or
you could take apart and reassemble
a Gentoo live install ISO.

## Taking apart a minimal install CD

`xorriso` might be able to do this properly, but I'm in a hacky mood.
Use `bsdtar` to extract the ISO,
keeping in mind that it'll splat files into your current directory;
`mkdir` a directory and `cd` into it, or use the `-C` option.

```
$ mkdir iso
$ bsdtar -C iso -xvf install-x86-minimal-20241202T164822Z.iso
```

You'll see a layout like this.

```
.
├── boot
│   ├── gentoo
│   ├── gentoo-config
│   ├── gentoo.igz
│   ├── grub
│   │ (...)
│   └── System-gentoo.map
├── boot.catalog
(...)
├── image.squashfs
├── livecd
(...)

16 directories, 973 files
```

The important bits are

* `gentoo` (the kernel),

* `gentoo.igz` (the initramfs, actually xz-compressed),

* `image.squashfs` (the live system).

You can replace the kernel and initramfs with the ones you just made.

If you choose to edit the live system,
you can use `rdsquashfs` and `gensquashfs` (part of squashfs-tools-ng)
to do so.

The `boot/grub/grub.cfg` on this ISO
is configured to look for a specific volume label.
Edit that out unless you intend on recreating it.

```diff
 menuentry 'Boot LiveCD (kernel: gentoo)' --class gnu-linux --class os {
-       search --no-floppy --set=root -l Gentoo-x86-20241202
-       linux /boot/gentoo dokeymap root=live:CDLABEL=Gentoo-x86-20241202 rd.live.dir=/ rd.live.squashimg=image.squashfs cdroot
+       linux /boot/gentoo dokeymap root=live:/dev/sr0 rd.live.dir=/ rd.live.squashimg=image.squashfs cdroot
        initrd /boot/gentoo.igz
 }

 menuentry 'Boot LiveCD (kernel: gentoo) (cached)' --class gnu-linux --class os {
-       search --no-floppy --set=root -l Gentoo-x86-20241202
-       linux /boot/gentoo dokeymap root=live:CDLABEL=Gentoo-x86-20241202 rd.live.dir=/ rd.live.squashimg=image.squashfs cdroot rd.live.ram=1
+       linux /boot/gentoo dokeymap root=live:/dev/sr0 rd.live.dir=/ rd.live.squashimg=image.squashfs cdroot rd.live.ram=1
        initrd /boot/gentoo.igz
 }
```

Once you're done, run `grub-mkrescue -o gentoo.iso iso` to generate your ISO.
For me, this yields an ISO that successfully boots to a shell,
though as mentioned earlier,
the emulator's bugs render this triumph short-lived.
