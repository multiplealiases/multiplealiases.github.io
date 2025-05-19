---
layout: post
title: "A (Linux host and guest) virtual machine root-on-NFS guide"
params:
    license: CC-BY-4.0
draft: true
permalink: /linux-linux-vm-root-on-nfs.html
excerpt: \"Works\", as long as your guest is Arch or Gentoo.
---

<div class="altcontext" markdown="1">
This post specifically focuses on

* Linux hosts:
	* using the in-kernel NFS server
	* running a recent Linux kernel (5.15+ should be fine)
	* running systemd for an init
* Linux guests:
	* running kernel 5.15+ or so
	* using Dracut for an initramfs generator
	* running systemd for an init
* libvirt (via virt-manager)
	* using the NAT network type

I cannot support every configuration.
</div>


Virtual machine disk images are a hassle.

1. They're just a Chunk of Space on your filesystem
   that you can't really reclaim
   (have fun shrinking your VM disks!).

2. qcow2, the 'standard' choice for virtual disk images on Linux/Qemu guests,
   interacts poorly with the likes of Btrfs and ZFS,
   needing you to remember to flip the `nodatacow` switch
   on the disk images so you don't get total garbage
   for write performance
   (and ungodly write amplification, I've heard)

3. Additional finagling is required to share files between host and guest.
   Pick your poison: `scp`, `sftp`, `rsync`,
   `virtiofs`, `virtio-9p`, Kermit-over-SSH, none of them are pleasant.
   None of them are very good for installing a rootfs onto.
   (I have written about [root-on-virtiofs](gentoo-on-virtiofs.html)
   before, but I concluded it was a mess)

If your guest is Windows, I believe you're out of luck.
All options I'm aware of (iSCSI, theoretically Network Block Devices?)
will need you to have a disk image somewhere in the process;
at no point will your Linux host be able to see just a mounted filesystem.
Fortunately, at least Linux guests
(maybe other Unix-likes, too?)
have a way out.

# NFS

Enter NFS: the Network File System.
It was and still is made for what's known as "diskless" setups.
What those do is use PXE to get a kernel loaded into RAM,
the kernel mounts a root file system off the network,
and it uses that as its 'disk'.

Unfortunately, it has some caveats;
the NFS server built into the kernel
doesn't appear to know about Linux filesystem
[capabilities](https://www.man7.org/linux/man-pages/man7/capabilities.7.html),
which renders Fedora unusable.

<div class="panel-danger" markdown="1">
## A note on security

NFS is a very trusting protocol.
It was designed to run over trusted networks,
across trusted devices,
without authentication nor encryption.
While you can cram Kerberos or SSH tunnels
or TLS into NFS,
none of these setups are easy.
To keep it simple,
this article will do none of those;
this is a local-only configuration.

It is crucial that you restrict the host's exports
to the virtual machine network's subnet.
The Linux NFS server listens on 0.0.0.0;
there is no option to make it listen only on one interface.

This setup will not perform any kind of UID/GID mapping.
UID 0 on the guest will be UID 0 on the host.

If you do not trust the software that will run on your virtual machine,
**do not use root-on-NFS**.
</div>

## Host setup

### Preamble: enable the libvirt NSS modules

It's a pain to finagle with IP addresses just to SSH into your virtual machines.
libvirt provides an alternative to this mess: NSS.

Refer to [the NSS section in the docs](https://libvirt.org/nss.html)
for more info, but here's the short of it:

Install `libvirt-nss` (however you do it)
and add at least one of `libvirt` (hostname by DHCP) and
`libvirt_guest` (hostname by libvirt domain name) to the
`/etc/nsswitch.conf` `hosts:` line.

For performance, you'll want to order the `hosts:` line
so that `resolve` (systemd-resolve) and `dns` are done last.
That is to say, it's preferable to do

<pre-header tag="snippet">/etc/nsswitch.conf</pre-header>
<pre>
hosts:      files myhostname mymachines libvirt_guest libvirt <b>resolve dns</b>
</pre>


instead of

<pre-header tag="snippet">/etc/nsswitch.conf</pre-header>
<pre>
hosts:      <b>resolve dns</b> mymachines libvirt_guest libvirt files myhostname
</pre>

Nothing breaks, necessarily,
but it is *really* annoying to have libvirt hosts
take way too long to resolve.
Don't make my mistake.

### Subdirectories

First, you'll need to have a subdirectory to export over NFS.
Convention dictates that they go under the `/export` directory,
but I'm going to suggest that you then make another subdirectory
under that, named for the virtual machine you're using.
That's to say,
if you're planning on installing
Fedora, Gentoo and Arch, run

```console
# mkdir -p /export/fedora
# mkdir -p /export/gentoo
# mkdir -p /export/arch
```

You don't need to do this,
specifically,
but it helps keep this stuff straight.

### Server

We'll need to configure the server.
This is going to be a bare-minimum config.

<div class="panel-info-half" markdown="1">
If you're on a non-systemd distro,
you'll need to read up on how
your init system configures and starts
the NFS daemon.
</div>

First, `nfs.conf`.

It's easy enough,
just add a reasonable-seeming number of threads
and maybe turn off all NFS versions that aren't the absolute latest.
(xattr support doesn't exist on earlier NFS versions)

<pre-header tag="file">/etc/nfs.conf</pre-header>
```ini
[nfsd]
threads=4
vers4.2=on
vers4.1=off
vers4.0=off
vers3=off
```

And now the exports. If you followed my example, your `exports` should look like this.

<pre-header tag="file">/etc/exports</pre-header>
```
/export/fedora    192.168.122.1/24(insecure,rw,async,no_root_squash,crossmnt)
/export/gentoo    192.168.122.1/24(insecure,rw,async,no_root_squash,crossmnt)
/export/arch      192.168.122.1/24(insecure,rw,async,no_root_squash,crossmnt)
```

If you're not using libvirt NAT (or you've changed the subnet),
change the IP address to the actual subnet in use.

And some explanation of the options.

* `insecure`: allow the use of ports higher than 1024
              (based on an outdated model of "security")

* `rw`: makes the export read-write

* `async`: improves performance
   at the cost of data integrity in the event of a crash
   (since the host-to-guest connection is all on the same machine,
   I don't see much point in having more reliability.)

* `no_root_squash`: NFS practices a very primitive form of UID mapping,
  where `root_squash` can map UID 0 ("root") to
  the NFS `nobody` user. We will not be needing that.

* `crossmnt`: allows the exports to "see" mounts inside the subtree.
  This allows you to `mount --bind` paths you feel
  should be shared between host and guest,
  such as `/var/db/repos` between Gentoo host and Gentoo guest.

And start and enable the server.

```command
# systemctl enable --now rpcbind nfs-server
```

## General guest setup

<div class="panel-info-half" markdown="1">
If you choose an initramfs generator other than Dracut,
you're on your own here.
Refer to its documentation for details, I suppose.
</div>

Make yourself a virtual machine with some amount of RAM and some number of CPUs,
and disable storage for it (you won't be needing it, after all).
You'll still need the installer ISO to set up the system for the first time.

In general,
you need some way to install a rootfs to the NFS mount,
and install and configure Dracut.

<pre-header tag="file">/etc/dracut.conf.d/nfs.conf</pre-header>
```sh
add_dracutmodules+=" nfs "
kernel_cmdline+=" rd.ip=auto root=192.168.122.1:/export/wherever "
```

Then configure Dracut to be the initramfs generator (if possible),
and generate the initramfs.

Configure fstab, too, so you don't end up with a read-only root.

<pre-header tag="snippet">/etc/fstab</pre-header>
```
192.168.1.122:/export/wherever    /    nfs4    defaults,rw    0    0
```

A typical diskless system on 'real steel' would use PXE,
which *is* possible to set up in Qemu,
but is an unnecessary pain,
given it's all virtual anyway.
Plus, all PXE does is load a kernel image and initramfs.

It turns out Qemu can do just that,
in the form of
[direct kernel boot](https://qemu-project.gitlab.io/qemu/system/linuxboot.html)
(virt-manager will have this under "Boot Options");
the kernel and initramfs will probably be in the
NFS export's `/boot`.
Since the kernel command line is in the initramfs,
you shouldn't need to set anything under "Kernel args".

Once booted, you should install and enable all applicable guest tools,
such as the Qemu Guest Agent
(typically called `qemu-guest-agent`)
and SPICE Guest Tools
(typically called `spice-vdagent` and `spice-webdavd`).

<div class="panel-warning" markdown="1">
### In-kernel NFS might not work

*If* your kernel has NFS and DHCP built-in,
it should be possible to set cmdline to

<pre-header tag="cmdline"></pre-header>
```
root=/dev/nfs nfsroot=192.168.122.1:/export/wherever ip=dhcp
```

But this is a big "if", as,
in the interest of keeping kernel images (`vmlinuz`) small,
distributions tend to choose to build drivers
like NFS support as kernel modules instead of built-in.

</div>

## Distribution-specific notes
<h-sub markdown="block">
wherein we descend into **hell**
</h-sub>

### Arch

Surprisingly painless
if you ignore the warnings.
The Arch wiki has instructions for using
[dracut as an initramfs generator in place of mkinitcpio](https://wiki.archlinux.org/title/Dracut#Generate_a_new_initramfs_on_kernel_upgrade).

Here's the
[mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio#Using_net) page,
in case you don't feel like doing Dracut.

### Alpine

A total bust.

You'll find that `setup-alpine` doesn't know anything about NFS,
so we'll need to perform a manual install using `apk --root`.
Get ready for some yak-shaving.

To even get the NFS share mounted from the official install media:

* `setup-interfaces` then `rc-service networking restart`,

* add your favorite nameserver to `/etc/resolv.conf`

* `setup-apkrepos` (enable community for good measure),

* `apk install nfs-utils`

* `rc-service nfs start`

And only now can you mount the NFS share
(for the sake of example, it's at `/mnt`)
and install a base system to it.
To make your life easier,
copy in `/etc/apk/repositories` and `/etc/resolv.conf`, too.

```console
# apk --root /mnt --initdb --arch $(arch) -X https://dl-cdn.alpinelinux.org/alpine/latest-stable/main -U --allow-untrusted add alpine-base
# cp -a /etc/apk/repositories /mnt/etc/apk/repositories
# cp -a /etc/resolv.conf /mnt/etc/resolv.conf
```

Either install and use `arch-chroot`,
or mount `/dev`, `/proc` and `/sys` manually
before `chroot`ing.

<details markdown="1">
<summary>Dracut instructions (that died due to a packaging bug)</summary>
Install `nfs-utils` (again), `dracut`,
`dracut-modules-network` and `kernel-hooks`.
NFS and Dracut should be obvious, but `kernel-hooks` needs
some explanation: it lets you add 'hooks' (shell scripts)
that run every time the kernel and associated modules
are installed, reinstalled, or removed.

I don't believe anyone's written a kernel hook for Dracut,
so here's one I wrote.
Be sure to `chmod +x` this after you copy it into the NFS share.

<pre-header tag="script">/etc/kernel-install.d/dracut</pre-header>
```sh
#!/bin/sh
# Copyright (c) 2025 multiplealiases
# SPDX-License-Identifier: MIT

set -o nounset

# deliberate choice:
# the hook has to keep going
# even if dracut returns nonzero
#set -o errexit

die() {
        # shellcheck disable=SC2059
        1>&2 printf "$@"
        exit 1
}

flavor="$1"
new="$2"
old="$3"

check_mode() {
    # if both $new and $old are non-empty,
    # then it must be asking to replace old with new
    if [ -n "$new" ] && [ -n "$old" ]
    then
        printf 'replace'
    # if $new is empty,
    # then it must be asking to remove old
    elif [ -z "$new" ] && [ -n "$old" ]
    then
        printf 'remove'
    # if $old is empty,
    # the kernel is being reinstalled,
    # or the moral equivalent
    # of @modules-rebuild is happening
    elif [ -n "$new" ] && [ -z "$old" ]
    then
        printf 'reinstall'
    else
        die 'I have no idea what is being asked of me.\n'
    fi
}

mode=$(check_mode)
if [ "$mode" = 'replace' ] || [ "$mode" = 'reinstall' ]
then
    # linux-* packages come with /boot/vmlinuz-$flavor
    dracut --force --noprefix /boot/initramfs-"$flavor".new "$new"
    # mv is atomic on most filesystems.
    mv /boot/initramfs-"$flavor".new /boot/initramfs-"$flavor"
elif [ "$mode" = 'remove' ]
then
    rm /boot/initramfs-"$flavor"
else
    die "check_mode() returned nothing?\n"
fi
```

Set up Dracut as before for NFS.
Make sure to disable `mkinitfs`'s autogeneration of initramfses.

<pre-header tag="file">/etc/mkinitfs/mkinitfs.conf</pre-header>
```sh
#features="ata base cdrom ext4 keymap kms mmc nvme raid scsi usb virtio"
disable_trigger="yes"
```

And it turns out this *doesn't work*.

```
# apk fix linux-lts
(1/1) Reinstalling linux-lts (6.12.28-r0)
Executing busybox-1.37.0-r12.trigger
Executing kernel-hooks-0.2-r1.trigger
kernel-hooks: executing hook dracut (lts, 6.12.28-0, )
dracut[I]: Executing: /usr/bin/dracut --force --noprefix /boot/initramfs-lts.new 6.12.28-0-lts
dracut[E]: Module 'network' depends on module 'network-legacy', which can't be installed
dracut[E]: Module 'nfs' depends on module 'network', which can't be installed
dracut[E]: Module 'nfs' cannot be installed.
```

[When Alpine 3.21 was released](https://wiki.alpinelinux.org/wiki/Release_Notes_for_Alpine_3.21.0),
they removed ISC DHCP,
which provides `dhclient`.
This package is required by the dracut module `network-legacy`,
one possible way to satisfy the deps of `network`,
pulled in by any module that needs to set up networking...
such as `nfs`.

They, of course, removed the module `network-legacy`,
but it turns out the alternative,
`network-manager`,
is unsatisfiable under Alpine.
There is literally no set of packages you could install to make this work.

See, the chain goes `network-manager` -> `dbus` -> `dbus-daemon` -> `systemd`.
Do you see the problem?
Alpine *doesn't use* systemd!

And that's why this section lives in a `<details>` tag.
</details>

And, no, it turns out `mkinitfs` can't save us, either.
For starters, it can't seem to get the network up,
so it kicks you to a shell.
So: babysit it by
configuring the network,
mount the NFS share,
and hit Ctrl+D.

<aside markdown="1">
The option used,
`-e`/`--exclusive`,
*does* exist in BusyBox,
under the name `-x`.
The funniest part is that `-e`
is *default* behavior
and is *exactly* the same as `-x`.

flock(1) is a util-linux invention.
It doesn't need to conform to prior standards because it *is* its own standard.
What happened here?
Why are there 2 names for the same flag?
</aside>

It'll get stuck trying to use `flock`...
with an option that doesn't exist in BusyBox's `flock`.
Install the `flock` package
(split off from `util-linux`, looks like)
and lo and behold,
**it gets stuck**,
seemingly waiting for a lock it can never acquire.

### Gentoo (systemd)

I'm going to assume you're using the
official Gentoo minimal install liveCD,
not because you need it to install Gentoo,
but because I'm going to assume
you're not trying to suffer more than you already are.

#### Start the chronyd service before chrooting

Because you're accessing files on the host, with *its* timestamps,
you may encounter clock desync between the guest and host.
Some packages may get confused by this,
such as packages using Meson for a build system.
Clock sync would usually be handled by Qemu Guest Agent,
but it's not on the liveCD.

Instead, start the NTP daemon
and tell it to abruptly jump
the system clock to the current time.

```console
# rc-service chronyd start
# chronyc makestep
```

#### Sporadic package failures may occur on NFS /var/tmp/portage

You may encounter failures like

```
!!! Error Extracting '/var/cache/binpkgs/sys-apps/miscfiles/miscfiles-1.5-r4-1.gpkg.tar',
[Errno 16] Device or resource busy: '/var/tmp/portage/sys-apps/miscfiles-1.5-r4/temp'
```

I believe this is a symptom of NFS locking,
because it happens randomly,
and you *can* just run `emerge` repeatedly until it succeeds.

<div class="panel-info-half" markdown="1">
Small `/var/tmp/portage`s should not be paired with
Portage parallelism (`-j` higher than 1),
as parallel Portage will store multiple packages
waiting to be merged;
this may cause your tmpfs to run out of space faster than expected.
</div>

You *will* need more RAM allocated to the VM
(I mean, you're doing Gentoo) to do this,
but the safest workaround is to just make `/var/tmp/portage`
a tmpfs inside the VM:


```console
# mount -t tmpfs AAA /var/tmp/portage
```

For good measure, configure it in `fstab` so you don't forget:

<pre-header tag="snippet">/etc/fstab</pre-header>
```fstab
AAA    /var/tmp/portage    tmpfs    defaults    0    0
```

Some packages are more cooperative than others.
It might be possible to build packages that are too large
[outside of the tmpfs](https://wiki.gentoo.org/wiki/Portage_TMPDIR_on_tmpfs).

### Gentoo (OpenRC)


### Fedora

I can't get it to work.

#### Fedora won't install to NFS

It just won't. The installer can do iSCSI, but not NFS.
This can be worked around by opening a terminal,
mounting the NFS export yourself, and running
`dnf --installroot`.

```
# dnf --use-host-config --installroot /mnt/sysimage --releasever=42 install @core @gnome-desktop
```

#### NFS doesn't know about filesystem capabilities (as far as I can tell)

You'll see this error.

```
>>> [RPM] unpacking of archive failed on file /usr/bin/newgidmap: cpio: cap_set_file failed - Operation not supported
>>> Unpack error: shadow-utils-2:4.17.4-1.fc42.x86_64
```

Getting around this requires an
[incredibly dirty hack](https://bugzilla.redhat.com/show_bug.cgi?id=648654#c59)
to stub out `cap_set_file()` and `cap_set_fd()`.
This comes in the form of a library
that you `LD_PRELOAD` into `dnf`.
In case that page gets deleted, the code for it is

<pre-header tag="code">disable-set-filecaps.c</pre-header>
```c
#include <sys/capability.h>

int cap_set_file(const char *path_p, cap_t cap_p) {
    return 0;
}

int cap_set_fd(int fd, cap_t caps) {
    return 0;
}
```

Compile and use as

```console
# dnf install @c-development libcap-devel
# gcc -shared -Wall disable-set-filecaps.c -o /disable-set-filecaps.o
# LD_PRELOAD=/disable-set-filecaps.o dnf --use-host-config --installroot /mnt/sysimage --releasever=42 install @core @gnome-desktop
```

#### SELinux hates your guts

Fine, so we'll just set it to permissive:

<pre-header tag="cmdline"></pre-header>
```
enforcing=0
```

#### Fedora hates your guts

As far as I'm able to tell,
this is the end of the line.
It seems to pivot into the final rootfs,
but a million services die at once after that.
