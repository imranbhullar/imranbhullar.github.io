---
layout: post
title: "NFS: Network File Sharing Service (Server)"
date: 2012-08-01 10:00:00 +0500
description: "A comprehensive guide on setting up and configuring the Network File System (NFS) protocol for file sharing in UNIX environments."
categories: linux, LFCS
tags: [nix-howtos, LFCS, nfs, networking, server-configuration, linux]
---
![nix](/assets/img/Linux_unix_bsd.png)

NFS (Network File System) is a protocol used for file sharing within UNIX environments. Windows users typically cannot access these shared files, as NFS is specifically designed for UNIX systems.

## Key Information

* **Package Name**: `nfs-utils`
* **Service Name**: `nfs`
* **Port Information**: NFS does not have a specific port; it relies on the portmapper service, which operates on port 111. To check the status of the portmapper service, use:

```bash
service portmap status
netstat -atnp | grep 111
```
Configuration Path: `/etc/exports`
Log File: `/var/log/messages`

*  **Sharing Files and Directories** 
To share a file or directory, you need to specify its path in the `/etc/exports` file using the following format:

`sharename ID/IP permissions`
Configuration Examples:
* Read-Only with Sync:  `/crackers *(ro,sync)`
This shares the `/crackers` directory with read-only access and ensures that new files are visible only after they are fully copied.
* Read-Write with Async:
`/crackers *(rw,async)`
This shares the `/crackers` directory with read-write access, allowing users to see new files before they are fully copied.
* Read-Write for a Specific Subnet:
`/crackers 192.168.0.0/255.255.255.0(rw,sync)`
This grants read-write access to all IPs within the specified subnet.
* Domain-Based Access:
`/crackers *.blackhats.com(rw,sync)`
This shares the `/crackers` directory with all users from the `blackhats.com` domain with read-write permissions.
* Mixed Permissions:
`/crackers *.blackhats.com(ro,sync) EXCEPT godfather.blackhats.com(rw,async)`
This grants read-only access to all users from the `blackhats.com` domain, while `godfather.blackhats.com` has read-write access with async visibility.

* **Accessing NFS Shares**
To access network shares, you must mount them using the following command:

`mount /source:/path@server /Destination`
For example:

`mount /192.168.0.1:/crackers /mnt/nfsshares`
Whenever you create a new mount point, restart the NFS service. However, during file replication, avoid restarting the NFS service. Instead, use the following commands to update:

`exportfs -ra `  # Updates NFS exports
`exportfs -v `   # Displays current mounts
It’s crucial that the root user sets appropriate permissions on these shares to allow user access.

NFS also facilitates installations within UNIX environments, making it a versatile tool for network file management.