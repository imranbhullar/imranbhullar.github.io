---
layout: post
title: "Linux Scheduling Tips: cron Gotchas and systemd Timers"
date: Feb-15-2026
categories: [Linux, SysAdmin]
tags: [linux, cron, systemd, timers, scheduling, automation]
image: /assets/img/Linux_unix_bsd.png
---

Scheduled jobs are simple until they silently fail. This post covers the cron gotchas that bite everyone, and when to reach for a systemd timer instead.

## cron Basics

```bash
# Edit your crontab
crontab -e

# List it
crontab -l

# Edit another user's crontab
sudo crontab -u www-data -e
```

The five fields are minute, hour, day of month, month, day of week:

```text
# min hour dom mon dow  command
  0   3   *   *   *     /usr/local/bin/backup.sh
```

## The Gotchas That Actually Bite

**1. cron has a minimal PATH.** Your interactive shell finds `/usr/local/bin/foo`; cron does not.

```text
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 3 * * * /usr/local/bin/backup.sh
```

**2. `%` is special in crontab.** It means newline. Escape it as `\%`.

```text
0 3 * * * date +\%Y-\%m-\%d >> /var/log/date.log
```

**3. cron does not source your shell profile.** Set environment variables explicitly in the crontab, or source the profile inside the script.

**4. Output goes to mail, which often goes nowhere.** Redirect it.

```text
0 3 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

**5. `@reboot` runs once at boot, not on a schedule.**

```text
@reboot /usr/local/bin/warm-cache.sh
```

## Where cron Files Live

```bash
# Per-user crontabs
/var/spool/cron/crontabs/

# System-wide
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
```

*Tip:* Files in `/etc/cron.d/` need a user field between the schedule and the command. User crontabs do not.

## systemd Timers: The Modern Alternative

Timers give you logging through journald, dependency ordering, and `Persistent=true` to catch up on missed runs.

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
Description=Run backup nightly

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
journalctl -u backup.service
```

*Tip:* `systemctl list-timers` shows the next and last run for every timer. It is the fastest way to confirm a schedule is actually active.

## cron or Timer?

* Use **cron** for quick one-liners and simple recurring jobs.
* Use a **timer** when you need logging, dependencies, catch-up on missed runs, or the job is part of a service.

*Tip:* `OnCalendar` is expressive: `Mon..Fri 09:00`, `hourly`, `daily`, `*-*-* 03:00:00`. Test it with `systemd-analyze calendar 'Mon..Fri 09:00'` before enabling.
