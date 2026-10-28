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

## Generating a CSR

```bash
# Generate a private key and CSR in one step
openssl req -new -newkey rsa:2048 -nodes \
  -keyout example.key -out example.csr \
  -subj "/C=US/ST=CA/L=SF/O=Example/CN=example.com"

# Generate a CSR from an existing key
openssl req -new -key example.key -out example.csr \
  -subj "/CN=example.com"

# Inspect a CSR
openssl req -in example.csr -noout -text

# Verify a CSR's signature
openssl req -in example.csr -noout -verify
```

*Tip:* Always include a Subject Alternative Name (SAN) in the CSR. Modern browsers ignore the Common Name entirely, and a certificate without a SAN is rejected.

## Self-Signed Certificates

```bash
# Generate a self-signed certificate valid for 365 days
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout example.key -out example.crt -days 365 \
  -subj "/CN=example.com" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"

# Inspect the certificate
openssl x509 -in example.crt -noout -text

# Verify the certificate against itself
openssl verify -CAfile example.crt example.crt
```

*Tip:* `-addext "subjectAltName=..."` is the correct way to add a SAN to a self-signed certificate. Without it, the certificate is rejected by every modern client.

## Building a Chain

```bash
# Concatenate the leaf, intermediate, and root
cat example.crt intermediate.crt root.crt > chain.pem

# Verify the chain
openssl verify -CAfile root.crt -untrusted intermediate.crt example.crt

# Show the chain
openssl crl2pkcs7 -nocrl -certfile chain.pem | openssl pkcs7 -print_certs -noout
```

*Tip:* The order matters: leaf first, then intermediates, then the root. A chain in the wrong order is rejected by most clients, and the error message rarely says so.

## PKCS#12: Bundling Key and Certificate

```bash
# Create a PKCS#12 bundle
openssl pkcs12 -export -out example.p12 \
  -inkey example.key -in example.crt -certfile chain.pem

# Inspect a PKCS#12 bundle
openssl pkcs12 -in example.p12 -info -noout

# Extract the key and certificate
openssl pkcs12 -in example.p12 -nocerts -nodes -out example.key
openssl pkcs12 -in example.p12 -clcerts -nokeys -out example.crt
```

*Tip:* PKCS#12 is the standard format for importing a certificate into a Java keystore, a Windows store, or a browser. It bundles the key, the certificate, and the chain in one file.

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
