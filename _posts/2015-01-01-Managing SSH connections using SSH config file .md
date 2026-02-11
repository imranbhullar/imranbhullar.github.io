---
layout: post
title: "Managing SSH connections using SSH config file"
date: 2015-01-01 00:00:00 +1100
description: "Learn how to simplify your DevOps workflow by automating complex SSH commands using the ~/.ssh/config file."
categories: [DevOps, Linux]
tags: [ssh, linux-admin, sysadmin, productivity, automation]
---
![ssh](/assets/img/posts/ssh.jpg)


One of the biggest messes a *nix admin has to handle is the list of all servers, their SSH ports, usernames, keys, and more. While there are dedicated SSH connection managers available, it can be hard for a Linux admin to leave the terminal—especially when working on a remote machine like a jump or bastion host without a GUI.

The good news is that this mess can be automated and managed systematically. This post explains how to manage all your SSH connections in a simple configuration file.

## SSH Precedence: How Parameters are Read

When you log into a machine, SSH reads parameters in a specific order of priority:

1.  **User command line options:** Standard SSH login commands (takes highest precedence).
2.  **~/.ssh/config:** The user-level configuration file (second precedence).
3.  **/etc/ssh/ssh_config:** The system-wide configuration file (third precedence).

In this post, we will focus on the second option: **~/.ssh/config**.

## The Problem: Long, Cumbersome Commands

Imagine you need to log into a remote MySQL database server using this command:

```bash
ssh -i /shared/keys/dbkeys/dbkey.pem -p 2222 -l dbadmin sec.db1.enigmabits.net
```
Typing this every time is inefficient. To fix this, we can define these parameters in `~/.ssh/config`.

The Solution: Defining Hosts
Add the following block to your `~/.ssh/config` file:

```bash
Host db1
    HostName sec.db1.enigmabits.net
    User dbadmin
    Port 2222
    IdentityFile /shared/keys/dbkeys/dbkey.pem
```    
Now, you can simply type:


`ssh db1`

SSH will automatically pull the rest of the parameters from the config file.
Setting Global Defaults
If you want default settings for all hosts (like a specific port or `keep-alive` settings), use the `Host *` parameter. These will be used unless overridden by a specific host entry.

```bash
Host *
    User nix
    Port 22
    Protocol 2
    ServerAliveInterval 90
    ServerAliveCountMax 10
```
### Minimalist Configurations

If you are using your workstation's default username and private key, you only need to specify the HostName:

```bash
Host jumphost2
    HostName sec.jumphost2.enigmabits.net
```

### Widely Used Parameters


While man `SSH_CONFIG` provides the full list, here are the essentials:

`Host`: A nickname for the connection or * for global rules.

`HostName`: The actual IP address or FQDN of the remote server.

`User`: The remote username.

`Port`: The SSH port (defaults to 22).

`ServerAliveInterval`: Sends a "keep-alive" message every X seconds to prevent timeouts.

`IdentityFile`: Path to your specific private key (PEM/RSA).

`StrictHostKeyChecking`: If set to no, it prevents being blocked when a host key changes (use with caution).

Putting It All Together: A Sample Config
Here is a complete example of a robust `~/.ssh/config` file:

### Default for all SSH connections ###
```bash
Host *
    User nix
    Port 22
    Protocol 2
    ServerAliveInterval 90
    ServerAliveCountMax 10
    StrictHostKeyChecking no
```
## Database Servers ##
```bash
Host db1
    HostName sec.db1.enigmabits.net
    User dbadmin
    Port 2222
    IdentityFile /shared/keys/dbkeys/dbkey.pem

Host db2
    HostName sec.db2.enigmabits.net
    User dbadmin
    Port 2222
    IdentityFile /shared/keys/dbkeys/dbkey.pem
```
## Storage Servers (On-Prem) ##
```bash
Host nas01
    HostName 172.24.1.250
    User nasadmin
    IdentityFile /shared/keys/naskeys/server1/naskey.pem

Host nas02
    HostName 172.24.1.251
    User nasadmin
    IdentityFile /shared/keys/naskeys/naskey.pem
```
## Staging Environment Jump/Bastion Hosts ##
``` bash
Host jumphost1
    HostName sec.jumphost1.enigmabits.net
    User nix
    IdentityFile /home/nix/jumphost1.pem

Host jumphost2
    HostName sec.jumphost2.enigmabits.net
```
### Conclusion
SSH is a powerful utility, but remembering long server names and specific keys is a hurdle we don't need. By utilizing the ~/.ssh/config file, you can streamline your workflow and focus on the actual work instead of the connection details.