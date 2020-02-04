---
layout: post
title: Anatomy of a Dockerfile
published: true
category: devops
---

As explained in [another post](https://mehlj.github.io/Docker/), a `Dockerfile` contains instructions on how to build your 
unique Docker container. It is a simple text file that is easily interpreted by both humans and the Docker engine. 

## Overview

## Example
### FROM
```
FROM alpine
```
The `FROM` instruction defines a base image to create this new image from. 

Docker images can be layered upon different images. This feature is useful for eliminating double-work in many situations. For example, if you needed a web server, you could use the official `debian` image available on Docker Hub. You could then add an install directive for the `apache` package somewhere later in your `Dockerfile`. In this scenario, you have saved a lot of time, as creating an operating system Dockerfile from scratch is non-trivial. 

Even better, you could simply use the official `apache` image.


### WORKDIR
### RUN
### USER
### EXPOSE
### COPY
### VOLUME

## Building

## Running
