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

## Cron vs. systemd Timers

Cron still works and is perfectly fine for simple jobs, but systemd timers offer better logging, dependency handling, and catch-up behaviour. The table below summarises the trade-offs.

| Feature | Cron | systemd Timers |
| :--- | :--- | :--- |
| **Logging** | Mailed to the user, or redirected manually | Captured in the journal automatically |
| **Missed runs** | Silently skipped | `Persistent=true` runs on next boot |
| **Dependencies** | None | Full unit dependency graph |
| **Randomised delay** | Not supported | `RandomizedDelaySec=` |
| **Resource limits** | Not supported | `CPUQuota=`, `MemoryMax=`, etc. |
| **Monitoring** | `crontab -l` | `systemctl list-timers` |
| **Granularity** | 1 minute | 1 second |
| **User jobs** | `crontab -e` | `systemctl --user` |

### Equivalent Examples

A cron entry that runs a backup every night at 2 a.m.:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

The same job as a systemd timer:

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

### When to Use Which

* **Use cron** for simple, self-contained jobs on systems where you already have cron configured, or when you need per-user schedules without systemd user sessions.
* **Use systemd timers** when you want logging, dependency ordering, resource limits, catch-up on missed runs, or when the job is part of a larger systemd-managed service.

*Tip:* `systemctl list-timers --all` shows every timer, including inactive ones, along with the last and next run times. It is the fastest way to audit scheduled jobs on a systemd host.

### Testing an `OnCalendar` Expression

`OnCalendar` is expressive but easy to get subtly wrong. Test it before you enable the timer.

```bash
# Show the next few times the expression would fire
systemd-analyze calendar 'Mon..Fri 09:00'
systemd-analyze calendar '*-*-* 03:00:00'
systemd-analyze calendar 'hourly'

# Normalise a shorthand into its full form
systemd-analyze calendar 'daily'
```

*Tip:* `systemd-analyze calendar` prints the normalised expression and the next elapse time. If the next run is not what you expected, the expression is wrong — fix it before enabling the timer.

## Cron Deep Dive: Syntax, crontab, and anacron

Cron is still the simplest way to schedule a job, and it is present on almost every Linux system. The syntax is compact but unforgiving, and a single misplaced field silently changes when a job runs.

### The Five Fields

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── day of week (0-7, 0 and 7 are Sunday)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

### Common Schedules

```cron
# Every minute
* * * * * /usr/local/bin/check.sh

# Every 5 minutes
*/5 * * * * /usr/local/bin/check.sh

# Every hour at minute 0
0 * * * * /usr/local/bin/hourly.sh

# Every day at 2:30 a.m.
30 2 * * * /usr/local/bin/backup.sh

# Every Monday at 9 a.m.
0 9 * * 1 /usr/local/bin/weekly.sh

# First day of every month at midnight
0 0 1 * * /usr/local/bin/monthly.sh

# Every weekday at 6 p.m.
0 18 * * 1-5 /usr/local/bin/report.sh
```

### Managing crontabs

```bash
# Edit the current user's crontab
crontab -e

# List the current user's crontab
crontab -l

# Remove the current user's crontab
crontab -r

# Edit another user's crontab (as root)
sudo crontab -u alice -e

# System-wide crontabs
ls /etc/cron.d/
cat /etc/crontab
```

*Tip:* `crontab -r` deletes the entire crontab without confirmation. If you meant `crontab -e`, you have just lost every scheduled job for that user.

### Environment and PATH

Cron runs jobs with a minimal environment. `PATH` is usually just `/usr/bin:/bin`, which is why scripts that work interactively fail under cron.

```cron
# Set PATH explicitly at the top of the crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Set other environment variables
SHELL=/bin/bash
MAILTO=ops@example.com
```

*Tip:* Always use absolute paths in cron jobs, or set `PATH` explicitly. "It works when I run it by hand" is almost always a `PATH` problem.

### The `%` Gotcha

In a crontab, `%` means "newline" and everything after it becomes the command's stdin. A `date +%Y` in a cron job silently truncates the command.

```cron
# Wrong: % is interpreted by cron, the command becomes "date +"
0 3 * * * date +%Y-%m-%d >> /var/log/date.log

# Right: escape the percent signs
0 3 * * * date +\%Y-\%m-\%d >> /var/log/date.log
```

*Tip:* This only applies to crontabs. Scripts called *from* cron are normal shell scripts and do not need escaping.

### Where cron Files Live

```bash
# Per-user crontabs
/var/spool/cron/crontabs/

# System-wide
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
```

*Tip:* Files in `/etc/cron.d/` need a **user field** between the schedule and the command, unlike user crontabs. Forgetting it is the most common reason a `/etc/cron.d/` entry never runs.

### Logging Cron Output

```cron
# Redirect stdout and stderr to a log file
30 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Discard output entirely
30 2 * * * /usr/local/bin/backup.sh >/dev/null 2>&1
```

*Tip:* By default, cron mails output to the user. On a server with no mail configured, that output is silently lost. Always redirect it to a file or to syslog.

### anacron: For Machines That Are Not Always On

Cron assumes the machine is running at the scheduled time. `anacron` runs missed jobs on the next boot, which is what you want on laptops and workstations.

```bash
# /etc/anacrontab
# period  delay  job-identifier  command
1         5      daily-backup    /usr/local/bin/backup.sh
7         10     weekly-cleanup  /usr/local/bin/cleanup.sh
```

*Tip:* `anacron` is not a replacement for cron on servers. It is a complement for machines that may be off at the scheduled time, and it is why desktop systems still run their daily jobs reliably.

### Cron vs. systemd Timers, Revisited

The table earlier in this post summarises the trade-offs. In practice:

* **Use cron** for simple, self-contained jobs, especially on systems where you already have a crontab workflow.
* **Use systemd timers** when you want logging, dependency ordering, resource limits, or catch-up on missed runs.

*Tip:* `crontab -l` is the fastest way to audit a user's scheduled jobs, and `systemctl list-timers --all` is the equivalent for systemd. Check both when you inherit a host.

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

## systemd-run: Transient Units and Resource Limits

`systemd-run` starts a command as a transient unit without writing a unit file. It is the fastest way to apply cgroup limits to a one-off command.

```bash
# Run a command as a transient service
sudo systemd-run --unit=myjob /usr/local/bin/backup.sh

# Run with a memory limit
sudo systemd-run --scope -p MemoryMax=512M /usr/local/bin/heavy.sh

# Run with a CPU quota
sudo systemd-run --scope -p CPUQuota=50% /usr/local/bin/heavy.sh

# Run a transient timer
sudo systemd-run --on-calendar="*-*-* 02:00:00" /usr/local/bin/backup.sh

# List transient units
systemctl list-units --type=service | grep run-
```

*Tip:* `systemd-run --scope` runs the command in the caller's cgroup, which is useful for interactive commands. Without `--scope`, it creates a separate service unit.

## systemd-cgtop: Live Resource Usage per cgroup

```bash
# Live view of cgroup resource usage
systemd-cgtop

# Sort by memory
systemd-cgtop -m

# Sort by CPU
systemd-cgtop -c

# Show only a specific slice
systemd-cgtop /system.slice
```

*Tip:* `systemd-cgtop` is the fastest way to see which service is consuming CPU or memory right now, without attaching to each process individually.

## systemd-analyze security: Scoring Unit Hardening

```bash
# Score a unit from 0 (bad) to 10 (good)
systemd-analyze security myapp

# Show the full list of directives and their status
systemd-analyze security myapp --no-pager

# Score every running service
systemd-analyze security --recursive
```

*Tip:* `systemd-analyze security` lists every directive you have not set, which makes it the fastest way to find the next hardening step.

## systemd-resolved and resolvectl

`systemd-resolved` is the local DNS stub resolver on many distributions.

```bash
# Is it running?
systemctl status systemd-resolved

# Show the current DNS servers
resolvectl status

# Query a name
resolvectl query example.com

# Flush the cache
sudo resolvectl flush-caches

# Show cache statistics
resolvectl statistics
```

*Tip:* `resolvectl query` uses the same resolution path as applications, including `/etc/hosts` and per-link DNS. It is the fastest way to confirm what a host will actually resolve.

## systemd-networkd

`systemd-networkd` is a lightweight network manager that reads `.network` and `.netdev` files.

```bash
# Is it running?
systemctl status systemd-networkd

# Show link status
networkctl status

# List links
networkctl list

# Reload configuration
sudo networkctl reload
```

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=10.0.0.5/24
Gateway=10.0.0.1
DNS=8.8.8.8
```

*Tip:* `networkctl status` shows the link state, the addresses, and which `.network` file matched. It is the fastest way to debug a networkd configuration.

## systemd-boot and Unified Kernel Images

`systemd-boot` is a simple UEFI boot manager, and Unified Kernel Images (UKIs) bundle the kernel, initramfs, and command line into a single signed file.

```bash
# Install systemd-boot
sudo bootctl install

# Show boot entries
bootctl list

# Show the current boot status
bootctl status

# Update the bootloader
sudo bootctl update

# Build a UKI
sudo kernel-install add $(uname -r) /boot/vmlinuz-$(uname -r)
```

*Tip:* UKIs are the cleanest way to sign a boot chain. The kernel, initramfs, and command line are all covered by a single signature, which makes Secure Boot verification straightforward.

## systemd-nspawn and machinectl

`systemd-nspawn` runs a container using namespaces and cgroups, with no daemon and no image format.

```bash
# Boot a container from a directory
sudo systemd-nspawn -D /var/lib/machines/mycontainer

# Boot a container and log in
sudo systemd-nspawn -bD /var/lib/machines/mycontainer

# List running containers
machinectl list

# Open a shell in a running container
sudo machinectl shell mycontainer

# Pull a container image
sudo machinectl pull-tar https://example.com/container.tar
```

*Tip:* `systemd-nspawn` is the simplest way to run a full system container on a single host. It is not a Docker replacement, but it is far lighter than a VM.

## systemd-creds and systemd-cryptenroll

`systemd-creds` encrypts secrets for use in unit files, and `systemd-cryptenroll` binds LUKS volumes to TPM2 or FIDO2.

```bash
# Encrypt a credential
systemd-creds encrypt --name=db-password - /etc/myapp/db.cred

# Use it in a unit
# LoadCredentialEncrypted=db-password:/etc/myapp/db.cred

# Enroll a TPM2 device in a LUKS volume
sudo systemd-cryptenroll --tpm2-device=auto /dev/sda3

# Enroll a FIDO2 key
sudo systemd-cryptenroll --fido2-device=auto /dev/sda3

# List enrolled keys
sudo systemd-cryptenroll /dev/sda3
```

*Tip:* `LoadCredentialEncrypted=` is the correct way to pass secrets to a service. The secret is decrypted into the service's private credential directory and never appears in the unit file or the environment.

## systemd-sysext and systemd-confext

`systemd-sysext` overlays extension images onto `/usr` and `/opt`, and `systemd-confext` does the same for `/etc`.

```bash
# Merge extension images
sudo systemd-sysext merge

# Show merged extensions
systemd-sysext status

# Unmerge
sudo systemd-sysext unmerge

# Refresh after adding an image
sudo systemd-sysext refresh
```

*Tip:* sysext images are read-only overlays, which makes them ideal for shipping vendor agents or debug tools without modifying the base OS.

## systemd-repart and systemd-homed

`systemd-repart` grows or creates partitions from a declarative config, and `systemd-homed` manages home directories as portable, encrypted images.

```bash
# Show what repart would do
sudo systemd-repart --dry-run=yes

# Apply the partition changes
sudo systemd-repart

# Create a homed user
sudo homectl create alice --storage=luks

# List homed users
homectl list

# Activate a home directory
sudo homectl activate alice
```

*Tip:* `systemd-repart` is the cleanest way to grow a root partition on first boot in a cloud image. It reads `/usr/lib/repart.d/*.conf` and applies the changes idempotently.

## systemd-oomd

`systemd-oomd` kills cgroups under memory pressure before the kernel OOM killer has to act.

```bash
# Is it running?
systemctl status systemd-oomd

# Show the current pressure
systemd-oomdctl

# Check the configured thresholds
cat /etc/systemd/oomd.conf
```

*Tip:* `systemd-oomd` acts on cgroup-level pressure, which is far more targeted than the kernel OOM killer. It is enabled by default on Fedora and many desktop distributions.

## systemd-pcrlock

`systemd-pcrlock` manages TPM2 PCR policies, which is how you bind a LUKS volume to a specific boot chain.

```bash
# Show the current PCR policy
sudo systemd-pcrlock

# Lock a policy
sudo systemd-pcrlock lock

# Remove a policy
sudo systemd-pcrlock remove
```

*Tip:* `systemd-pcrlock` is the modern replacement for hand-written PCR policies. It is the correct way to bind disk encryption to Secure Boot measurements.

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
