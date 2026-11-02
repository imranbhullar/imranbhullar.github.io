---
layout: post
title: "Scheduling Jobs on Linux: cron, at, anacron, and systemd Timers"
date: Oct-11-2012
categories: [Linux, SysAdmin]
tags: [linux, cron, systemd, scheduling, automation]
image: /assets/img/cron-scheduling-wall-clock.jpg
---

Every Linux box ends up running something on a schedule: log rotation, backups, certificate renewals, cleanup scripts. `cron` has handled this since the 1970s, `at` covers the one-off case cron was never meant for, and `systemd` timers now offer a more observable alternative to both. Knowing when to reach for which saves you from either overengineering a five-minute job or losing a backup because a laptop was asleep at 2 a.m.

## Crontab Syntax, Field by Field

Five fields control when a cron job runs: minute, hour, day of month, month, and day of week, followed by the command to execute.

```bash
# minute hour dom month dow command
# * * * * *  command
#
# Run at 3:30am every day
30 3 * * * /usr/local/bin/backup.sh

# Run every 15 minutes
*/15 * * * * /usr/local/bin/healthcheck.sh

# Run at 9am on weekdays only
0 9 * * 1-5 /usr/local/bin/send-report.sh

# Run on the 1st of every month
0 0 1 * * /usr/local/bin/monthly-invoice.sh
```

*Tip:* `crontab -l` shows the current user's crontab; `crontab -e` edits it safely through a temp file, which is why you should never hand-edit `/var/spool/cron/crontabs/*` directly.

## Special Strings and Environment

cron supports a handful of shorthand strings, and every job runs with a much smaller environment than an interactive shell gets, which is the single most common cause of "it works when I run it by hand."

```bash
# Shorthand equivalents
@reboot /usr/local/bin/on-boot.sh   # once, at startup
@daily  /usr/local/bin/daily.sh     # same as 0 0 * * *
@hourly /usr/local/bin/hourly.sh    # same as 0 * * * *

# cron's PATH is minimal; set it explicitly at the top of the crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com
```

*Tip:* Always use absolute paths inside cron scripts, or source the environment explicitly at the top of the script. A script that calls `python3` or `aws` and works at the prompt will fail silently under cron if PATH isn't set.

## System Crontabs and /etc/cron.d

Besides per-user crontabs, the system has its own schedule, split across `/etc/crontab` and `/etc/cron.d/`, both of which carry an extra "user" field that per-user crontabs don't have.

```bash
cat /etc/crontab
# m h dom mon dow user  command
# 25 6  * * *   root  test -x /usr/sbin/anacron || run-parts --report /etc/cron.daily

ls /etc/cron.d/
# sysstat  certbot  php8.1-fpm

# Packages drop their own schedules here instead of editing /etc/crontab
cat /etc/cron.d/certbot
# 0 */12 * * * root certbot -q renew
```

*Tip:* `/etc/cron.d/` is the right place for a package or deployment script to install a schedule, since it avoids merge conflicts with a shared `/etc/crontab` that other tools might also touch.

## The cron.{hourly,daily,weekly,monthly} Directories

Rather than one-off crontab entries, many distros run entire directories of scripts via `run-parts`, driven by the single `/etc/crontab` entry shown above.

```bash
ls /etc/cron.daily/
# apt-compat  logrotate  man-db  dpkg

# Add your own: just drop an executable script in, no crontab edit needed
sudo tee /etc/cron.daily/cleanup-tmp <<'EOF'
#!/bin/sh
find /tmp -type f -mtime +7 -delete
EOF
sudo chmod +x /etc/cron.daily/cleanup-tmp

# Test what run-parts would execute without waiting for the schedule
run-parts --test /etc/cron.daily/
```

*Tip:* Scripts dropped into these directories must be executable and must not have a file extension on most distros' `run-parts` configuration (Debian/Ubuntu skip anything with a dot in the name), which is a common reason a new script silently never runs.

## One-Off Jobs with at

`cron` is for recurring schedules; `at` is for "run this once, later," without needing a crontab entry at all.

```bash
sudo apt install at
sudo systemctl enable --now atd

# Run a command in two hours
echo "/usr/local/bin/send-reminder.sh" | at now + 2 hours

# Run at a specific time
at 23:00 <<'EOF'
/usr/local/bin/nightly-export.sh
EOF

# List pending jobs, and remove one by its job number
atq
atrm 3
```

*Tip:* `at` jobs inherit the environment and working directory present when you submitted them, which makes them more forgiving than cron for a quick one-off, but also means they can behave differently if run from a different shell session.

## anacron for Machines That Aren't Always On

cron assumes the system is always running at the scheduled time; a laptop suspended overnight simply misses the job. `anacron` tracks when each job last ran and catches up after the fact.

```bash
cat /etc/anacrontab
# period  delay  job-identifier  command
# 1       5       cron.daily      run-parts --report /etc/cron.daily
# 7       10      cron.weekly     run-parts --report /etc/cron.weekly
# @monthly 15     cron.monthly    run-parts --report /etc/cron.monthly

# Timestamps anacron uses to decide what's overdue
ls /var/spool/anacron/
# cron.daily  cron.weekly  cron.monthly
```

*Tip:* On most desktop and laptop distros, `/etc/cron.daily` etc. are actually driven by anacron, not cron directly, specifically so missed jobs run shortly after the next boot instead of being skipped entirely.

## systemd Timers: The Modern Alternative

A systemd timer pairs with a matching `.service` unit, giving you `systemctl status`, dependency ordering, and resource limits that bare cron entries don't have.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup.service nightly

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
```

*Tip:* `Persistent=true` is the systemd-timer equivalent of anacron: if the machine was off when the timer should have fired, the job runs once shortly after the next boot instead of being skipped.

## OnCalendar vs OnBootSec

Timers support two different kinds of scheduling: calendar-based, similar to cron, and monotonic, relative to an event like boot.

```ini
# Calendar-style: every day at 03:30
OnCalendar=*-*-* 03:30:00

# Every Monday at 09:00
OnCalendar=Mon *-*-* 09:00:00

# Monotonic: 15 minutes after boot, then every hour after that
OnBootSec=15min
OnUnitActiveSec=1h
```

```bash
# Check what a calendar expression actually means, before relying on it
systemd-analyze calendar "Mon *-*-* 09:00:00"
```

*Tip:* `systemd-analyze calendar` is worth running on every `OnCalendar=` expression you write; the syntax is more flexible than cron's but easy to get subtly wrong, and this command shows you the next few actual run times.

## Inspecting and Debugging Timers

Unlike cron, which logs tersely to syslog if at all, a systemd timer's execution history lives in the journal alongside its service.

```bash
# See every timer, system and user, with next/last run times
systemctl list-timers --all

# Check whether the timer itself is active
systemctl status backup.timer

# Check what happened the last time the service ran
systemctl status backup.service
journalctl -u backup.service --since "1 day ago"

# Trigger it immediately, without waiting for the schedule
sudo systemctl start backup.service
```

*Tip:* `systemctl list-timers` sorts by next run time by default, which makes it the fastest way to answer "what's about to fire on this box" — a question that takes much longer to answer by grepping crontabs.

## Choosing Between Them

Each tool fits a different shape of job, and mixing them on the same box is normal.

```bash
# Rule of thumb:
# - cron: simple, well-understood recurring jobs, especially on older or minimal systems
# - at: genuine one-off, "run this later today" tasks
# - anacron: cron-style jobs on machines that aren't always powered on
# - systemd timers: anything you want to monitor, restart, sandbox, or chain with other units
```

*Tip:* If a scheduled job's failure would actually page someone, prefer a systemd timer over cron — the journal integration and `OnFailure=` unit directive give you a real failure signal, whereas a failed cron job is easy to miss unless `MAILTO` is configured and someone is actually reading that mailbox.
