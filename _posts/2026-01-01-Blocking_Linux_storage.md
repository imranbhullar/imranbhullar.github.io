---
layout: post
title: "Blocking Mass Storage in Linux"
date: 2011-08-22 10:00:00 -0000
description: "A step-by-step guide on how to disable USB mass storage devices in Linux by blacklisting the usb_storage module."
categories: [Linux, Security]
tags: [Tips and Tricks, Linux, SysAdmin]
---

## Disabling Mass Storage in Linux-Based Distros

To prevent USB storage devices from being automatically recognized and mounted on your Linux system, follow these steps:

### 1. Unmount existing devices
If a USB device is currently connected, unmount it first:

```bash
[root@centos ~]# umount /dev/sdb
```
Note: The device name (e.g., /dev/sdb) may vary depending on your system configuration.

2. Remove the USB storage module
Manually remove the driver currently loaded in the kernel:
```bash
[root@centos ~]# modprobe -r usb_storage
```

3. Blacklist the module
To make this change permanent so the module doesn't load on the next reboot or when a device is plugged in, edit the blacklist configuration file:

Open /etc/modprobe.d/blacklist.conf (or create it if it doesn't exist) and add the following line:
```bash
blacklist usb_storage
```
Save the file and exit.
Conclusion
Now, when a USB flash drive is connected, the hotplug scripts will not load the driver automatically, effectively blocking mass storage access on the machine.