---
layout: post
title: "Linux Privilege Management: Mastering su, visudo, and Polkit"
description: "A deep dive into substituting users, safe sudoers configuration, and recovering from misconfigurations."
date: 2025-10-12
categories: [LFCS, Linux, DevOps]
tags: [bash, LFCS, visudo, polkit, su]

---
![nix](/assets/img/Linux_unix_bsd.png)

In Linux administration, managing how users gain elevated privileges is a fundamental skill. Whether you are moving between accounts or granting administrative rights, doing it correctly is the difference between a secure system and a broken one.

## 1. The `su` Command: Substituting Users

The `su` (Substitute User) command allows you to run commands with the privileges of another user. By default, it switches to the `root` account.

### Common Variations:
* **`su`**: Switches to the root user but **keeps your current environment** variables (like your home directory and PATH).
* **`su -`** or **`su -l`**: The "l" stands for login. This starts a **full login shell**, loading the target user's profile and environment. This is generally the safest way to switch to root.
* **`su - bob`**: Switches specifically to the user "bob" with a full login environment.

---

## 2. Managing Access with `visudo`

While `su` switches accounts, `sudo` allows a user to run specific commands as root. You should **never** edit `/etc/sudoers` directly with a standard text editor. Instead, always use `visudo`.

### Why use `visudo`?
As seen in terminal logs, `visudo` performs a **syntax check** before saving. If you make a mistake, it will warn you:
`>>> /etc/sudoers.d/bob: syntax error near line 1 <<<`

It provides a "What now?" prompt, preventing you from saving a broken file that could lock everyone out of administrative access.

### Configuration Examples:

| Goal | Sudoers Entry |
| :--- | :--- |
| **Basic Sudo Access** | `bob  ALL=(ALL:ALL) ALL` |
| **Passwordless Sudo** | `bob  ALL=(ALL:ALL) NOPASSWD: ALL` |
| **Restrict to Specific App** | `bob  ALL=(ALL:ALL) /usr/bin/apt-get` |
| **Group Access** | `%sudo  ALL=(ALL:ALL) ALL` |

> **Pro Tip:**  
> For better organization and safer sudo management, avoid editing `/etc/sudoers` directly.  
> Instead, create separate configuration files under `/etc/sudoers.d/` using:
>
> ```bash
> sudo visudo -f /etc/sudoers.d/devops-admins
> ```

### Example: `/etc/sudoers.d/devops-admins`

The following snippet allows members of the `devops` group to:
- Restart services
- View system logs
- Run package updates  
without being prompted for a password.

```sudoers
# Allow DevOps team limited administrative access
%devops ALL=(ALL) NOPASSWD: \
    /bin/systemctl restart *, \
    /bin/systemctl status *, \
    /usr/bin/journalctl *, \
    /usr/bin/apt update, \
    /usr/bin/apt upgrade
```
---

## 3. Polkit (PolicyKit)

Polkit is a separate authorization framework often used in desktop systems. It allows fine-grained control over system-wide services.

If you are running tasks via `pkexec` (the Polkit equivalent of `sudo`) in a headless environment, you might need a TTY Agent:

```bash
# To authenticate in a terminal without a GUI
pkttyagent -p $(echo $$) &
pkexec cat /etc/shadow
```
## 4. How to Fix Broken Sudo

If a user manages to save "garbage" or a misconfiguration into a sudoers file (for example, by forcing a save in visudo with the `Q` option), sudo will likely stop working.

Method 1: Use Polkit (pkexec)
If sudo is broken but Polkit is still configured correctly, you can bypass sudo to fix the file:
```bash
pkexec visudo -f /etc/sudoers.d/broken_file
```
### Method 2: The su Fallback
If you know the root password, switch to the root user directly to delete or fix the configuration:
```bash
su -
# Once root:
rm /etc/sudoers.d/misconfigured_file
```
