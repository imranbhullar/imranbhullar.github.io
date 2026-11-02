---
layout: post
title: "Portainer: A Web UI for Docker and Container Management"
date: Jan-11-2026
categories: [Linux, DevOps]
tags: [linux, docker, portainer, containers, devops]
image: /assets/img/portainer-docker-containers-dashboard.jpg
---

`docker ps`, `docker logs -f`, and `docker exec -it` cover most day-to-day container work, but juggling a dozen containers, several stacks, and more than one host over SSH gets tedious fast. Portainer wraps the Docker API in a web dashboard: starting, stopping, and inspecting containers, managing volumes and networks, and deploying multi-container stacks, all without memorizing another flag.

## Installing Portainer as a Container

Portainer runs as a container itself, managing Docker by talking to the Docker socket, which is why that socket has to be mounted in.

```bash
docker volume create portainer_data

docker run -d \
  -p 8000:8000 -p 9443:9443 \
  --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

*Tip:* Mounting `/var/run/docker.sock` gives Portainer full control over the Docker daemon, equivalent to root on the host. Only run Portainer on trusted networks, and lock down who gets an account in it.

## First Login and Environment Setup

On first access, Portainer prompts you to create an admin account and connect an environment — the Docker host it'll manage.

```bash
# Access the UI
https://<server-ip>:9443

# Create the initial admin user when prompted (do this immediately;
# leaving it unset risks someone else claiming it on a shared network)

# Then select "Get Started" to connect the local Docker environment
# that Portainer is already running on, via the mounted socket
```

*Tip:* Portainer locks the setup wizard after a short timeout if no admin account is created, as a safeguard against someone else claiming it first. If you miss the window, restart the container to get a fresh one.

## The Container List and Quick Actions

The Containers view replaces a chunk of your `docker ps`/`docker start`/`docker stop` workflow with clickable actions.

```bash
# What Portainer's container list is showing you is equivalent to:
docker ps -a
docker stats --no-stream

# Actions available per-container in the UI:
# start / stop / restart / pause / kill / remove, console access, and log viewing
```

*Tip:* The "Quick actions" column lets you jump straight into a container's shell or logs without typing `docker exec -it <container> sh` or `docker logs -f <container>` — genuinely faster once you're managing more than two or three containers.

## Deploying Stacks with Compose

Portainer's Stacks feature takes a docker-compose.yml and deploys it, giving you a web-based alternative to running `docker compose up` over SSH every time.

```yaml
# Paste directly into Portainer's Stacks > Add Stack editor
version: "3"
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
  redis:
    image: redis:alpine
```

```bash
# The CLI equivalent of what "Deploy the stack" is doing behind the scenes
docker compose -f stack.yml up -d
```

*Tip:* Stacks can also pull a compose file directly from a Git repository, which turns a deploy into "push to main, redeploy in Portainer" instead of copy-pasting YAML into a text box each time.

## Managing Volumes, Networks, and Images

The same inspection and cleanup work you'd do with `docker volume`/`docker network`/`docker image` subcommands is one click away, with usage shown per resource.

```bash
# What these Portainer sections correspond to on the CLI
docker volume ls
docker network ls
docker image ls

# Finding and removing genuinely unused resources
docker system df
docker system prune -a --volumes
```

*Tip:* Portainer's volume and image lists show you what's actually in use by a running container versus what's orphaned, which is the one piece of information `docker volume ls` alone doesn't give you without cross-referencing `docker inspect`.

## Live Container Logs and Console Access

Both log streaming and an interactive shell into a running container are available straight from the browser, no SSH session required.

```bash
# Equivalent CLI commands for what the UI buttons trigger
docker logs -f --tail 200 <container>
docker exec -it <container> /bin/sh
```

*Tip:* The in-browser console is handy for a quick look, but it's still just `exec` into the container's own shell — if the image is `FROM scratch` or otherwise has no shell, the console button will fail the same way `docker exec` would.

## Role-Based Access Control

Teams managing shared infrastructure need more than one shared admin login; Portainer Business and the free Community Edition both support creating additional users with scoped permissions.

```bash
# In the UI: Users > Add user, then assign a role
# Roles include: Administrator, Helpdesk (read-only operations),
# Standard User (restricted to specific environments/stacks),
# and custom roles with granular permissions
```

*Tip:* Give most team members a Standard User role scoped to only the stacks they actually own. A shared admin login with no audit trail is exactly the setup that makes "who restarted the production container at 2am" unanswerable later.

## Managing Multiple Docker Hosts from One Portainer

Portainer's Environments feature lets a single Portainer instance manage several remote Docker hosts or Swarm clusters, not just the one it's running on.

```bash
# On the remote host, run the Portainer agent so the main
# Portainer instance can reach it
docker run -d \
  -p 9001:9001 --name portainer_agent --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Then in the main Portainer UI: Environments > Add environment > Agent,
# pointing at <remote-host-ip>:9001
```

*Tip:* Running the lightweight agent on remote hosts, rather than exposing the Docker socket directly over the network, keeps the attack surface smaller — the agent only talks to Portainer, not to anyone who can reach the host's network.
