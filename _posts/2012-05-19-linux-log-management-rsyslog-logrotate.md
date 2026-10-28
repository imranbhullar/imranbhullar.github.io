---
layout: post
title: "Linux Log Management: rsyslog, logrotate, and Centralised Logging"
date: May-19-2012
categories: [Linux, SysAdmin]
tags: [linux, logging, rsyslog, logrotate, syslog]
image: /assets/img/Linux_unix_bsd.png
---

Logs are the only reliable record of what happened on a system, and yet they are the first thing to fill a disk and the last thing to be configured properly. This post covers the syslog pipeline, log rotation, and centralised collection.

## The Syslog Pipeline

On most Linux systems, logs flow through `rsyslog` (or `syslog-ng`) to files under `/var/log/`.

```bash
# Is rsyslog running?
systemctl status rsyslog

# Main configuration
cat /etc/rsyslog.conf

# Additional rules
ls /etc/rsyslog.d/

# Where are logs going?
ls -lh /var/log/
```

*Tip:* On systemd hosts, many services log to the journal instead of syslog. `journalctl` is the first place to look, and `rsyslog` may only receive a subset of messages.

## rsyslog Configuration

A rule has a selector and an action.

```bash
# /etc/rsyslog.d/50-default.conf

# All auth messages to a dedicated file
auth,authpriv.*                 /var/log/auth.log

# All messages at warning or above to a file
*.warn                          /var/log/warn.log

# Everything except mail and auth to the main log
*.*;mail.none;auth.none         /var/log/syslog

# Send everything to a remote server over TCP
*.* @@logserver.example.com:514
```

*Tip:* `@` is UDP, `@@` is TCP. Always prefer TCP for remote logging; UDP silently drops messages under load.

## logrotate: Keeping Logs Under Control

`logrotate` rotates, compresses, and deletes old logs on a schedule.

```bash
# Main configuration
cat /etc/logrotate.conf

# Per-application configuration
ls /etc/logrotate.d/

# Test a configuration without rotating
sudo logrotate -d /etc/logrotate.d/nginx

# Force a rotation
sudo logrotate -f /etc/logrotate.d/nginx
```

A typical configuration:

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp myapp
    sharedscripts
    postrotate
        systemctl reload myapp >/dev/null 2>&1 || true
    endscript
}
```

*Tip:* `copytruncate` is tempting for applications that cannot reopen their log file, but it can lose lines written between the copy and the truncate. Prefer `postrotate` with a reload.

## Centralised Logging with rsyslog

Sending logs to a central server is the single biggest improvement you can make to incident response.

```bash
# On the server: enable TCP reception
# /etc/rsyslog.d/10-server.conf
module(load="imtcp")
input(type="imtcp" port="514")

# On the client: forward everything
# /etc/rsyslog.d/50-forward.conf
*.* @@logserver.example.com:514

# Restart both
sudo systemctl restart rsyslog
```

*Tip:* Use TLS for remote logging on any network you do not fully control. Plain TCP syslog sends credentials and log contents in clear text.

## Structured Logging

Plain text logs are hard to query. Structured logs (JSON) are not.

```bash
# Emit JSON from a script
printf '{"time":"%s","level":"info","msg":"started"}\n' "$(date -Iseconds)"

# Query JSON logs with jq
journalctl -u myapp -o json | jq -r 'select(.PRIORITY == "3") | .MESSAGE'

# Query a JSON log file
jq -r 'select(.level == "error") | .msg' /var/log/myapp/app.json
```

*Tip:* If you control the application, log JSON. If you do not, `jq` can still parse most structured formats, and `journalctl -o json` gives you structured output for free.

## Disk Usage and Retention

```bash
# How much space are logs using?
du -sh /var/log/*

# Journal disk usage
journalctl --disk-usage

# Vacuum the journal to a size limit
sudo journalctl --vacuum-size=500M

# Vacuum to a time limit
sudo journalctl --vacuum-time=30d
```

*Tip:* A full `/var/log` is one of the most common causes of a full root filesystem. Set a journal size limit and a logrotate retention policy on every host.

## Quick Reference

```bash
# Where are logs going?
ls -lh /var/log/

# Test logrotate
sudo logrotate -d /etc/logrotate.d/myapp

# Journal disk usage
journalctl --disk-usage

# Vacuum the journal
sudo journalctl --vacuum-size=500M

# Forward logs to a central server
echo '*.* @@logserver.example.com:514' | sudo tee /etc/rsyslog.d/50-forward.conf
```

*Tip:* Work in order: find the logs, rotate them, forward them, then set retention. Most log incidents are a missing rotation policy or a journal that was never size-limited.
