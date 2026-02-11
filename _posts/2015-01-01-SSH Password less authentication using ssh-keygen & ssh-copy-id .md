---
layout: post
title: "SSH Passwordless Authentication using ssh-keygen & ssh-copy-id"
date: 2015-01-01 10:00:00 +0000
description: "A guide to setting up secure, key-based SSH authentication to replace complex password management."
categories: [Tutorials, Security]
tags: [ssh, linux, devops, sysadmin, openssh]
---
![ssh](/assets/img/posts/ssh.jpg)

I still remember the days when one had to remember long passwords to login to network machines. From a security perspective, since "each and every machine" had to use a different password, teams started to store their passwords in different encrypted or password-protected files. It felt a bit funny having a "master password" just to access all the other network passwords!

**Passwordless authentication** is the way to deal with that situation. You don't have to remember or manage creepy passwords; instead, you can simply rely on key-based authentication. This is significantly more secure and is the preferred method for modern authentication. 

### How it Works
In this process, we create a **public and private key pair** on our workstation. We then copy the public key to the remote host. The remote machine then allows authentication from our workstation without a password. 

Think of it in layman's terms: it works like a **lock and key**. You keep the key, and you place a specific lock on the remote host that only your key can open.
---

### 1. Generating Public/Private Keys

The `ssh-keygen` utility is used to create the key pair. Run the following command on your local workstation:

```bash
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ec2-user/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/ec2-user/.ssh/id_rsa.
Your public key has been saved in /home/ec2-user/.ssh/id_rsa.pub.
The key fingerprint is:
94:b4:d8:d3:8a:06:a9:2e:27:d2:1b:b7:dd:2d:c3:f5 ec2-user@ip-172-31-2-70
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|      . + +      |
|     o . * .     |
|    . . o o      |
|   .   o S       |
|  o   .    .     |
| + = .  . . .    |
| .+ + o .+.  E   |
|   . . . .o.     |
+-----------------+
```

The utility creates two files:

* Private Key: /home/ec2-user/.ssh/id_rsa (Keep this secret!)

* Public Key: /home/ec2-user/.ssh/id_rsa.pub (This goes on the server.)

### 2. Copying the Public Key to the Remote Host
Now, use the ssh-copy-id utility to install your public key on the remote server:

``` Bash
ssh-copy-id -i /home/ec2-user/.ssh/id_rsa.pub ouruser@remotehost
```
This command automatically appends your public key to the `~/.ssh/authorized_keys` file on the remote machine. Once finished, you can verify it by checking that file on the server.

### 3. Testing the Connection
If the steps above were successful, you should now be able to log in without entering a password:

```Bash
ssh username@remotehost
```
### 4. Logging in from a Different Machine
If you need to log in from a different machine (other than your primary workstation), you must copy your private key `(id_rsa)`to that machine. You can then use the -i flag to specify the key:

```Bash
ssh -i /path/to/private_key username@remotehost
```
Security Note: Always protect your private key. If someone gains access to your id_rsa file, they can access any server where your public key is installed.

### Conclusion
We've gone through the process of getting rid of long, cumbersome passwords in favor of a more secure and robust authentication method. Hopefully, this makes the lives of system admins a bit simpler!