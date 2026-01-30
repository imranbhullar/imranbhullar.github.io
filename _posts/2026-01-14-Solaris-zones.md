---
title: "Solaris Zones: Complete Step-by-Step Guide"
date: 2011-08-15 10:00:00 +1000
categories: [Unix, Solaris, Virtualization]
tags: [solaris, zones, virtualization, unix, sysadmin]
description: A complete step-by-step guide to Solaris Zones explaining concepts, lifecycle, and practical configuration with examples.
---

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
