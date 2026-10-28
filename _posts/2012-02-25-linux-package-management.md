---
layout: post
title: "Linux Package Management: apt, dnf, yum, zypper, and pacman"
date: Feb-25-2012
categories: [Linux, SysAdmin]
tags: [linux, package-management, apt, dnf, yum, zypper, pacman]
image: /assets/img/Linux_unix_bsd.png
---

Every distribution has its own package manager, but they all answer the same four questions: what is installed, what is available, how do I install or remove something, and how do I keep the system patched. This post maps the commands across the major families so you can move between Debian, RHEL, SUSE, and Arch without re-learning the basics.

## The Two Families

* **Debian family** (Debian, Ubuntu, Mint): `apt` / `dpkg`, packages are `.deb`.
* **RHEL family** (RHEL, CentOS, Rocky, Alma, Fedora): `dnf` (or legacy `yum`), packages are `.rpm`.
* **SUSE family** (SLES, openSUSE): `zypper`, packages are `.rpm`.
* **Arch family** (Arch, Manjaro): `pacman`, packages are `.pkg.tar.zst`.

## Updating the Package Index

Before installing anything, refresh the metadata so you are not installing stale versions.

```bash
# Debian / Ubuntu
sudo apt update

# RHEL / Fedora
sudo dnf check-update

# SUSE
sudo zypper refresh

# Arch
sudo pacman -Sy
```

*Tip:* `apt update` refreshes metadata only. `apt upgrade` installs the updates. Running `upgrade` without `update` first is a common source of "package not found" errors.

## Installing and Removing Packages

```bash
# Debian / Ubuntu
sudo apt install nginx
sudo apt remove nginx
sudo apt purge nginx          # remove config files too
sudo apt autoremove           # clean up orphaned dependencies

# RHEL / Fedora
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf autoremove

# SUSE
sudo zypper install nginx
sudo zypper remove nginx

# Arch
sudo pacman -S nginx
sudo pacman -R nginx
sudo pacman -Rns nginx        # remove deps and config
```

*Tip:* `purge` on Debian and `-Rns` on Arch remove configuration files as well. Use them when you want a truly clean reinstall.

## Searching and Inspecting Packages

```bash
# Search the repositories
apt search nginx
dnf search nginx
zypper search nginx
pacman -Ss nginx

# Show package details
apt show nginx
dnf info nginx
zypper info nginx
pacman -Si nginx

# List installed packages
apt list --installed
dnf list installed
zypper se --installed-only
pacman -Q

# Which package owns a file?
dpkg -S /usr/sbin/nginx
rpm -qf /usr/sbin/nginx
pacman -Qo /usr/sbin/nginx
```

*Tip:* "Which package owns this file?" is the single most useful query when you inherit a system and need to know where a binary came from.

## Upgrading the System

```bash
# Debian / Ubuntu
sudo apt update && sudo apt upgrade
sudo apt full-upgrade       # allows removing packages if needed

# RHEL / Fedora
sudo dnf upgrade

# SUSE
sudo zypper update
sudo zypper dup             # distribution upgrade

# Arch
sudo pacman -Syu
```

*Tip:* On Arch, never do a partial upgrade (`pacman -Sy` followed by installing a single package). Always `-Syu` so the whole system stays consistent.

## Holding and Pinning Versions

Sometimes you need to freeze a package at a specific version.

```bash
# Debian / Ubuntu
sudo apt-mark hold nginx
sudo apt-mark unhold nginx
apt-mark showhold

# RHEL / Fedora
sudo dnf versionlock add nginx
sudo dnf versionlock delete nginx

# SUSE
sudo zypper addlock nginx
sudo zypper removelock nginx

# Arch
# Add to /etc/pacman.conf:
# IgnorePkg = nginx
```

*Tip:* `apt-mark hold` is the fastest way to stop an unattended-upgrades run from breaking a pinned service.

## Working with Local Package Files

```bash
# Install a local .deb
sudo dpkg -i package.deb
sudo apt install -f          # fix missing dependencies

# Install a local .rpm
sudo dnf install ./package.rpm
sudo rpm -ivh package.rpm

# Install a local Arch package
sudo pacman -U package.pkg.tar.zst
```

*Tip:* `dpkg -i` does not resolve dependencies. Follow it with `apt install -f` to pull in whatever is missing.

## Repository Configuration

```bash
# Debian / Ubuntu sources
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# RHEL / Fedora repos
dnf repolist
ls /etc/yum.repos.d/

# SUSE repos
zypper repos

# Arch repos
cat /etc/pacman.conf
```

*Tip:* On modern Debian and Ubuntu, prefer files under `/etc/apt/sources.list.d/` over editing `/etc/apt/sources.list` directly. It makes third-party repos easy to add and remove.

## Quick Reference

| Task | Debian | RHEL | SUSE | Arch |
| :--- | :--- | :--- | :--- | :--- |
| Refresh index | `apt update` | `dnf check-update` | `zypper refresh` | `pacman -Sy` |
| Install | `apt install` | `dnf install` | `zypper install` | `pacman -S` |
| Remove | `apt remove` | `dnf remove` | `zypper remove` | `pacman -R` |
| Upgrade all | `apt upgrade` | `dnf upgrade` | `zypper update` | `pacman -Syu` |
| Search | `apt search` | `dnf search` | `zypper search` | `pacman -Ss` |
| File owner | `dpkg -S` | `rpm -qf` | `rpm -qf` | `pacman -Qo` |

*Tip:* Learn the four verbs — refresh, install, remove, upgrade — and you can operate any distribution. The flags differ, the mental model does not.
