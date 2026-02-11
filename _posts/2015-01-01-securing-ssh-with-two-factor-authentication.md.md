---
layout: post
title: "Securing SSH with Two-Factor Authentication (2FA) using Google Authenticator"
date: 2015-01-01 00:00:00 +0000
description: "Take your SSH security to the next level by adding a TOTP layer. Learn how to configure Google Authenticator for SSH on Linux."
categories: [Security, Linux]
tags: [ssh, 2fa, google-authenticator, centos, hardening]
---
![ssh](/assets/img/posts/ssh.jpg)


In a previous post, we looked at how to set up passwordless SSH authentication to get rid of managing "creepy" passwords. Today, we’re taking security to the next level by enabling **Two-Factor Authentication (2FA)**. By doing this, even if your private key is lost or stolen, an attacker still won't be able to log in to your machines without the time-based token.

In many production environments, access is restricted to a single **Jump Host** or **Bastion** server. Hardening this specific entry point with 2FA is a best practice for securing your entire infrastructure. 

> **Note:** This guide was tested on CentOS 7, but the logic applies to most Linux distributions (Ubuntu, Debian, Fedora). Only the package manager (`yum` vs `apt`) and specific file paths might differ slightly.

---

## Pre-requisites

1.  **Key-based Authentication:** Ensure you have already set up SSH keys. If not, follow our [previous guide](https://www.imranbhullar.com/posts/SSH-Password-less-authentication-using-ssh-keygen-&-ssh-copy-id/).
2.  **Google Authenticator App:** Download the app to your mobile device to generate the 30-second TOTP tokens.
    * [Android (Play Store)](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2)
    * [Apple (App Store)](https://itunes.apple.com/us/app/google-authenticator/id388497605?mt=8)

---

## 1. Installing Necessary Packages

On CentOS 7, the Google Authenticator module is available via the **EPEL** repository.

```bash
# Install EPEL repository
sudo yum -y install epel-release

# Install the Google Authenticator PAM module
sudo yum install google-authenticator -y
```

## 2. Configuring the Authenticator
Login as the non-root user for whom you want to enable 2FA and run the utility:


```Bash
google-authenticator
Step-by-Step Prompts:
Time-based tokens? Type y.
```
`QR Code`: A large QR code will appear. Scan this with your mobile app.

![QRCODE](/assets/img/posts/qrcode.png "QRCODE")

`Emergency Codes`: Save the "emergency scratch codes" in a safe place (like a password manager). These are your only way in if you lose your phone.

`Update your configuration`? Type y to save the secret to ~/.google_authenticator.

`Disallow multiple users`? Type y to prevent replay attacks.

`Increase time window?` Type y if you worry about time-drift between your phone and the server.

`Enable rate-limiting?` Type y to prevent brute-force attempts.

## 3. Configuring System Files
We need to modify two files to tell the system to ask for the token.

A. Modify `/etc/pam.d/sshd`
Append the following line to the bottom of the file:

`auth required pam_google_authenticator.so nullok`
(The nullok flag allows users who haven't set up 2FA yet to still log in. Remove it once everyone is set up.)

Next, find and comment out the password authentication line so the system doesn't fallback to standard passwords:
```bash
# auth       substack     password-auth
```
B. Modify `/etc/ssh/sshd_config`
Edit the SSH daemon configuration to allow challenge-response prompts:
```bash
# Change to yes
`ChallengeResponseAuthentication yes`

# Ensure the old 'no' setting is commented out
# ChallengeResponseAuthentication no
```
Finally, add this line to the bottom of the file to require both a public key and the 2FA code:

AuthenticationMethods publickey,keyboard-interactive
## 4. Restart and Test
Restart the SSH service to apply changes:

```Bash
sudo systemctl restart sshd.service
```
Now, try to log in from a new terminal window. Do not close your current session until you verify you can get back in!

```Bash
$ ssh centos@destination_host
Verification code: 
```
Once you enter the 6-digit code from your phone, you're in! You have successfully added a significant layer of security to your server, giving you much-needed peace of mind.