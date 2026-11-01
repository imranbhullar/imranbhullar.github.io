---
layout: post
title: "Linux Disk Quotas: Limiting Usage with quota and edquota"
date: Sep-15-2012
categories: [Linux, SysAdmin]
tags: [linux, quota, disk-quotas, storage, filesystems]
image: /assets/img/linux-disk-quotas-hdd.jpg
---

One runaway log file or a user's forgotten video collection can fill a shared filesystem just as effectively as an attacker, and the fix after the fact is always more painful than the limit that would have prevented it. Disk quotas let you cap how much space and how many files a user or group can consume, before `df` ever hits 100%.

## Why Quotas Still Matter on Shared Systems

Any filesystem shared by multiple users or applications, a home server, a shared `/home` on a multi-user box, a `/tmp` used by several services, benefits from a hard ceiling that no single consumer can cross. Quotas operate per-filesystem, so the limits you set on `/home` have no effect on `/var`.

```bash
# Install the quota tools
sudo apt install quota     # Debian/Ubuntu
sudo dnf install quota     # RHEL/CentOS/Fedora

# Confirm the kernel supports quotas (it does on any modern kernel)
grep -i quota /boot/config-$(uname -r)
```

*Tip:* Quotas are enforced per filesystem, not per directory, unless you're using XFS project quotas (covered below). A quota on `/home` does nothing to limit usage under `/var/log`.

## Enabling Quota Support on a Filesystem

Classic `ext4` quotas require the filesystem to be mounted with quota options, set either in `/etc/fstab` or, more efficiently, as a filesystem feature.

```bash
# Add quota options to the relevant line in /etc/fstab
# /dev/sdb1  /home  ext4  defaults,usrquota,grpquota  0  2

# Remount without rebooting
sudo mount -o remount /home

# Confirm the options took effect
mount | grep /home
```

*Tip:* On modern `ext4`, you can enable quota tracking as a filesystem feature instead of a mount option with `tune2fs -O quota /dev/sdb1`, which avoids needing a separate accounting pass on every mount.

## Initializing Quota Accounting Files

Before any limit can be enforced, the system needs to scan the filesystem once and build its accounting database.

```bash
# Scan the filesystem and create aquota.user / aquota.group
sudo quotacheck -cug /home

# Turn quota enforcement on
sudo quotaon /home

# Confirm it's active
sudo quotaon -p /home
```

*Tip:* Run `quotacheck` with the filesystem unmounted or in read-only mode if you can afford the downtime; otherwise the numbers it calculates can be slightly stale for files that changed mid-scan. For a one-time setup on a quiet filesystem, this rarely matters.

## Setting User and Group Limits with edquota

`edquota` opens an editor with a user's current limits, which you can change directly, no separate configuration file to hunt for.

```bash
# Edit limits interactively for a user
sudo edquota -u alice
```

```text
Disk quotas for user alice (uid 1001):
  Filesystem   blocks   soft   hard   inodes   soft   hard
  /dev/sdb1    102400   512000 600000 4200     10000  12000
```

```bash
# Set the grace period, how long a user can exceed the soft limit before it's enforced as hard
sudo edquota -t

# Check current usage and limits for a user
sudo quota -u alice
```

*Tip:* The soft limit is a warning threshold a user can exceed temporarily, for the grace period. The hard limit is an absolute wall; once hit, writes fail with "disk quota exceeded" no matter how long it's been.

## Checking Usage Across All Users

`repquota` gives you the view `edquota` can't: everyone's usage and limits on a filesystem in a single table.

```bash
# Report for every user on a filesystem
sudo repquota /home

# Report in human-readable sizes
sudo repquota -s /home

# Include users with no usage at all
sudo repquota -a
```

*Tip:* `repquota` is the command to run first when a filesystem is unexpectedly full. It immediately tells you which user, not which directory, is responsible, which is often a faster path to the answer than a `du` sweep.

## Copying Limits Between Users

Setting identical limits for every new user by hand doesn't scale. `edquota -p` copies one user's limits onto another, or onto every user at once.

```bash
# Copy alice's limits to bob
sudo edquota -p alice -u bob

# Copy alice's limits to every user with a UID above 1000
for uid in $(awk -F: '$3>=1000{print $1}' /etc/passwd); do
  sudo edquota -p alice -u "$uid"
done
```

*Tip:* Set one user's limits exactly the way you want them first, then use them as the template. It is far less error-prone than retyping the same soft/hard values for every account.

## XFS Quotas Without quotacheck

XFS handles quotas differently: enforcement is a mount option and accounting happens as part of normal filesystem operation, with its own `xfs_quota` tool instead of `edquota`.

```bash
# Mount option for XFS quotas (in /etc/fstab)
# /dev/sdb1  /data  xfs  defaults,usrquota,grpquota  0  2

# Set a limit using xfs_quota
sudo xfs_quota -x -c 'limit bsoft=500m bhard=600m alice' /data

# Report usage
sudo xfs_quota -x -c 'report -h' /data
```

*Tip:* XFS never needs `quotacheck`; accounting is always on once the mount options are set, which avoids the stale-accounting risk that `quotacheck` on a live `ext4` filesystem carries.

## XFS Project Quotas: Limiting a Directory, Not a User

Project quotas are XFS's answer to "I need a limit on this directory tree regardless of who owns the files in it," which neither `ext4` nor standard user/group quotas can do.

```bash
# Mount with project quota support
# /dev/sdb1  /data  xfs  defaults,pquota  0  2

# Set up a project ID for a directory
sudo xfs_quota -x -c 'project -s -p /data/teamshare 42' /data

# Apply a limit to that project
sudo xfs_quota -x -c 'limit -p bhard=50g 42' /data

# Report by project
sudo xfs_quota -x -c 'report -p' /data
```

*Tip:* Project quotas are the right tool for a shared team directory where you want to cap the directory's total size, not any individual user's, since several different users may be writing into it.

## Enforcing and Reporting Automatically

A quota that silently blocks writes without warning anyone is a support ticket waiting to happen. `warnquota` emails users approaching their limit, and it's meant to run on a schedule.

```bash
# Send warning emails to users over their soft limit, configured in /etc/warnquota.conf
sudo warnquota

# A systemd timer or cron entry to run it daily
# 0 6 * * * /usr/sbin/warnquota
```

*Tip:* Configure `warnquota.conf` with a real `SUPPORT` email address before relying on it. The default template references a generic admin address that users will have no idea how to interpret.
