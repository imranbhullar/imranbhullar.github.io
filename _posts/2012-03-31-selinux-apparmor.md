---
layout: post
title: "SELinux and AppArmor: Mandatory Access Control Without the Pain"
date: Mar-31-2012
categories: [Linux, Security]
tags: [linux, selinux, apparmor, mac, security, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

Mandatory Access Control (MAC) is the layer that stops a compromised service from reading files it has no business reading, even if it runs as root. SELinux and AppArmor are the two implementations you will meet, and both have a reputation for being painful. They are not, once you know the four commands that matter.

## Which One Is Active?

```bash
# SELinux status
getenforce
sestatus

# AppArmor status
sudo aa-status
```

*Tip:* `getenforce` returns `Enforcing`, `Permissive`, or `Disabled`. `Permissive` logs violations but does not block them, which is exactly what you want while debugging.

## SELinux: Labels, Contexts, and Booleans

SELinux labels every file, process, and port with a context. Access is allowed only when the source context is permitted to reach the target context.

```bash
# Show the context of a file
ls -Z /var/www/html/index.html

# Show the context of a process
ps -eZ | grep httpd

# Show the context of a port
sudo semanage port -l | grep http

# Change a file's context
sudo chcon -t httpd_sys_content_t /var/www/html/index.html

# Restore the default context
sudo restorecon -Rv /var/www/html/
```

*Tip:* `chcon` changes are temporary and get overwritten by a relabel. Use `semanage fcontext` plus `restorecon` for permanent changes.

## SELinux Booleans

Booleans toggle pre-defined policy behaviours without writing custom policy.

```bash
# List all booleans
sudo getsebool -a

# Show a specific boolean
sudo getsebool httpd_can_network_connect

# Enable it persistently
sudo setsebool -P httpd_can_network_connect on
```

*Tip:* `httpd_can_network_connect` is the single most common boolean you will need. It is what allows a web app to make outbound connections, and it is off by default.

## Diagnosing SELinux Denials

```bash
# Recent denials
sudo ausearch -m avc -ts recent

# Human-readable explanation
sudo ausearch -m avc -ts recent | audit2why

# Generate a policy module from denialssudo ausearch -m avc -ts recent | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp
```

*Tip:* `audit2why` explains the denial in plain English. `audit2allow` generates a policy module, but only use it after confirming the denial is legitimate. Blindly allowing every denial defeats the purpose of SELinux.

## Switching SELinux Modes

```bash
# Temporary, until reboot
sudo setenforce 0     # permissive
sudo setenforce 1     # enforcing

# Permanent
sudo vi /etc/selinux/config
# SELINUX=enforcing
# SELINUX=permissive
# SELINUX=disabled
```

*Tip:* Never set `SELINUX=disabled` to "fix" a problem. Use `permissive` while debugging, then fix the policy and return to `enforcing`.

## AppArmor: Profiles

AppArmor uses path-based profiles instead of labels, which makes it easier to reason about but less flexible than SELinux.

```bash
# List loaded profiles
sudo aa-status

# Put a profile in complain mode
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# Return to enforce mode
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx

# Reload a profile
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

*Tip:* Complain mode logs violations without blocking them. It is the AppArmor equivalent of SELinux permissive mode and the right first step when a service mysteriously cannot read a file.

## Diagnosing AppArmor Denials

```bash
# Recent denials
sudo journalctl -k | grep -i apparmor
sudo dmesg | grep -i apparmor

# Generate a new profile from a running process
sudo aa-genprof /usr/sbin/myapp

# Update an existing profile interactively
sudo aa-logprof
```

*Tip:* `aa-logprof` reads the denial log and walks you through adding the missing rules. It is the fastest way to fix a profile without hand-editing it.

## Custom SELinux Policy Modules

When a boolean does not exist for your use case, you can write a custom policy module.

```bash
# Generate a policy skeleton from a running process
sudo sepolicy generate --init /usr/local/bin/myapp

# Build the module
cd myapp && sudo ./myapp.sh

# Install the module
sudo semodule -i myapp.pp

# List installed modules
sudo semodule -l

# Remove a module
sudo semodule -r myapp
```

*Tip:* `sepolicy generate` produces a complete policy skeleton, including file contexts and a systemd unit. It is the fastest way to start a custom policy.

## audit2allow: Generating Policy from Denials

```bash
# Generate a policy module from recent denials
sudo ausearch -m avc -ts recent | audit2allow -M mypolicy

# Install it
sudo semodule -i mypolicy.pp

# Explain the denial in plain English
sudo ausearch -m avc -ts recent | audit2why
```

*Tip:* `audit2allow` is a starting point, not a solution. Review the generated policy before installing it, because it allows exactly what was denied, which may be more than you intended.

## AppArmor Profile Authoring

AppArmor profiles are plain text files under `/etc/apparmor.d/`.

```bash
# Generate a profile from a running process
sudo aa-genprof /usr/local/bin/myapp

# Update a profile interactively from the denial log
sudo aa-logprof

# Reload a profile
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp

# Put a profile in complain mode
sudo aa-complain /etc/apparmor.d/usr.local.bin.myapp
```

```text
# /etc/apparmor.d/usr.local.bin.myapp
#include <tunables/global>

/usr/local/bin/myapp {
  #include <abstractions/base>

  /etc/myapp/** r,
  /var/lib/myapp/** rw,
  /var/log/myapp/** w,

  deny /etc/shadow r,
}
```

*Tip:* `aa-logprof` is the fastest way to build a profile. Run the application in complain mode, exercise every code path, then run `aa-logprof` to convert the denials into rules.

## Quick Reference

```bash
# Which MAC is active?
getenforce
sudo aa-status

# SELinux: why was it denied?
sudo ausearch -m avc -ts recent | audit2why

# SELinux: allow a boolean
sudo setsebool -P httpd_can_network_connect on

# AppArmor: complain mode
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
```

*Tip:* Work in order: confirm the mode, find the denial, understand it, then fix the policy. Most "SELinux is blocking me" incidents are a missing boolean or a file with the wrong context.
