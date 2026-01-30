---
layout: post
title: "IPS (Image Packaging System) in Solaris 10"
description: "A guide on achieving IPS-like package management functionality in Oracle Solaris 10 using OpenCSW and BlastWave to handle dependencies efficiently."
date: 2011-08-15 12:00:00 +0000
categories: [nix Howto's]
tags: [solaris, ips, opencsw, blastwave]
---

In Standard Solaris, IPS (Image Packaging System) is exclusively a feature of Oracle Solaris 11. While Oracle Solaris 10 provides some tools to achieve similar functionality, their use in production environments raises questions about reliability.

Historically, package installation in Solaris has been challenging. Fortunately, some solutions now provide IPS-like functionality for Solaris 10.

### 1. OpenCSW

To utilize IPS functionality with OpenCSW in Oracle Solaris 10, follow these steps:

1. **Download the package:**
   [http://mirror.opencsw.org/opencsw/current/i386/5.10/pkgutil-2.4,REV=2011.05.15-SunOS5.8-i386-CSW.pkg.gz](http://mirror.opencsw.org/opencsw/current/i386/5.10/pkgutil-2.4%2CREV%3D2011.05.15-SunOS5.8-i386-CSW.pkg.gz)

2. **Unzip the downloaded package:**
   ```bash
   gunzip pkgutil-2.4,REV=2011.05.15-SunOS5.8-i386-CSW.pkg.gz
   ```
2. **Unzip the downloaded package:**
```bash
pkgadd -d pkgutil-2.4,REV=2011.05.15-SunOS5.8-i386-CSW.pkg
```
