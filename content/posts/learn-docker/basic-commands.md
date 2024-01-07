---
title: "Docker Basic Commands"
summary: "Basic introduction to Docker."
date: "2023-12-26"
tags: ["Docker", "Basics"]
series: ["Learning Docker"]
author: ["Sanyam Jain"]
draft: false
---

# Setup
- Install docker through the official website.
- Check if docker is installed by running `docker --version` in the terminal.
- Configure docker settings for memory and CPU usage.
- Run `docker run hello-world` to check if docker is working properly.

# Some starter containers
Sometimes, it is not always possible to install stuff locally on your system. Like specific databases. In this case, you can use docker to run a container that has the database installed. You can then connect to this container from your local system.
## Postgres
- Run `docker run --name postgres -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 postgres`
- This will run a postgres container with the name `postgres` and password `postgres` on port `5432`.
- Other options:
    - `-e POSTGRES_USER=postgres` to set the username to `postgres`.
    - `-e POSTGRES_DB=postgres` to set the database name to `postgres`.
    - `-v /path/to/local/folder:/var/lib/postgresql/data` to mount a local folder to the container. This is useful if you want to persist the data in the container.
    - `-d` to run the container in the background.
    - `-p 5432:5432` to map the port `5432` of the container to the port `5432` of the host machine.

## MongoDB
- Run `docker run --name some-mongo -d mongo`
- This will run a mongo container with the name `some-mongo`.
- Other options:
    - `-e MONGO_INITDB_ROOT_USERNAME=mongoadmin` to set the username to `mongoadmin`.
    - `-e MONGO_INITDB_ROOT_PASSWORD=secret` to set the password to `secret`.
    - `-v /path/to/local/folder:/data/db` to mount a local folder to the container. This is useful if you want to persist the data in the container.
    - `-d` to run the container in the background.
    - `-p 27017:27017` to map the port `27017` of the container to the port `27017` of the host machine.

# Other useful commands
- `docker ps` to list all running containers.
- `docker ps -a` to list all containers.
- `docker stop <container_name>` to stop a container.
- `docker rm <container_name>` to remove a container.
- `docker images` to list all images.
- `docker rmi <image_name>` to remove an image.
- `docker exec -it <container_name> bash` to run a bash shell inside a container.
- `docker logs <container_name>` to view the logs of a container.
- `docker inspect <container_name>` to view the details of a container.
- `docker inspect <container_name> | grep IPAddress` to view the IP address of a container.

# Note
- Any data that is stored inside a container is lost when the container is removed. To persist data, you can mount a local folder to the container.
- Containers are not meant to be used as VMs. They are meant to be used as a single process. If you want to run multiple processes, you should use multiple containers.
- Containers are not meant to be used as a permanent storage solution. If you want to store data permanently, you should use a database.