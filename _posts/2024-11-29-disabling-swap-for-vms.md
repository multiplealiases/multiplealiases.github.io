---
layout: post
title: Disabling swap for Qemu/libvirt/virt-manager virtual machines (on systemd distros)
date: 2024-11-29
permalink: /disabling-swap-for-vms.html
---

I run [zram swap](https://wiki.archlinux.org/title/Zram#Usage_as_swap) on my machines.
Normally this works great,
but stuff like virtual machines absolutely hate having their memory compressed.
They grind to a halt as kswapd works its magic.

It is possible to disable zram swap,
but I'd like to keep it on while not letting my VMs swap.
Contrary to the belief of
[a Stack Overflow answer that annoyed me](https://superuser.com/a/243367),
this *is* possible through the magic of control groups and
[systemd's abstraction over it](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html),
specifically `MemorySwapMax`.

## Direct invocation

If you invoke Qemu directly,
you can use `systemd-run` to set resource limits on Qemu.

```
systemd-run --scope -p MemorySwapMax=0 --user qemu -arg -arg -arg
```

## Overriding machine.slice

virt-manager does not spawn Qemu instances directly,
so the `systemd-run` method is ineffective.
It instead delegates to `libvirt`, but it would appear
at first that there isn't a way to get your hooks in.
However, it does register virtual machines with `systemd-machined`,
which means its VMs are under `machine.slice`.

Run `systemctl edit machine.slice` and add the following override:

```ini
[Slice]
MemorySwapMax=0
```

Shut down all your VMs and run `systemctl restart machine.slice`.
This will disable swap for VMs and containers that register themselves
with `systemd-machined`,
though in practice I've only observed `libvirt` and
systemd's own container manager `machinectl` doing so.
Docker and Podman containers aren't affected by this override.

## Note: How can you tell it's working?

Qemu will use as much RAM as is allocated to it
(with a negligible amount of overhead),
so in your favorite process monitor, watch the RAM usage.
If you allocated 16 GB to your VM
and it hasn't settled at 16 GB used,
instead shrinking down to 1 GB or less,
it's probably swapping.
