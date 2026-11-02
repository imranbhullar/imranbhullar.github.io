---
layout: post
title: "File Integrity and Rootkit Detection on Linux: AIDE, rkhunter, and Lynis"
date: Jan-03-2013
categories: [Linux, Security]
tags: [linux, security, aide, rkhunter, lynis]
image: /assets/img/aide-rkhunter-lynis-security-shield.jpg
---

Firewalls and SSH hardening keep attackers out; these three tools are about noticing if one got in anyway. AIDE flags files that changed when they shouldn't have, rkhunter checks for signatures of known rootkits and backdoors, and Lynis audits general system hardening. None of them replace the others — each catches a different class of problem.

## What Each Tool Actually Does

Before installing anything, it's worth being clear on the division of labour, since the three tools are often lumped together but answer different questions.

```bash
# AIDE: "did any file change since my last known-good baseline?"
# rkhunter: "does anything on this system match a known rootkit signature or behaviour?"
# lynis: "how well-hardened is this system overall, and what's missing?"
```

*Tip:* AIDE is the one that actually catches a compromise in progress, because it doesn't need to recognise the specific malware — just that a file changed. rkhunter and Lynis are both signature/heuristic based and can miss anything novel.

## Installing the Three Tools

All three are packaged for every major distribution and need no special repositories.

```bash
sudo apt install aide rkhunter lynis           # Debian/Ubuntu
sudo dnf install aide rkhunter lynis           # RHEL/CentOS/Fedora
```

*Tip:* Install and baseline AIDE as early as possible after provisioning a machine — ideally right after the OS install and before the box is exposed to any untrusted network traffic. A baseline taken after a compromise just captures the compromise as "normal."

## Initializing an AIDE Database

AIDE's first run builds a database of hashes, permissions, and metadata for every file it's configured to watch — this becomes the baseline every future check compares against.

```bash
sudo aideinit
# or on some distros:
sudo aide --init

# Activate the freshly built database
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

*Tip:* `/etc/aide/aide.conf` controls what's watched and how strictly. Watching `/etc`, `/bin`, `/sbin`, and `/usr` with full checksums, while excluding genuinely volatile paths like `/var/log`, keeps the signal-to-noise ratio usable.

## Running a Check and Reading a Clean Diff

Day-to-day, the only command that matters is `aide --check`, run on a schedule, comparing the live filesystem against the saved baseline.

```bash
sudo aide --check
# AIDE found differences between database and filesystem!!
# Changed files:
# changed: /etc/passwd
# changed: /etc/shadow

sudo aide --check -V
# detail view shows exactly which attribute changed: Mtime, SHA256, Perm, etc.
```

*Tip:* A changed `/etc/passwd` or `/etc/shadow` isn't automatically an incident — adding a user via `useradd` legitimately changes both. The value is that you *know* it changed and can confirm whether you expected it, rather than finding out from someone else.

## Updating the Baseline After Legitimate Changes

Every real system change — a package upgrade, a new user, a config edit — will show up as a diff. Re-baselining after you've verified a change is expected keeps future checks meaningful.

```bash
sudo aide --update
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

*Tip:* Never re-baseline reflexively just to silence a diff. Read what changed first — re-baselining over an actual compromise erases the only evidence AIDE gave you.

## Running rkhunter's Update and Check Cycle

rkhunter ships a database of known rootkit file signatures and suspicious system properties, which needs periodic updates separately from the scan itself.

```bash
sudo rkhunter --update
sudo rkhunter --propupd   # baseline "properties" like file hashes, after a legitimate change

sudo rkhunter --check --sk  # --sk skips the "press enter to continue" prompts
# Checking for rootkits...
#   Rootkit 'Hide_Port'...                                    [ Not found ]
# Warning: /usr/bin/lwp-request was not found...
```

*Tip:* Fresh installs commonly throw warnings about binaries rkhunter expects but the distro doesn't ship, or hash mismatches against its bundled reference list. Investigate each warning once, then run `--propupd` to accept known-good state rather than re-triaging the same false positive every run.

## Running a Lynis Security Audit

Lynis doesn't look for compromise — it scores overall system hardening and lists concrete, actionable suggestions, from SSH config to kernel parameters to filesystem mount options.

```bash
sudo lynis audit system
# [+] System Tools
# [+] SSH Support
# [+] Firewalls
# ...
# Hardening index : 68 [#############       ]
# Suggestions : 24
```

*Tip:* The numeric hardening index is a rough relative gauge, useful for tracking improvement over time on the same system — it's not meaningful to compare index scores across two differently-purposed machines.

## Reading Lynis Warnings and Suggestions

The real output worth reading is the suggestions list at the end of the run, each tagged with a control ID you can look up for detail.

```bash
cat /var/log/lynis-report.dat | grep ^suggestion
# suggestion[]=AUTH-9262|Default umask in /etc/login.defs...|
# suggestion[]=KRNL-5820|Install a PAM module for password strength...|

lynis show details AUTH-9262
```

*Tip:* Treat the suggestion list as a backlog, not a to-do list for one sitting. Fix the ones relevant to this specific machine's role — a suggestion about hardening a web server config is noise on a box that isn't running one.

## Scheduling All Three Unattended

None of these tools are useful run once and forgotten; schedule them as systemd timers (or cron) and make sure the output actually reaches a human.

```bash
# /etc/systemd/system/aide-check.timer - daily AIDE check
[Timer]
OnCalendar=daily
Persistent=true

# rkhunter and lynis ship their own cron entries under /etc/cron.daily/ by default
ls /etc/cron.daily/ | grep -E 'rkhunter|lynis'
```

*Tip:* By default these tools log locally and may only mail `root` on the local machine, which nobody reads. Pipe their output into whatever centralised logging or alerting you already have, or the entire exercise amounts to generating logs nobody looks at.
