---
layout: post
title: "systemd: Managing Services, Boot Behaviour, and Custom Apps the Right Way"
date: Jan-21-2012
categories: [Linux, DevOps]
tags: [linux, systemd, services, boot, troubleshooting, automation]
image: /assets/img/Linux_unix_bsd.png
---

Every modern Linux distribution boots with systemd, yet most admins only ever touch `systemctl start` and `systemctl status`. Understanding units, targets, and drop-in overrides turns systemd from a black box into the cleanest way to run your own applications reliably on any server.

## Why systemd Replaced SysVinit and Upstart

The old init systems started shell scripts in sequence, which made dependencies implicit and parallel boot nearly impossible. systemd models everything as a unit with declared dependencies, starts independent services in parallel, tracks processes with cgroups, and restarts anything that crashes. It also gives you a single consistent interface for services, timers, mounts, sockets, and more.

```bash
# Confirm systemd is PID 1
ps -p 1 -o comm=

# Check the systemd version
systemctl --version

# Overall system state
systemctl is-system-running
```

*Tip:* `systemctl is-system-running` returns `running`, `degraded`, `starting`, or `maintenance`. A `degraded` state almost always means at least one unit failed, so follow up with `systemctl --failed`.

## Understanding Units, Targets, and the Unit File Layout

A unit is the basic object systemd manages. Services are `.service` units, timers are `.timer`, mounts are `.mount`, and targets are `.target` units that group other units together, much like old runlevels. Unit files live in three places, and systemd merges them in order of precedence.

```bash
# Vendor-supplied units, never edit these directly
ls /usr/lib/systemd/system/

# Admin overrides, this is where your changes belong
ls /etc/systemd/system/

# Runtime units, cleared on reboot
ls /run/systemd/system/

# Show the full unit file and where it was loaded from
systemctl cat nginx
```

*Tip:* Never edit files under `/usr/lib/systemd/system/`. Package upgrades overwrite them. Use `systemctl edit <unit>` to create a drop-in override under `/etc/systemd/system/<unit>.d/` instead.

## Starting, Stopping, and Checking Service Status

The day-to-day commands are simple, but the difference between `start` and `enable` trips up almost everyone at least once.

```bash
# Start and stop a service right now
sudo systemctl start nginx
sudo systemctl stop nginx

# Restart or reload without dropping connections
sudo systemctl restart nginx
sudo systemctl reload nginx

# Current status, recent logs, and the main PID
systemctl status nginx

# Is it running right now?
systemctl is-active nginx

# Is it enabled at boot?
systemctl is-enabled nginx
```

*Tip:* `reload` only works if the unit defines `ExecReload`. When in doubt, `systemctl reload-or-restart nginx` does the right thing either way.

## Enabling and Disabling Services at Boot

`start` affects the current boot, `enable` affects future boots. They are independent, and mixing them up is the most common source of "it worked until I rebooted" surprises.

```bash
# Enable at boot and start immediately
sudo systemctl enable --now nginx

# Enable at boot without starting now
sudo systemctl enable nginx

# Disable at boot but leave it running
sudo systemctl disable nginx

# Disable at boot and stop it now
sudo systemctl disable --now nginx

# Mask a unit so nothing can start it, not even as a dependency
sudo systemctl mask nginx
sudo systemctl unmask nginx
```

*Tip:* `mask` is stronger than `disable`. A masked unit cannot be started by hand or pulled in as a dependency, which is exactly what you want when a package keeps re-enabling something you do not need.

## Listing, Filtering, and Inspecting Units

On a busy host there are hundreds of units, so filtering is essential.

```bash
# All loaded units
systemctl list-units --type=service

# Every installed unit file, including disabled ones
systemctl list-unit-files --type=service

# Only failed units
systemctl --failed

# Units that depend on a given unit
systemctl list-dependencies nginx

# Which unit provides a given PID
systemctl status 1234
```

*Tip:* `systemctl list-unit-files --state=enabled` is the fastest way to audit exactly what will start on the next boot.

## Reading Logs with journalctl

systemd captures stdout and stderr from every service into the journal, so you rarely need to hunt for log files anymore.

```bash
# Logs for a single unit
journalctl -u nginx

# Follow a unit's logs live
journalctl -u nginx -f

# Only errors and above, since the last boot
journalctl -p err -b

# Logs from the last 30 minutes
journalctl --since "30 min ago"

# Logs for a specific PID
journalctl _PID=1234
```

*Tip:* Add `--no-pager` when piping `journalctl` into `grep` or `less`, otherwise the pager can swallow your output in scripts.

## Writing a Custom Service for a Home-Built App

This is where systemd earns its keep. Any binary or script can become a first-class service with restart-on-failure, logging, and boot integration, without a wrapper script or a cron hack.

```bash
# Create the unit file
sudo vim /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Home-Built Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yml
Restart=on-failure
RestartSec=5
Environment=APP_ENV=production
StandardOutput=journal
StandardError=journal

# Basic hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

```bash
# Reload systemd so it sees the new unit
sudo systemctl daemon-reload

# Enable and start it
sudo systemctl enable --now myapp

# Confirm it is healthy
systemctl status myapp
journalctl -u myapp -f
```

*Tip:* `Type=simple` is correct for anything that stays in the foreground. Use `Type=oneshot` for scripts that run and exit, and `Type=forking` only for legacy daemons that background themselves.

## Running a Script on a Schedule with systemd Timers

Timers replace cron with better logging, dependency handling, and the ability to catch up on missed runs.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Nightly Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run the nightly backup

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# Enable the timer, not the service
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# List active timers and their next run
systemctl list-timers

# Trigger the service manually for testing
sudo systemctl start backup.service
```

*Tip:* `Persistent=true` makes the timer run immediately if the machine was off at the scheduled time, which is exactly what you want for backups on laptops and VMs.

## Overriding Vendor Units Without Editing Them

Drop-in overrides let you change one directive without copying an entire unit file and inheriting its maintenance burden.

```bash
# Open an editor that creates the drop-in for you
sudo systemctl edit nginx
```

```ini
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65535
Environment=NGINX_WORKERS=4
```

```bash
# Apply the change
sudo systemctl daemon-reload
sudo systemctl restart nginx

# Verify the merged result
systemctl cat nginx
```

*Tip:* `systemctl edit --full nginx` copies the whole unit into `/etc/systemd/system/` for larger changes, but prefer drop-ins whenever a single directive will do.

## Troubleshooting Failed Services

When a unit refuses to start, systemd usually tells you exactly why if you know where to look.

```bash
# See what failed and why
systemctl --failed
systemctl status myapp

# Full logs for the failed unit
journalctl -u myapp -b --no-pager

# Check the unit file for syntax errors
systemd-analyze verify /etc/systemd/system/myapp.service

# See what took the longest during boot
systemd-analyze blame
systemd-analyze critical-chain
```

*Tip:* `systemd-analyze verify` catches typos in unit files before you restart anything, and `systemd-analyze critical-chain` shows the exact dependency path that is slowing down boot.

## Bonus: Useful systemctl Shortcuts Worth Memorising

A handful of commands cover most day-to-day work and save a surprising amount of typing.

```bash
# Reboot and power off through systemd
sudo systemctl reboot
sudo systemctl poweroff

# Enter rescue or emergency mode
sudo systemctl rescue
sudo systemctl emergency

# Reload all unit files after editing anything under /etc/systemd/system
sudo systemctl daemon-reload

# Show the default target, the modern equivalent of a runlevel
systemctl get-default
sudo systemctl set-default multi-user.target
```

*Tip:* After any change under `/etc/systemd/system/`, always run `systemctl daemon-reload`. Forgetting it is the single most common reason a new or edited unit appears to be ignored.
