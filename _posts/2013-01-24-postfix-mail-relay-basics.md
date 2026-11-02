---
layout: post
title: "Postfix Basics: Setting Up a Linux Mail Relay for Alerts"
date: Jan-24-2013
categories: [Linux, SysAdmin]
tags: [linux, postfix, mail, smtp, sysadmin]
image: /assets/img/postfix-mail-relay-letters.jpg
---

A server doesn't need to host mailboxes to need outbound mail. Cron jobs email their failures, monitoring tools email their alerts, and `logrotate` emails you when something goes wrong — all of it depends on a working local MTA. Postfix set up as a satellite relay, forwarding everything through an external provider, is the smallest useful configuration for this.

## Installing Postfix as a Satellite System

During install, most distros ask what kind of mail configuration you want; "Satellite system" or "No local mail delivery" is the right answer for a server that only sends, never receives.

```bash
sudo apt install postfix
# Debconf prompt: General type of mail configuration -> Satellite system
# System mail name: myhost.example.com
# SMTP relay host: [smtp.provider.example]:587

# RHEL/Fedora installs without the prompt; edit main.cf directly afterward
sudo dnf install postfix
```

*Tip:* A "satellite system" never queues mail for local users — everything, including root's mail, gets forwarded to the relay host. That's exactly the behavior you want for an alerting-only box.

## The Key Settings in main.cf

Postfix's entire runtime configuration lives in `/etc/postfix/main.cf`; a satellite relay only needs a handful of directives set correctly.

```bash
sudo vi /etc/postfix/main.cf
# myhostname = myhost.example.com
# mydomain = example.com
# myorigin = $mydomain
# relayhost = [smtp.provider.example]:587
# inet_interfaces = loopback-only
# inet_protocols = ipv4
```

*Tip:* `inet_interfaces = loopback-only` matters: it means Postfix only listens on 127.0.0.1, not on a public interface. This box sends mail; it has no business accepting SMTP connections from the internet.

## Authenticating to an External Relay

Most providers (and most ISPs blocking port 25 outbound) require SASL authentication to their submission endpoint on port 587.

```bash
sudo apt install libsasl2-modules

# /etc/postfix/sasl_passwd
echo "[smtp.provider.example]:587 alerts@example.com:yourpassword" | \
  sudo tee /etc/postfix/sasl_passwd

sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

*Tip:* `postmap` compiles the plaintext map into the `.db` lookup table Postfix actually reads. Forgetting to re-run it after editing `sasl_passwd` is the most common reason credential changes don't take effect.

## Wiring SASL Into main.cf

With the password map in place, tell Postfix to actually use it for outbound authentication.

```bash
sudo postconf -e "smtp_sasl_auth_enable = yes"
sudo postconf -e "smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd"
sudo postconf -e "smtp_sasl_security_options = noanonymous"
sudo postconf -e "smtp_tls_security_level = encrypt"

sudo systemctl restart postfix
```

*Tip:* `postconf -e` edits `main.cf` in place and is safer than hand-editing for one-line changes, since it won't accidentally duplicate an existing directive further down the file.

## Making Root's Mail Go Somewhere a Human Reads

By default, system mail addressed to `root` sits in a local mailbox nobody checks. `/etc/aliases` redirects it to a real address.

```bash
sudo vi /etc/aliases
# root: alerts@example.com

sudo newaliases
```

*Tip:* `newaliases` rebuilds the aliases database from the plain-text file, the same way `postmap` does for SASL passwords. Any alias edit without it is silently ignored until the next Postfix restart forces a reload anyway.

## Sending a Test Message

Before trusting this for real alerts, confirm mail actually leaves the box and arrives.

```bash
echo "Test message body" | mail -s "Postfix relay test" you@example.com

# Or build a full message by hand
printf "To: you@example.com\nSubject: Test\n\nBody text\n" | sendmail -t
```

*Tip:* If the `mail` command isn't found, install `mailutils` (Debian/Ubuntu) or `mailx` (RHEL/Fedora) — Postfix provides the transport, not the user-facing `mail` client.

## Watching the Mail Log

Every send attempt, bounce, and deferral is logged in detail; this is where you confirm what actually happened.

```bash
# Debian/Ubuntu
sudo tail -f /var/log/mail.log

# RHEL/Fedora
sudo tail -f /var/log/maillog

# A successful relay hand-off looks like:
# ... status=sent (250 2.0.0 OK ... queued as ...)
```

*Tip:* `relay=none, status=deferred` with a connection-refused reason almost always means either the relayhost or the port is wrong, not a credentials problem — check those first before touching SASL settings again.

## Managing the Mail Queue

Messages that can't be delivered immediately sit in Postfix's queue, where you can inspect, retry, or purge them.

```bash
# List everything currently queued
mailq

# Force an immediate delivery attempt for the whole queue
sudo postqueue -f

# Delete a specific stuck message by its queue ID
sudo postsuper -d 4A1B2C3D4E
```

*Tip:* A queue that keeps growing usually means every message is hitting the same failure — check the mail log for the first entry's exact error before assuming a retry will fix it.

## Troubleshooting Stuck Mail

When mail refuses to leave the queue, the cause is almost always one of a short list.

```bash
# Confirm Postfix is actually running and listening locally
sudo systemctl status postfix
ss -tlnp | grep :25

# Test raw connectivity to the relay host and port from this server
nc -zv smtp.provider.example 587

# Re-check the compiled SASL password map matches main.cf's path
postconf smtp_sasl_password_maps
```

*Tip:* A relay host that works from your laptop but not from the server usually points at outbound firewall rules blocking 587 or 465 at the network level — confirm with `nc -zv` before assuming it's a Postfix misconfiguration.
