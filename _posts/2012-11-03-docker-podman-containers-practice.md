---
layout: post
title: "Docker and Podman in Practice: Running Containers Day to Day"
date: Nov-03-2012
categories: [Linux, DevOps]
tags: [linux, docker, podman, containers, devops]
image: /assets/img/docker-podman-containers.jpg
---

Namespaces and cgroups are the kernel primitives that make containers possible, but almost nobody talks to them directly day to day. Docker and Podman are the layer on top that turns "isolated process with its own filesystem" into "pull an image and run it," and that's the layer you'll actually spend your time in.

## Docker vs Podman: Daemon vs Daemonless

Docker runs a privileged background daemon that owns every container; Podman has no daemon at all, launching containers as regular child processes, which is also what makes its rootless mode genuinely rootless rather than rootless-with-a-root-daemon-behind-it.

```bash
# Debian/Ubuntu: Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# Podman is in the default repos on most distributions
sudo apt install podman     # Debian/Ubuntu
sudo dnf install podman     # RHEL/CentOS/Fedora
```

*Tip:* Podman's command-line interface deliberately mirrors Docker's almost exactly. In most of the examples below, `docker` and `podman` are interchangeable; where they diverge, it's called out.

## Running Your First Container

A container is a process running from an image, isolated by namespaces, with its own filesystem view layered from that image.

```bash
# Run a container and see output immediately
docker run hello-world

# Run something interactive
docker run -it ubuntu:24.04 /bin/bash

# Run detached, in the background, exposing a port
docker run -d -p 8080:80 --name my-nginx nginx
```

*Tip:* `-p 8080:80` maps host port 8080 to the container's port 80. Get the order backwards and you'll be confused why `curl localhost:80` doesn't reach a container that's clearly running.

## Images: Pulling, Listing, and Building

An image is the read-only template a container is created from. You can pull a published one or build your own from a `Dockerfile`.

```bash
# Pull an image explicitly
docker pull postgres:16

# List images you have locally
docker images

# Remove one you no longer need
docker rmi postgres:16

# A minimal Dockerfile
cat <<'EOF' > Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
EOF

docker build -t myapp:latest .
```

*Tip:* Order `Dockerfile` instructions from least to most frequently changing. Copying `requirements.txt` and installing dependencies before copying the rest of the source means Docker's layer cache skips the slow dependency install on every rebuild where only your code changed.

## Managing Container Lifecycle

Once something is running, these are the commands for inspecting, entering, and cleaning it up.

```bash
# List running containers
docker ps

# Include stopped ones too
docker ps -a

# Follow a container's logs
docker logs -f my-nginx

# Get a shell inside a running container
docker exec -it my-nginx /bin/bash

# Stop, then remove
docker stop my-nginx
docker rm my-nginx

# Remove every stopped container, unused network, and dangling image in one shot
docker system prune
```

*Tip:* `docker exec` opens a shell in an already-running container's existing process namespace, which is different from `docker run -it`, which starts a brand new container. Mixing these up is a common source of "wait, why is my change gone" confusion.

## Volumes and Bind Mounts for Persistent Data

A container's writable layer disappears when the container is removed. Anything that needs to survive, a database's data directory, uploaded files, has to live outside that layer.

```bash
# Named volume, managed by Docker/Podman, good for most database use cases
docker volume create pgdata
docker run -d --name postgres -v pgdata:/var/lib/postgresql/data postgres:16

# Bind mount, for directly exposing a host directory into the container
docker run -d --name web -v /home/alice/site:/usr/share/nginx/html:ro nginx

# List and inspect volumes
docker volume ls
docker volume inspect pgdata
```

*Tip:* Use named volumes for data the container itself owns and manages, like a database's storage. Use bind mounts when you specifically need to edit files from the host side while the container runs, like local web development.

## Networking Between Containers

Containers on the same user-defined network can resolve each other by container name, which is what makes multi-container applications work without hardcoded IPs.

```bash
# Create a network
docker network create app-net

# Run two containers on it
docker run -d --name db --network app-net postgres:16
docker run -d --name web --network app-net -p 8080:80 myapp:latest

# From inside "web", "db" resolves to the database container's IP automatically
docker exec web ping -c 1 db
```

*Tip:* The default bridge network does not provide automatic name resolution between containers; a user-defined network like `app-net` does. Always create one explicitly rather than relying on the default.

## Compose for Multi-Container Applications

Running several related containers by hand, in the right order, with the right networks and volumes, doesn't scale past two or three. Compose describes the whole stack in one file.

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: example
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - db

volumes:
  pgdata:
```

```bash
# Bring the whole stack up
docker compose up -d

# Tear it down
docker compose down

# Podman's equivalent
podman-compose up -d
```

*Tip:* `depends_on` controls start order, not readiness. A database container can be "started" long before `postgres` is actually accepting connections, so an application that connects immediately on boot still needs its own retry logic.

## Resource Limits on Containers

Without limits, a single misbehaving container can consume all of a host's memory or CPU, exactly the problem namespaces alone don't solve on their own.

```bash
# Cap memory and CPU for a container
docker run -d --memory=512m --cpus=1.5 myapp:latest

# Check current usage against limits for running containers
docker stats
```

*Tip:* Set a `--memory` limit on anything running in production. Without one, a memory leak in a single container can trigger the host's OOM killer, which may just as easily pick an unrelated, healthy process to kill instead.

## Bonus: Rootless Podman and Why It Matters

Docker's daemon traditionally runs as root, which means anyone who can talk to it can, in practice, get root on the host. Podman's rootless mode runs containers as your own unprivileged user from the start.

```bash
# Run a container as your own user, no daemon, no root involved
podman run -d -p 8080:80 --name my-nginx nginx

# Confirm it's actually running without root privileges
podman info | grep -i rootless
```

*Tip:* Rootless Podman relies on user namespaces to map container "root" to an unprivileged host UID. It closes an entire class of container-escape-to-host-root vulnerabilities by design, not by configuration you have to remember to apply.
