---
layout: post
title: Gentoo on virtiofs
date: 2024-07-27
params:
    license: CC-BY-4.0
permalink: /gentoo-on-virtiofs.html
---

Not very good. Breakage because it can't emulate {U,G}IDs to the satisfaction
of `emerge`, X11, and some other things I'm probably forgetting.
Just about boots Gentoo musl/OpenRC to a tty. X will not run, but starting
KDE/Wayland with `startplasma-wayland` works.

Versions of stuff used:

| **Software** |     **Version**    | **Released (YYYY-MM-DD)** |
|:------------:|:------------------:|:-------------------------:|
|   virtiofsd  |       1.10.0       |         2024-01-19        |
| virt-manager |        4.1.0       |         2022-08-05        |
|     qemu     |        8.2.3       |         2024-04-24        |
|    libvirt   |       10.3.0       |         2024-05-02        |
| Linux kernel | 6.6.30-gentoo-dist |         2024-05-02        |

x86_64-pc-linux-gnu host, x86_64-pc-linux-musl guest (Qemu/KVM).

## Longer answer

### Enable xattrs

I did this in `virt-manager`. To save you some pain: enable xattrs on the
virtiofs filesystem and set the `--xattrmap` option Somewhere™.

I'm gonna cite a mailing list archive post:
https://listman.redhat.com/archives/virtio-fs/2021-June/003529.html

Enabling xattrs *can* be done by adding

```xml
  <binary xattr="on"/>
```

but libvirt's XML schema isn't aware of some options to `virtiofsd`
[(I checked!)](https://libvirt.org/formatdomain.html), so that needs
to go in a wrapper script.

`/usr/local/bin/wrapper-virtiofsd` (and mark executable!):

```bash
#!/usr/bin/env bash
set -o allexport
/usr/libexec/virtiofsd "$@" --xattrmap ':map::user.virtiofs.:'
```

So that makes the actual line:

```xml
  <binary path="/usr/local/bin/wrapper-virtiofsd" xattr="on"/>
```

You'll need `--xattrmap` because the VM will want to set capabilities on
binaries (think `ping` and how it has `CAP_SYS_ADMIN`) that `virtiofsd`
shouldn't itself be able to set.

### Install

I started with an `amd64-musl` stage3 and
used the `default/linux/amd64/23.0/musl` profile.

#### Mounting root

It's pretty much the same as any other install, though you'll
have to change your `mount` invocation to

```
mount -t virtiofs fs /mnt/gentoo
```

your fstab should look like

```
# <fs>  <mountpoint>  <type>    <opts>    <dump>  <pass>
fs      /             virtiofs  defaults  0       1
```

and your cmdline (either by passing it in Direct Kernel Boot or
into the initramfs) should have `root=fs rootfstype=virtiofs` in it.

#### CMake

If building CMake breaks because of the following error:

```
-- Checking if compiler supports C++ filesystem
-- Checking if compiler supports C++ filesystem - no
CMake Error at CMakeLists.txt:93 (message):
  The C++ compiler does not support C++11 (e.g.  std::unique_ptr).
```

mount a tmpfs on `/var/tmp` by running `mount -t tmpfs vartmp -o size=16G /var/tmp`.

For reasons I can't comprehend, I think it might actually be
poking at the filesystem it's building on for this test.
[A similar issue was noted on Stack Overflow](https://stackoverflow.com/questions/55772725/the-c-compiler-does-not-support-c11-e-g-stdunique-ptr-building-openwrt)
and the top answer was "don't use NFS".

#### Sporadic failures

Sometimes emerge just doesn't like virtiofs. Here's an example:

```
!!! failed to properly create symlink:
!!! /usr/lib/libnghttp2.so.14 -> libnghttp2.so.14.28.0
!!! [Errno 1] Operation not permitted: b'/usr/lib/libnghttp2.so.14'
!!! Failed to move file.
!!! /usr/lib/libnghttp2.so.14 -> libnghttp2.so.14.28.0
```

I don't know how or why this happened, but I emerged it again and it worked?!

```
>>> Completed (1 of 2) net-libs/nghttp2-1.61.0::gentoo
```

Sometimes it *consistently* doesn't like virtiofs.

```
Operation Not Permitted: chown('/var/tmp/portage/app-text/asciidoc-10.2.0/temp/python3.12/pkgconfig/python3.pc', 250, 250)
Operation Not Permitted: chown('/var/tmp/portage/app-text/asciidoc-10.2.0/temp/python3.12/pkgconfig/python3-embed.pc', 250, 250)
Operation Not Permitted: chown('/var/tmp/portage/app-text/asciidoc-10.2.0/temp/python3.12/bin/2to3', 250, 250)
```

It steamrolled through this just fine, but `sys-kernel/gentoo-kernel-bin`
wouldn't install at all.

#### The kernel

No. Just no. It does not.

```
>>> Install sys-kernel/gentoo-kernel-bin-6.6.30 into /var/tmp/portage/sys-kernel/gentoo-kernel-bin-6.6.30/image
>>> Completed installing sys-kernel/gentoo-kernel-bin-6.6.30 into /var/tmp/portage/sys-kernel/gentoo-kernel-bin-6.6.30/image

 * Final size of build directory: 1548476 KiB (  1.4 GiB)
 * Final size of installed tree:   529784 KiB (517.3 MiB)

[ERROR] Exception in callback AsynchronousTask._exit_listener_cb(<bound method...7fbd318fdb30>>)
handle: <Handle AsynchronousTask._exit_listener_cb(<bound method...7fbd318fdb30>>)>
Traceback (most recent call last):
  File "/usr/lib/python3.12/site-packages/portage/util/__init__.py", line 1238, in apply_permissions
    portage.data.lchown(filename, uid, gid)
  File "/usr/lib/python3.12/site-packages/portage/__init__.py", line 281, in __call__
    rval = self._func(*wrapped_args, **wrapped_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
PermissionError: [Errno 1] Operation not permitted: b'/var/tmp/portage/sys-kernel/gentoo-kernel-bin-6.6.30/image/usr/src/linux-6.6.30-gentoo-dist/include/dt-bindings/input/linux-event-codes.h'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib/python3.12/asyncio/events.py", line 88, in _run
    self._context.run(self._callback, *self._args)
  File "/usr/lib/python3.12/site-packages/_emerge/AsynchronousTask.py", line 209, in _exit_listener_cb
    listener(self)
  File "/usr/lib/python3.12/site-packages/_emerge/EbuildPhase.py", line 315, in _ebuild_exit
    self._ebuild_exit_unlocked(ebuild_process)
  File "/usr/lib/python3.12/site-packages/_emerge/EbuildPhase.py", line 380, in _ebuild_exit_unlocked
    _post_src_install_uid_fix(settings, out)
  File "/usr/lib/python3.12/site-packages/portage/package/ebuild/doebuild.py", line 2817, in _post_src_install_uid_fix
    apply_secpass_permissions(
  File "/usr/lib/python3.12/site-packages/portage/util/__init__.py", line 1426, in apply_secpass_permissions
    apply_permissions(
  File "/usr/lib/python3.12/site-packages/portage/util/__init__.py", line 1243, in apply_permissions
    raise OperationNotPermitted(func_call)
portage.exception.OperationNotPermitted: chown('b'/var/tmp/portage/sys-kernel/gentoo-kernel-bin-6.6.30/image/usr/src/linux-6.6.30-gentoo-dist/include/dt-bindings/input/linux-event-codes.h'', 0, 0)
Terminated
```

I got around this problem in the dirtiest possible way: I just transplanted
the host's (also Gentoo) kernel into the VM!

```console
# cp -ar /usr/lib/modules/6.6.30-gentoo-dist/ /virtiofs/usr/lib/modules/
# cp -ar /usr/src/linux-6.6.30-gentoo-dist    /virtiofs/usr/src
# cp -a --dereference /virtiofs/usr/src/linux-6.6.30-gentoo-dist/vmlinuz /virtiofs/boot/vmlinuz-6.6.30-gentoo-dist
```

Honestly quite incredible.

You can now generate an initramfs with `dracut`, keeping in mind that `--kver`
needs to be passed explicitly if you're still in the LiveCD environment.

```console
# dracut --kver 6.6.30-gentoo-dist --kernel-cmdline 'root=root rootfstype=virtiofs'
```

#### A graphical environment

SDDM won't run.

```console
# dbus-launch sddm
[18:34:26.993] (II) DAEMON: Initializing...
[18:34:27.001] (II) DAEMON: Starting...
[18:34:27.001] (II) DAEMON: Logind interface found
[18:34:27.003] (II) DAEMON: Adding new display...
[18:34:27.004] (II) DAEMON: Loaded empty theme configuration
[18:34:27.009] (II) DAEMON: Xauthority path: "/run/sddm/xauth_FizuhC"
[18:34:27.009] (II) DAEMON: Using VT 7
[18:34:27.009] (II) DAEMON: Display server starting...
[18:34:27.009] (II) DAEMON: Writing cookie to "/run/sddm/xauth_FizuhC"
[18:34:27.009] (II) DAEMON: Running: /usr/bin/X -nolisten tcp -background none -seat seat0 vt7 -auth /run/sddm/xauth_FizuhC -noreset -displayfd 15
[18:34:27.864] (II) DAEMON: Setting default cursor
[18:34:27.867] (WW) DAEMON: Could not setup default cursor
[18:34:27.867] (II) DAEMON: Running display setup script  "/usr/share/sddm/scripts/Xsetup"
[18:34:27.882] (II) DAEMON: Display server started.
[18:34:27.882] (II) DAEMON: Socket server starting...
[18:34:27.884] (II) DAEMON: Socket server started.
[18:34:27.886] (WW) DAEMON: Failed to change owner of the socket
```

It switches VTs, but it doesn't display anything. I'm unsure why.
Is it just because it's using X? It's possible to start KDE/Wayland with
`startplasma-wayland`; Firefox segfaults if you launch it. Launching the
binary directly works, though, so it's some variable in the wrapper script.
