---
layout: post
title: "Automating the Boring Stuff with Pexpect: A Sysadmin's Guide"
date: 2015-06-06 13:00:00 +0500
description: "Learn how to use the Pexpect Python module to automate CLI interactions with FTP servers, MySQL databases, and network appliances."
categories: [Automation, Python]
tags: [pexpect, sysadmin, devops, networking, scripting]
---
![nix](/assets/img/posts/python-automation.png)

Ever wanted to automate some boring tasks while managing a bench of remote machines? Today we will play with **Pexpect**, an "Expect-like" Python module which can be used to interact with almost any application that has a Command Line Interface (CLI).

### Prerequisites
To follow along, you need:
* **Python** (Works with 2.7.x, though Python 3 is recommended for modern environments).
* **Pexpect module**: Install it via `pip install pexpect`.

Pexpect has two primary interfaces:
1.  **`pexpect.run()`**: Used to interact with the local machine to execute commands (similar to `os.system()`).
2.  **`pexpect.spawn()`**: Used to interact with remote services and external processes. This is the interface we will be using today.

---

### 1. Playing with FTP Servers
If you have a remote FTP server and want to programmatically download files, Pexpect handles the challenge-response pattern beautifully.


```python
import pexpect

child = pexpect.spawn('ftp ftp.freebsd.org')
child.expect('Name .*: ')
child.sendline('anonymous')
child.expect('Password:')
child.sendline('me@mydomain.com')
child.expect('ftp> ')

print(child.before) # Print output before the prompt

child.sendline('cd /pub/FreeBSD')
child.expect('ftp> ')
child.sendline('get README.TXT')
child.expect('ftp> ')
child.sendline('bye')

```

### 2. Playing with MySQL Databases
Sometimes we need to generate reports from DB servers—checking hosted databases, active processes, or global variables. Pexpect can navigate the MySQL shell for you.

```python
import pexpect

# Replace with your actual credentials and host
child = pexpect.spawn('mysql -umyuser -pmysupersecret -hxxx.xxx.xxx.xxx')
child.expect('mysql> ')

child.sendline('show databases;')
child.expect('mysql> ')
print(child.before)

child.sendline('show processlist;')
child.expect('mysql> ')
print(child.before)

child.sendline('quit')
```

### 3. Playing with Network Appliances
Pexpect is incredibly useful for configuring network gear. In this example, we log into a Citrix NetScaler via SSH to update its hostname.

```python
import pexpect

# Bypass StrictHostKeyChecking for automated scripts
child = pexpect.spawn('ssh username@xxx.xxx.xxx.xxx -o StrictHostKeyChecking=no')
child.expect('Password:')
child.sendline('mysupersecret')
child.expect('.*>')

child.sendline('show Hostname')
child.expect('>')
print(child.before)

child.sendline('set HostName EDGE-Device')
child.expect('>')

child.sendline('save config')
child.expect('>')
child.sendline('quit')

```
### Conclusion
While there are specific libraries for specific tasks (like ftplib for FTP or paramiko for SSH), Pexpect remains a powerful, generic tool in a sysadmin's toolkit. If it has a CLI, you can automate it.

Hopefully, this helps my fellow sysadmins in automating those repetitive, boring tasks!