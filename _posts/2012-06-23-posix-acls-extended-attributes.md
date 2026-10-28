---
layout: post
title: "POSIX ACLs and Extended Attributes: Permissions Beyond chmod"
date: Jun-23-2012
categories: [Linux, SysAdmin]
tags: [linux, permissions, acl, setfacl, getfacl, xattr]
image: /assets/img/Linux_unix_bsd.png
---

The classic `rwx` permission model has three slots: owner, group, and other. That is fine until you need to give one extra user read access to a directory without adding them to a group, or you need to tag a file with metadata the filesystem does not natively understand. That is where POSIX ACLs and extended attributes come in.

## Checking ACL Support

```bash
# Is the filesystem mounted with ACL support?
findmnt -no OPTIONS /

# Show the ACL on a file
getfacl /var/www/html/index.html

# Show the ACL on a directory
getfacl /srv/data
```

*Tip:* Most modern filesystems (ext4, XFS, btrfs) enable ACLs by default. If `getfacl` reports no ACL, check the mount options for `noacl`.

## Reading ACLs

```bash
# Standard output
getfacl /srv/data

# Without the comment header
getfacl -c /srv/data

# Recursive
getfacl -R /srv/data

# Show only the ACL entries
getfacl -c -p /srv/data
```

*Tip:* `getfacl` output includes a `mask` line. The mask is the maximum effective permission for named users and groups, and it is the most common reason an ACL appears to be ignored.

## Setting ACLs

```bash
# Give a specific user read access
setfacl -m u:alice:r /srv/data

# Give a specific user read-write access
setfacl -m u:alice:rw /srv/data

# Give a specific group read access
setfacl -m g:devops:r /srv/data

# Set a default ACL for new files in a directory
setfacl -d -m u:alice:rw /srv/data

# Remove a specific entry
setfacl -x u:alice /srv/data

# Remove all ACL entries
setfacl -b /srv/data

# Apply recursively
setfacl -R -m u:alice:rw /srv/data
```

*Tip:* Default ACLs (`-d`) only apply to directories, and they are inherited by new files and subdirectories. They are the cleanest way to make a shared directory behave the way users expect.

## Copying and Backing Up ACLs

```bash
# Copy a file with its ACLs
cp --preserve=all source dest

# Copy a directory tree with ACLs
rsync -avAX source/ dest/

# Archive with ACLs
tar --acls --xattrs -czvf archive.tar.gz /srv/data
```

*Tip:* `rsync -A` preserves ACLs and `-X` preserves extended attributes. Without them, a restore silently loses every ACL you set.

## Extended Attributes

Extended attributes (xattrs) attach arbitrary key-value metadata to a file. They are used by SELinux, by `capabilities`, and by many backup and container tools.

```bash
# List xattrs on a file
getfattr -d /srv/data/file.txt

# List xattrs including SELinux context
getfattr -m - -d /srv/data/file.txt

# Set an xattr
setfattr -n user.comment -v "reviewed 2026-01-01" /srv/data/file.txt

# Remove an xattr
setfattr -x user.comment /srv/data/file.txt
```

*Tip:* The `user.` namespace is available to unprivileged users. The `security.`, `trusted.`, and `system.` namespaces require root, and `security.selinux` is managed by SELinux, not by `setfattr`.

## File Capabilities

Capabilities are a special xattr that grants a process a subset of root's powers without running as root.

```bash
# Show capabilities
getcap /usr/bin/ping

# Give a binary the ability to bind to low ports
sudo setcap cap_net_bind_service=+ep /usr/local/bin/myapp

# Remove capabilities
sudo setcap -r /usr/local/bin/myapp
```

*Tip:* `cap_net_bind_service` is the standard replacement for running a web server as root just to bind port 80. It is far safer than `setuid root`.

## Common Pitfalls

```bash
# ACLs are ignored if the mask is too restrictive
getfacl /srv/data | grep mask

# Fix the mask
setfacl -m m::rwx /srv/data

# Check whether the filesystem supports xattrs
touch /tmp/test && setfattr -n user.test -v 1 /tmp/test && getfattr -d /tmp/test
```

*Tip:* If `setfacl` succeeds but the user still cannot access the file, check the mask first. It is the single most common ACL problem.

## Quick Reference

```bash
# Show ACLs
getfacl /srv/data

# Give a user access
setfacl -m u:alice:rw /srv/data

# Default ACL for new files
setfacl -d -m u:alice:rw /srv/data

# Show xattrs
getfattr -d file.txt

# Grant a capability
sudo setcap cap_net_bind_service=+ep /usr/local/bin/myapp
```

*Tip:* Work in order: check the ACL, check the mask, check the mount options. Most "ACL is not working" incidents are a restrictive mask or a filesystem mounted with `noacl`.
