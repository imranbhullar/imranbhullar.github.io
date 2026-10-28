---
layout: post
title: "Check TLS Certificate Expiry from the CLI"
date: Feb-18-2012
categories: [Linux, Networking, DevOps]
tags: [linux, tls, ssl, openssl, certificates, troubleshooting]
image: /assets/img/Linux_unix_bsd.png
---

An expired TLS certificate is one of the most common causes of "the site is down" incidents, and it is also one of the easiest to prevent. These are the commands I use to check expiry dates, inspect certificate chains, and script expiry monitoring.

## Checking a Remote Certificate with `openssl s_client`

The canonical way to inspect a certificate served by a remote host.

```bash
# Show the full certificate chain
openssl s_client -connect example.com:443 -servername example.com </dev/null

# Show only the leaf certificate
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Show the expiry date only
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -enddate
```

*Tip:* Always pass `-servername` when checking a host that serves multiple certificates. Without SNI, you may get the default certificate, which is often not the one your users see.

## Checking a Local Certificate File

For certificates on disk, `openssl x509` reads the file directly.

```bash
# Subject, issuer, and validity dates
openssl x509 -in /etc/ssl/certs/example.pem -noout -subject -issuer -dates

# Expiry date only
openssl x509 -in /etc/ssl/certs/example.pem -noout -enddate

# Full certificate details
openssl x509 -in /etc/ssl/certs/example.pem -noout -text
```

*Tip:* `-enddate` prints `notAfter=...`, which is easy to parse in scripts. `-dates` prints both `notBefore` and `notAfter`.

## Checking Expiry with `curl`

`curl` can report certificate details without any extra tooling, which is handy on minimal systems.

```bash
# Verbose output includes certificate dates
curl -vI https://example.com 2>&1 | grep -E "expire date|start date|subject:|issuer:"

# Fail silently if the certificate is invalid
curl -sS -o /dev/null -w "%{http_code}\n" https://example.com
```

*Tip:* `curl -vI` performs a HEAD request and prints the TLS handshake details, including the certificate expiry date, in a single command.

## Checking Expiry with `gnutls-cli`

If `openssl` is not available, `gnutls-cli` is a solid alternative.

```bash
# Show certificate details
gnutls-cli --print-cert example.com </dev/null | openssl x509 -noout -dates

# Or use gnutls-cli's own output
gnutls-cli -p 443 example.com </dev/null
```

*Tip:* `gnutls-cli` is often installed on systems that ship with GnuTLS instead of OpenSSL, such as some minimal container images.

## Scripting Expiry Monitoring

A short script that warns when a certificate is close to expiry is worth its weight in gold.

```bash
#!/usr/bin/env bash
# Warn if the certificate expires within 30 days
HOST="example.com"
PORT="443"
WARN_DAYS=30

end_date=$(openssl s_client -connect "${HOST}:${PORT}" -servername "${HOST}" </dev/null 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2)

end_epoch=$(date -d "${end_date}" +%s)
now_epoch=$(date +%s)
days_left=$(( (end_epoch - now_epoch) / 86400 ))

echo "${HOST}: ${days_left} days until expiry"

if [ "${days_left}" -lt "${WARN_DAYS}" ]; then
  echo "WARNING: certificate for ${HOST} expires in ${days_left} days" >&2
  exit 1
fi
```

*Tip:* Run this from cron or a systemd timer daily and alert on non-zero exit. It catches expiries weeks before users do.

## Verifying the Full Chain

An expired intermediate certificate can break clients even when the leaf certificate is valid.

```bash
# Verify the chain against the system trust store
openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null

# Verify a certificate file against a CA bundle
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/example.pem
```

*Tip:* `-showcerts` prints every certificate in the chain. Check the `notAfter` of each one, not just the leaf.

## Quick Reference

```bash
# Remote certificate expiry
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -enddate

# Local certificate expiry
openssl x509 -in cert.pem -noout -enddate

# Quick check with curl
curl -vI https://example.com 2>&1 | grep "expire date"
```

*Tip:* Check expiry on every certificate in the chain, not just the leaf. The most common surprise is an expired intermediate that the leaf certificate still points to.
