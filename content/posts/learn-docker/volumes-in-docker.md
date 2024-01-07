---
title: "Volumes in Docker"
summary: "How to use volumes in Docker."
date: "2023-12-26"
tags: ["Docker", "Volumes"]
series: ["Learning Docker"]
author: ["Sanyam Jain"]
draft: false
---

# Data in Docker Containers
- By default, any data that is stored inside a container is lost when the container is removed.
- We should never make modifications to the container's file system. Any modification done to get it to the state we want, should be done in the image itself.
- To persist data, you can mount a volume. There are two types of volumes:
    - **Bind Mounts**: Mounts a local folder to the container.
    - **Volumes**: Creates a volume that is managed by Docker Desktop VM.

# Bind Mounts
- Bind mounts are used to mount a local folder to the container.

## Example
- Run `docker run -it --rm --mount type=bind,source="$(pwd)"/my-data,destination=/my-data/ ubuntu:22.04` to mount the current directory to the container.

# Volume Mounts
- Volume mounts are used to create a volume that is managed by Docker Desktop VM.
- Docker recommends using volume mounts over bind mounts.
- Volumes are stored in `/var/lib/docker/volumes` on Linux and `C:\ProgramData\Docker\volumes` on Windows.

## Example
- Run `docker volume create my-volume` to create a volume.
- Run `docker run -it --rm --mount source=my-volume,destination=/my-data/ ubuntu:22.04` to attach the volume
- Run `docker volume inspect my-volume` to inspect the volume.