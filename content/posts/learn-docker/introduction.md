---
title: "Docker Intro"
summary: "Basic introduction to Docker."
date: "2023-12-26"
tags: ["Docker", "Introduction","IDK"]
series: ["Learning Docker"]
author: ["Sanyam Jain"]
draft: false
---

# Why I want to learn Docker

# Motivation for containerization
- Instead of running a bunch of different scripts to install software, you can just run a single command to set up a container that already has the software you need.
- To deploy an application, you can just use the same "image" that you used to develop it.
- You can run multiple containers on the same machine, and they won't interfere with each other.
- You can easily share images with other developers, so they can use the same software as you.

# What is a container?
- Their website says that "A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings"
- A container is a running instance of an image.

# What is OCI?
- Open Container Initiative (OCI) is an open governance structure. 
- Early days, multiple players were doing the same thing in a slightly different way. They all came together and formed OCI to standardize the way containers are built and run.
- Runtime Specification, Image Specification, and Distribution Specification are the three specifications that OCI has defined.
- Docker is a member of OCI.

# Difference between Desktop Container Platforms and Container Runtime
- Will come back to this later. I also don't know the difference right now. <mark>**IDK**</mark>.

# What are orchestration tools?
- Orchestration tools are used to manage multiple containers.
- I only know this much about orchestration tools right now. Will come back to this later. <mark>**IDK**</mark>.

# Linux Building blocks

## Namespaces
- Namespaces are used to isolate resources.
- It basically wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource.
- For example, a process in a namespace can have the same file name as a process in another namespace, but they will be different files.

## Control Groups (cgroups)
- Control Groups are used to limit the amount of resources that a process can use.
- For example, you can limit the amount of CPU, memory, and disk space that a process can use.

## Union File Systems 
- Union File Systems are used to create layers of files.
- Allows files and directories of separate file systems, known as branches, to be transparently overlaid, forming a single coherent file system.
- For example, you can have a base layer that contains the operating system, and then you can have another layer that contains your application.=