---
title: "Solaris Zones: Complete Step-by-Step Guide"
date: 2011-08-15 10:00:00 +1000
categories: [Unix, Solaris, Virtualization]
tags: [solaris, zones, virtualization, unix, sysadmin]
description: A complete step-by-step guide to Solaris Zones explaining concepts, lifecycle, and practical configuration with examples.
---
![nix*](/assets/img/Linux_unix_bsd.png)
## Introduction

Solaris Zones provide **operating system–level virtualization**, allowing multiple isolated environments to run on a single Solaris instance.  
Unlike traditional virtual machines, zones share the **same kernel**, making them extremely lightweight and highly efficient.

This technology later inspired modern container platforms such as Docker and Kubernetes.

---

## What Are Solaris Zones?

A **zone** is an isolated execution environment within Solaris.

Each zone behaves like an independent server with:

- Its own process tree
- Separate users and groups
- Dedicated network configuration
- Controlled resource usage

At the same time, all zones safely share the same Solaris kernel.

---

## Zone Types

### Global Zone

The Global Zone is the main operating system and has:

- Full access to hardware
- Permission to create and manage zones
- Responsibility for system-wide administration

Best practice is to **avoid running applications in the Global Zone**.

---

### Non-Global Zones

Non-Global Zones are isolated environments created inside the Global Zone.

They are typically used for:

- Application hosting
- Service isolation
- Multi-tenant environments

---

## Whole Root vs Sparse Root Zones

### Whole Root Zone

- Contains a full copy of the Solaris OS
- Uses more disk space
- Slower to deploy
- Rarely used in production

---

### Sparse Root Zone (Recommended)

- Shares most system files with the Global Zone
- Faster boot time
- Lower disk and memory usage
- Ideal for production workloads

---

## Zone Lifecycle States

During its lifetime, a zone moves through multiple states:

- **Configured** – Configuration exists but zone is not installed  
- **Incomplete** – Installation or removal in progress  
- **Installed** – Required packages installed  
- **Ready** – Kernel resources prepared  
- **Running** – Zone is active  
- **Down** – Zone is halted  

---

## Creating a Solaris Zone (Step-by-Step)

Below is a complete and practical workflow for creating a **Sparse Root Zone**.

---

### Step 1: Verify Global Zone

```bash
zonename
#This must return:
global
#Zones can only be created from the Global Zone.
```
### Step 2: Create Zone Directory
```bash 
mkdir /solzone
chmod 700 /solzone
```
This directory will store the zone’s file system and must be accessible only by root.
### Step 3: Start Zone Configuration
```bash
zonecfg -z solzone
```
### Step 4: Define Basic Zone Properties
If the zone does not exist, Solaris will prompt to create it.
```bash
create
set zonepath=/solzone/
set autoboot=true
```
zonepath defines where the zone will reside
autoboot=true ensures the zone starts automatically after system reboot

### Step 5: Configure Networking
```bash
add net
set address=192.168.56.200
set physical=e1000g1
end
```
This assigns a dedicated IP address to the zone using the specified physical interface.

### Step 6: Configure Memory Limits
```bash
add capped-memory
set physical=512m
set swap=512m
end
```
Resource limits prevent a single zone from exhausting system memory.
###  Step 7: Verify and Save Configuration

```bash
verify
commit
exit
```
At this stage, the zone is successfully configured.

### Step 8: Install the Zone
```bash
zoneadm -z solzone install
```
Solaris installs the required OS packages into the zone.
This step may take several minutes.

### Step 9: Prepare the Zone
```bash
zoneadm -z solzone ready
```
This prepares kernel-level resources required to start the zone.
### Step 10: Boot the Zone
```bash
zoneadm -z solzone boot
```
The zone transitions into the Running state.

### Step 11: Access the Zone Console
```bash
zlogin -C solzone
```
This opens the zone console for initial setup such as:

* Root password
* Timezone
* System configuration

##  Common Administration Commands 
### List All Zones
```bash
zoneadm list -v
```
### Halt a Zone
```bash
zoneadm -z solzone halt
```
Stops the zone safely.
### Reboot a Zone 
```bash
zoneadm -z solzone reboot
```
### Modify Zone Configuration 
```bash
zonecfg -z solzone
```
(Used to update memory, network, or other settings, the zone must be halted before modification.)
### Uninstall a Zone
```bash
zoneadm -z solzone uninstall
```
Removes installed packages but keeps the zone definition.
### Delete Zone Completely
```bash
zonecfg -z solzone delete
```
Permanently removes the zone configuration.

### Final Thoughts

Solaris Zones were far ahead of their time.

They provide:
* Near-native performance
* Strong isolation
* Excellent resource control
* Minimal overhead

Many modern container technologies (LXC, Docker etc.) follow the same core principles originally introduced by Solaris Zones and BSD Jails.
