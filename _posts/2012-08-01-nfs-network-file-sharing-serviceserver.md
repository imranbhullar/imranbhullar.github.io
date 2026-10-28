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

## Sharing Files and Directories
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

## Accessing NFS Shares
To access network shares, you must mount them using the following command:

`mount /source:/path@server /Destination`
For example:

`mount /192.168.0.1:/crackers /mnt/nfsshares`
Whenever you create a new mount point, restart the NFS service. However, during file replication, avoid restarting the NFS service. Instead, use the following commands to update:

`exportfs -ra `  # Updates NFS exports
`exportfs -v `   # Displays current mounts
It’s crucial that the root user sets appropriate permissions on these shares to allow user access.

NFS also facilitates installations within UNIX environments, making it a versatile tool for network file management.

## NFSv4: The Modern Protocol

NFSv4 is the default on modern systems. It uses a single port (2049), supports Kerberos, and has built-in ACLs.

```bash
# Check the NFS version
nfsstat -m

# Mount with NFSv4
sudo mount -t nfs4 nfs-server:/export/data /mnt/data

# Mount with NFSv4.2
sudo mount -t nfs -o vers=4.2 nfs-server:/export/data /mnt/data

# Show the server's exports
showmount -e nfs-server
```

*Tip:* NFSv4 only needs port 2049, which makes it far easier to firewall than NFSv3. If you can choose, always use NFSv4.

## autofs: Automounting NFS Shares

`autofs` mounts NFS shares on first access, which is the standard way to manage large NFS environments.

```bash
# Install
sudo apt install autofs

# Configure a direct map
# /etc/auto.master.d/data.autofs
# /-  /etc/auto.data

# /etc/auto.data
# /mnt/data  -rw,soft,intr  nfs-server:/export/data

# Restart
sudo systemctl restart autofs
```

*Tip:* `autofs` is more flexible than systemd automount for large NFS environments, because it supports wildcards and indirect maps. Use systemd automount for a handful of mounts and `autofs` for hundreds.

## nfsstat and nfsiostat

```bash
# Show NFS statistics
nfsstat -c    # client
nfsstat -s    # server

# Show per-mount I/O statistics
nfsiostat 2

# Show mount options
nfsstat -m
```

*Tip:* `nfsiostat` is the fastest way to see whether an NFS mount is slow. It shows the read and write latency per mount, which is far more useful than a generic `iostat`.

## pNFS: Parallel NFS

pNFS lets clients read and write directly to storage servers, bypassing the metadata server. It is the standard way to scale NFS for high-performance workloads.

```bash
# Check whether pNFS is in use
cat /proc/self/mountstats | grep -i pnfs

# Mount with pNFS
sudo mount -t nfs -o vers=4.1 nfs-server:/export/data /mnt/data
```

*Tip:* pNFS requires server-side support, which most enterprise NFS servers provide. It is the only way to get near-linear scaling from NFS.
