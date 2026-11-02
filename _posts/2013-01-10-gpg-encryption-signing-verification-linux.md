---
layout: post
title: "GPG on Linux: Encrypting, Signing, and Verifying Files"
date: Jan-10-2013
categories: [Linux, Security]
tags: [linux, gpg, encryption, security]
image: /assets/img/gpg-padlock-key.jpg
---

Every distro ships GnuPG, and almost every sysadmin only touches it twice a year: to verify a downloaded ISO's signature, or to encrypt something in a panic before sending it over email. That's a shame, because GPG covers encryption, signing, and identity verification with the same small set of commands. This post covers the parts worth actually knowing.

## Generating a Keypair

A GPG identity is a public/private keypair tied to a name and email address. The private key never leaves your machine; the public key is what you hand out.

```bash
gpg --full-generate-key
# Choose: (1) RSA and RSA, 4096 bits, key does not expire (or set 1-2y)
# Real name: Alice Admin
# Email address: alice@example.com

# List the keys now in your keyring
gpg --list-secret-keys --keyid-format=long
# sec   rsa4096/3AA5C34371567BD2 2013-01-10 [SC]
#       Key fingerprint = 7F2D 1A3B ... 3AA5 C343 7156 7BD2
# uid                 [ultimate] Alice Admin <alice@example.com>
```

*Tip:* Note the key ID (`3AA5C34371567BD2`) or fingerprint now. You'll need it to refer to this specific key in every command below, especially once you have more than one.

## Backing Up and Exporting Your Keys

Losing your private key means losing access to everything ever encrypted for it. Export both halves before you need them.

```bash
# Export your public key, to hand out or upload
gpg --armor --export alice@example.com > alice-public.asc

# Export your private key, to back up somewhere safe (encrypted volume, not email)
gpg --armor --export-secret-keys alice@example.com > alice-private.asc

# Generate a revocation certificate while you still can
gpg --output alice-revoke.asc --gen-revoke alice@example.com
```

*Tip:* Store `alice-private.asc` and the revocation certificate on encrypted offline media. If your key is ever compromised, the revocation certificate is the only clean way to tell the world to stop trusting it.

## Importing Someone Else's Public Key

To encrypt something for someone else, or verify something they signed, you need their public key in your keyring first.

```bash
# Import a key from a file
gpg --import bob-public.asc

# Or fetch it from a keyserver by key ID
gpg --keyserver hkps://keys.openpgp.org --recv-keys 4BD6EC30

# List imported public keys
gpg --list-keys
```

*Tip:* An imported key isn't automatically trusted. GPG will still warn you when encrypting to it until you explicitly sign or trust it with `gpg --sign-key` or `gpg --edit-key` → `trust`.

## Encrypting a File for a Recipient

Public-key encryption means only the holder of the matching private key can decrypt it — not even you, once it's encrypted for someone else.

```bash
# Encrypt report.pdf so only Bob can read it
gpg --encrypt --recipient bob@example.com report.pdf
# Produces report.pdf.gpg

# Encrypt for multiple recipients at once
gpg --encrypt --recipient bob@example.com --recipient carol@example.com report.pdf

# ASCII-armored output instead of binary, handy for pasting into email
gpg --armor --encrypt --recipient bob@example.com report.pdf
```

*Tip:* GPG will refuse to encrypt to a key it doesn't trust without `--trust-model always` or an explicit confirmation prompt. Don't bypass that warning for a key you haven't actually verified belongs to who you think it does.

## Decrypting a File

Decryption uses your own private key and, if it's protected, prompts for its passphrase.

```bash
gpg --output report.pdf --decrypt report.pdf.gpg
# You need a passphrase to unlock the secret key for
# user: "Alice Admin <alice@example.com>"
# Enter passphrase: ********
```

*Tip:* If `gpg-agent` is running, it caches your passphrase for a few minutes so repeated decrypts in a script or session don't re-prompt every time. Check its cache timeout in `~/.gnupg/gpg-agent.conf` if that behavior surprises you.

## Symmetric Encryption Without a Keypair

Sometimes there's no recipient keypair involved at all, just a shared secret both sides already know, like a password communicated over the phone.

```bash
# Encrypt with a passphrase instead of a keypair
gpg --symmetric --cipher-algo AES256 secrets.txt
# Enter passphrase: ********
# Produces secrets.txt.gpg

# Decrypt with the same passphrase
gpg --output secrets.txt --decrypt secrets.txt.gpg
```

*Tip:* `--symmetric` is the right tool for encrypting a personal backup archive you'll decrypt yourself later, where setting up a full keypair exchange would be overkill.

## Signing a File

Signing proves a file came from you and hasn't been altered, without necessarily hiding its contents.

```bash
# Detached signature, keeps the original file untouched, produces release.tar.gz.sig
gpg --detached-sign release.tar.gz

# ASCII-armored detached signature
gpg --armor --detached-sign release.tar.gz

# Clear-sign a text message, wraps it with a visible, verifiable signature block
gpg --clearsign announcement.txt
```

*Tip:* Software projects publish a detached `.sig` or `.asc` file alongside a tarball for exactly this reason: it lets you verify integrity without needing to decrypt or alter the original artifact at all.

## Verifying a Signature

Verification confirms both that the signature is valid and which key signed it, which only means something once you've independently confirmed that key belongs to who you think.

```bash
# Verify a detached signature against the file it covers
gpg --verify release.tar.gz.sig release.tar.gz
# gpg: Good signature from "Alice Admin <alice@example.com>" [ultimate]

# Verify a clear-signed message
gpg --verify announcement.txt.asc
```

*Tip:* "Good signature" only tells you the file matches what was signed and the signing key is in your keyring — it does not mean the key is trustworthy. Check the fingerprint against a channel you trust (the project's website, a conference badge, a prior in-person exchange) before relying on it.

## Checking Key Fingerprints

Key IDs can collide or be spoofed; the full fingerprint is what you actually compare when verifying someone's identity.

```bash
gpg --fingerprint bob@example.com
# pub   rsa4096/4BD6EC30 2012-05-14 [SC]
#       Key fingerprint = 1234 5678 9ABC DEF0 1234  5678 9ABC DEF0 1234 5678
```

*Tip:* Read fingerprints out loud in groups of four when confirming them over a phone call or in person. It's slower than eyeballing a screen, but it's how transposition errors actually get caught.
