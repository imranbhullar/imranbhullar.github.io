---
layout: post
title: "Linux Containers from First Principles: Namespaces and cgroups"
date: May-12-2012
categories: [Linux, DevOps]
tags: [linux, containers, namespaces, cgroups, docker, podman]
image: /assets/img/Linux_unix_bsd.png
---

Containers are not a Linux feature. They are a combination of two existing kernel features — namespaces and cgroups — plus a filesystem. Understanding that makes Docker and Podman far less magical, and makes debugging them far easier.

## What a Container Actually Is

A container is a process (or a group of processes) that:

1. Has its own view of the system, provided by **namespaces**.
2. Has resource limits, provided by **cgroups**.
3. Runs from a **filesystem image** rather than the host root.

That is it. There is no container runtime in the kernel.

## Namespaces: Isolating the View

Each namespace type isolates one aspect of the system.

| Namespace | Isolates |
| :--- | :--- |
| `pid` | Process IDs |
| `net` | Network interfaces, routes, ports |
| `mnt` | Mount points |
| `uts` | Hostname and domain name |
| `ipc` | Inter-process communication |
| `user` | User and group IDs |
| `cgroup` | cgroup hierarchy view |

```bash
# List namespaces for a process
sudo lsns -p 1234

# Enter a container's network namespace
sudo nsenter -t 1234 -n ip addr

# Enter a container's mount namespace
sudo nsenter -t 1234 -m ls /
```

*Tip:* `nsenter` is the single most useful debugging tool for containers. It lets you run any command inside a container's namespaces without `docker exec`.

## cgroups: Limiting Resources

cgroups (control groups) limit and account for CPU, memory, I/O, and PIDs.

```bash
# Show the cgroup of a process
cat /proc/1234/cgroup

# Show memory limits for a cgroup
cat /sys/fs/cgroup/system.slice/memory.max

# Show current memory usage
cat /sys/fs/cgroup/system.slice/memory.current

# Show CPU limits
cat /sys/fs/cgroup/system.slice/cpu.max
```

*Tip:* cgroup v2 uses a unified hierarchy under `/sys/fs/cgroup/`. If you see `memory.limit_in_bytes` instead of `memory.max`, you are on cgroup v1.

## Creating a Container by Hand

You can build a minimal container with nothing but `unshare` and `chroot`.

```bash
# Create a new PID, mount, and UTS namespace, with a new hostname
sudo unshare --pid --mount --uts --fork --hostname mycontainer /bin/bash

# Inside the namespace:
mount -t proc proc /proc
ps aux
hostname
```

*Tip:* This is exactly what a container runtime does, plus cgroup setup and a filesystem image. Running it by hand once makes every container bug easier to reason about.

## Docker and Podman

Docker and Podman are front-ends that do the namespace, cgroup, and filesystem setup for you.

```bash
# Run a container
docker run -d --name web -p 8080:80 nginx

# Same with Podman
podman run -d --name web -p 8080:80 nginx

# List running containers
docker ps
podman ps

# Execute a command inside
docker exec -it web bash
podman exec -it web bash

# Inspect the container's namespaces
sudo lsns -p $(docker inspect -f '{{.State.Pid}}' web)
```

*Tip:* Podman is daemonless and rootless by default, which makes it a better fit for single-host deployments and CI. Docker is still the more common choice for developer workflows.

## Debugging Containers

```bash
# Container logs
docker logs web

# Container resource usage
docker stats

# Inspect the container's PID on the host
docker inspect -f '{{.State.Pid}}' web

# Enter the container's namespaces
sudo nsenter -t $(docker inspect -f '{{.State.Pid}}' web) -n -m -p

# Check cgroup limits
cat /sys/fs/cgroup/system.slice/docker-*.scope/memory.max
```

*Tip:* When a container is OOM-killed, `docker inspect` shows `OOMKilled: true`. The fix is usually a higher `--memory` limit or a memory leak in the application, not a Docker bug.

## Quick Reference

```bash
# What namespaces does this process have?
sudo lsns -p 1234

# Enter a container's namespaces
sudo nsenter -t 1234 -n -m -p

# What are the cgroup limits?
cat /proc/1234/cgroup

# Container resource usage
docker stats
```

*Tip:* Work in order: identify the process, inspect its namespaces, inspect its cgroups, then look at the application. Most "container is broken" incidents are a resource limit or a missing mount, not a runtime bug.
