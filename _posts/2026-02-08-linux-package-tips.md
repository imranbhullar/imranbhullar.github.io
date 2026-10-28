---
layout: post
title: "Linux Package Tips: Which Package Owns This File, Holds, and Rollbacks"
date: Feb-08-2026
categories: [Linux, SysAdmin]
tags: [linux, package-management, apt, dnf, dpkg, rpm, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

Installing and removing packages is the easy part. The day-to-day questions are harder: which package installed this binary, how do I stop an upgrade, and how do I roll back a bad update. This post covers those across the Debian and RHEL families.

## Which Package Owns This File?

```bash
# Debian family
dpkg -S /usr/bin/curl
apt-file search bin/curl

# RHEL family
rpm -qf /usr/bin/curl
dnf provides '*/bin/curl'
```

*Tip:* `dpkg -S` only knows about installed packages. `apt-file` searches the archive, so it also finds packages you have not installed yet. Run `sudo apt update && sudo apt install apt-file && sudo apt-file update` once.

## What Files Does a Package Install?

```bash
# Debian family
dpkg -L nginx

# RHEL family
rpm -ql nginx
```

## Hold a Package at Its Current Version

```bash
# Debian family
sudo apt-mark hold nginx
apt-mark showhold
sudo apt-mark unhold nginx

# RHEL family
sudo dnf versionlock add nginx
sudo dnf versionlock list
sudo dnf versionlock delete nginx
```

*Tip:* On RHEL, `versionlock` comes from the `dnf-plugins-core` package. On Debian, `apt-mark hold` is built in.

## Roll Back a Bad Update

```bash
# RHEL family: undo the last transaction
sudo dnf history
sudo dnf history undo last

# Debian family: install a specific older version
apt-cache policy nginx
sudo apt install nginx=1.18.0-6ubuntu14
```

*Tip:* Debian does not keep a transaction history like `dnf`. Pin the version explicitly, and keep the `.deb` files in `/var/cache/apt/archives` if you need to reinstall an older build.

## See What Would Change Before You Change It

```bash
# Debian family
apt list --upgradable
sudo apt -s upgrade

# RHEL family
dnf check-update
sudo dnf upgrade --assumeno
```

## Clean Up Safely

```bash
# Debian family
sudo apt autoremove --purge
sudo apt clean

# RHEL family
sudo dnf autoremove
sudo dnf clean all
```

*Tip:* `autoremove` removes packages that were pulled in as dependencies and are no longer needed. Review the list before confirming — occasionally it wants to remove something you installed manually long ago.
