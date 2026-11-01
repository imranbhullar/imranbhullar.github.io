---
layout: post
title: "chroot Environments: Building and Using a Linux Jail"
date: Sep-29-2012
categories: [Linux, SysAdmin]
tags: [linux, chroot, jail, sandboxing, troubleshooting]
image: /assets/img/chroot-jail-prison.jpg
---

Long before containers existed, `chroot` was how Linux admins built an isolated filesystem view for a process: a jail where `/` means something different from the real root. It is decades older than Docker, far simpler, and still exactly the right tool for a surprising number of jobs, from rescuing a broken system to sandboxing an SFTP user.

## What chroot Actually Isolates, and What It Doesn't

`chroot` changes a process's apparent root directory. Anything the process does with a path is resolved relative to that new root, so it cannot see or touch files outside it. What `chroot` does not do is isolate process IDs, networking, or users; a `chroot`ed process still shares the kernel, the process table, and (unless you drop privileges) root's full set of system capabilities with everything else on the host.

```bash
# The most basic possible example
mkdir /tmp/myjail
sudo chroot /tmp/myjail /bin/sh
```

*Tip:* That command will immediately fail with "No such file or directory," even though `/bin/sh` clearly exists on the real system. That failure is the whole lesson: the jail has no `/bin/sh` of its own, because it has nothing at all yet.

## Building a Minimal chroot by Hand

A working chroot needs the binaries you intend to run, plus every shared library those binaries depend on, copied inside the jail at the matching paths.

```bash
JAIL=/tmp/myjail
mkdir -p $JAIL/{bin,lib,lib64,lib/x86_64-linux-gnu}

# Copy the shell binary itself
cp /bin/bash $JAIL/bin/

# Find and copy every shared library bash depends on
ldd /bin/bash
for lib in $(ldd /bin/bash | grep -o '/lib.*\.[0-9]*'); do
  mkdir -p "$JAIL$(dirname $lib)"
  cp "$lib" "$JAIL$lib"
done

# Try it
sudo chroot $JAIL /bin/bash
```

*Tip:* `ldd` is the command that makes manual chroot-building tractable at all. Run it against every binary you add to the jail, and copy in whatever it lists, or the binary will fail with a cryptic "not found" that has nothing to do with `PATH`.

## Building a Full chroot with debootstrap or dnf --installroot

For anything beyond a single static binary, building the jail by hand is tedious and error-prone. Both major distribution families provide a tool to install an entire minimal userspace into a directory instead.

```bash
# Debian/Ubuntu: build a full Debian userspace in a directory
sudo apt install debootstrap
sudo debootstrap stable /srv/chroot/debian-stable http://deb.debian.org/debian

# RHEL/CentOS/Fedora: install an entire package set into a root directory
sudo dnf --installroot=/srv/chroot/rocky9 --releasever=9 install @core -y
```

*Tip:* `debootstrap` and `dnf --installroot` both resolve full dependency chains for you. They are almost always less work than hand-copying binaries and libraries, except for the smallest, single-purpose jails.

## Entering a chroot Properly: Mounting proc, sys, and dev

A full chroot needs access to the kernel's virtual filesystems to behave like a normal system: without `/proc`, commands like `ps` and `top` simply don't work inside the jail.

```bash
CHROOT=/srv/chroot/debian-stable

sudo mount --bind /proc $CHROOT/proc
sudo mount --bind /sys $CHROOT/sys
sudo mount --bind /dev $CHROOT/dev
sudo mount --bind /dev/pts $CHROOT/dev/pts

# Now enter it
sudo chroot $CHROOT /bin/bash
```

*Tip:* Forgetting to bind-mount `/dev` is the classic cause of "can't open /dev/null" errors from otherwise perfectly functional programs running inside a chroot.

## Using chroot for System Rescue

When a system won't boot because of a broken bootloader, a bad fstab entry, or a corrupted initramfs, booting from a live USB or rescue environment and `chroot`ing into the installed system is the standard recovery technique.

```bash
# From a rescue/live environment, mount the broken system's root
sudo mount /dev/sda2 /mnt

# Mount its boot partition if separate
sudo mount /dev/sda1 /mnt/boot

# Bind-mount the virtual filesystems, exactly as above
for fs in proc sys dev; do sudo mount --bind /$fs /mnt/$fs; done

# chroot in and the broken system's own tools now work
sudo chroot /mnt /bin/bash

# From inside, reinstall a bootloader, regenerate initramfs, fix fstab, etc.
grub-install /dev/sda
update-initramfs -u
```

*Tip:* Once you're inside the chroot, `grub-install`, `update-grub`, and similar commands operate on the broken system exactly as if you'd booted into it normally, which is the entire point of this technique.

## SSH and SFTP Chroot Jails

OpenSSH can confine specific users to a chroot the moment they authenticate, which is the standard way to offer SFTP-only access without giving someone a real shell on the whole filesystem.

```bash
# /etc/ssh/sshd_config
Match User sftpuser
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

```bash
# The ChrootDirectory and everything above it must be owned by root
sudo chown root:root /srv/sftp/sftpuser
sudo chmod 755 /srv/sftp/sftpuser

# A writable subdirectory the user can actually use
sudo mkdir /srv/sftp/sftpuser/upload
sudo chown sftpuser:sftpuser /srv/sftp/sftpuser/upload

sudo systemctl restart sshd
```

*Tip:* OpenSSH refuses to apply `ChrootDirectory` unless the directory itself, and every directory above it up to and including root, is owned by `root` and not group- or world-writable. That ownership requirement is the single most common cause of SFTP chroot setups failing silently.

## Cleaning Up and Leaving a chroot Safely

The bind mounts you set up to make a chroot usable need to come back out in the right order, or you'll be fighting "target is busy" errors.

```bash
# Exit the chroot shell first
exit

# Unmount in reverse order of how they were mounted
sudo umount /mnt/dev/pts
sudo umount /mnt/dev
sudo umount /mnt/sys
sudo umount /mnt/proc
sudo umount /mnt/boot
sudo umount /mnt
```

*Tip:* If `umount` reports "target is busy," check for any process still running inside the chroot with `fuser -v /mnt` before assuming the mount itself is stuck. A leftover shell or daemon inside the jail is the usual culprit.

## chroot vs Containers vs systemd-nspawn

```bash
# chroot:          filesystem isolation only, shares everything else with the host
# systemd-nspawn:  filesystem + PID namespace + network namespace, "chroot with teeth"
# Docker/Podman:   full namespace isolation, cgroups, images, and a management layer on top
```

*Tip:* If you find yourself wanting process isolation, not just a different filesystem view, a chroot is the wrong tool. Reach for `systemd-nspawn` or a container runtime instead; both build on the same namespace primitives but actually isolate the process tree and network stack.

## Bonus: systemd-nspawn as a Modern chroot

`systemd-nspawn` takes the same "run something against a different root" idea as `chroot` and adds the namespace isolation that `chroot` alone never had, without requiring a full container engine.

```bash
# Boot a full chroot-like directory as an isolated container
sudo systemd-nspawn -D /srv/chroot/debian-stable

# Boot it with its own network namespace
sudo systemd-nspawn -D /srv/chroot/debian-stable --network-veth

# Run a single command instead of a full boot
sudo systemd-nspawn -D /srv/chroot/debian-stable /bin/bash
```

*Tip:* Any directory you built for a classic `chroot` with `debootstrap` or `dnf --installroot` works directly with `systemd-nspawn`. It is a drop-in upgrade path once a plain chroot's lack of process isolation starts to matter.
