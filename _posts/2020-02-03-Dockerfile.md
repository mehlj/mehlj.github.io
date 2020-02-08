---
layout: post
title: Anatomy of a Dockerfile
published: true
category: devops
---

As explained in [another post](https://mehlj.github.io/Docker/), a `Dockerfile` contains instructions on how to build 
your unique Docker container. It is a simple text file that is easily interpreted by both humans and the Docker engine. 

A `Dockerfile` is then _built_ into a Docker _image_ using `docker build`. A Docker image is then _run_, and a running
instance of a Docker image is referred to as a Docker _container_. You can have many Docker images stored locally, 
from the Internet, or in private repositories. But they all were once built from a `Dockerfile`. 

![](/images/dockerfile_diag.png)

Think of the `Dockerfile` as instructions for a construction worker. The worker reads the instructions, 
follows them step by step, and produces a final product in a package, ready to be used at the discretion of the 
customer. This final product is the Docker image.

## Example
```
FROM alpine:latest

# Update and install vim
RUN apk update && \
    apk add vim

# Create a new user
RUN adduser -S test_user

# Copy a file from host -> container
COPY testfile testfile

# Create a volume
RUN mkdir -p /data/hostvolume
RUN echo "Persistent data!" > /data/hostvolume/data_from_container
VOLUME /data/hostvolume

# Expose a network port
EXPOSE 80
```

### FROM
```
FROM alpine:latest
```
The `FROM` instruction defines a base image and version to create this new image from. 

In this instance, Docker is instructed to build from the official `alpine` base image. `alpine` is a lightweight Linux 
distribution (4-5 MB) that is commonly used in Docker container due to its speed and security. The `latest` keyword always ensures 
usage of the latest version of `alpine`.   

Docker images can be layered upon different images. This feature is useful for eliminating double-work in many situations. For example, if you needed a web server, you could use the official `debian` image available on Docker Hub. You could then add an install directive for the `apache` package somewhere later in your `Dockerfile`. In this scenario, you have saved a lot of time, as creating an operating system Dockerfile from scratch is non-trivial. 

Even better, you could simply use the official `apache` image.

### RUN
The `RUN` instruction performs a task, run by the container shell, in the background. This is the instruction
you should use if you want tasks to be performed at the OS level, but want output to be obstructed from users.

In this instance, Docker is instructed to update `alpine` using `apk`, install the `vim` package, and 
create a new system user named `test_user`. 

### COPY
```
COPY testfile testfile
```
The `COPY` instruction can be difficult to read, but is simple in actuality. It copies a file from the `source` 
to the `destination` (`COPY source destination`). The source is a directory/file on the host operating system, and the 
destination is a directory/file on the Docker container itself.

In this scenario, a simple text file was generated from the Docker host:
```
[mehlj@docker dockerfile]$ echo "File from host" >> testfile
```

The file `testfile` is copied from the host operating system to a file with the same name on the Docker container.

### VOLUME
```
RUN mkdir -p /data/hostvolume
RUN echo "Persistent data!" > /data/hostvolume/data_from_container
VOLUME /data/hostvolume
```
The `VOLUME` instruction defines where to create a Docker volume. Docker volumes are the solution to an issue that 
most users run into - if Docker containers are volatile and all data on the container file system is deleted 
after the container dies, then where can I store state/permanent data?
 
A Docker volume, in its most simple form, is shared disk space between the host operating system and the container.
The `VOLUME` instruction tells Docker where on the container to share files/directories. 

In this instance, the `/data/hostvolume/` directory is created on the container, a file is created, 
and then shared to the host operating system. The file `data_from_container` will not be lost when the container
is inevitably destroyed. 

### EXPOSE
```
EXPOSE 80
```
The `EXPOSE` instruction defines the network ports that the Docker container should listen on when run. By default, no
ports are exposed to others, even if services such as `httpd` are enabled and running. This is a necessary instruction
when utilizing Docker containers for network services.

In this scenario, the container is configured to listen on port 80.

## Building
Docker images are built from a `Dockerfile`. They are built using the `docker build` command:
```
$ docker build -t <my_tag_name> <directory containing Dockerfile>  
```
The `-t` flag allows the user to intelligently name the image, and potentially tag it with a version for tracking 
purposes.

The `<directory containing Dockerfile>` positional argument is often `.` - as this is shorthand for the `current
working directory`. This is often the location of your `Dockerfile`.

We can build our custom `Dockerfile` in the same way:

```
[mehlj@docker dockerfile]$ docker build -t my_alpine_image .
Sending build context to Docker daemon  3.584kB
Step 1/8 : FROM alpine:latest
 ---> e7d92cdc71fe
Step 2/8 : RUN apk update &&     apk add vim
 ---> Running in de8a90cfdee1
fetch http://dl-cdn.alpinelinux.org/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
v3.11.3-55-g1fe0a66db9 [http://dl-cdn.alpinelinux.org/alpine/v3.11/main]
v3.11.3-33-gadd23db31f [http://dl-cdn.alpinelinux.org/alpine/v3.11/community]
OK: 11262 distinct packages available
(1/6) Installing xxd (8.2.0-r0)
(2/6) Installing lua5.3-libs (5.3.5-r2)
(3/6) Installing ncurses-terminfo-base (6.1_p20191130-r0)
(4/6) Installing ncurses-terminfo (6.1_p20191130-r0)
(5/6) Installing ncurses-libs (6.1_p20191130-r0)
(6/6) Installing vim (8.2.0-r0)
Executing busybox-1.31.1-r9.trigger
OK: 42 MiB in 20 packages
Removing intermediate container de8a90cfdee1
 ---> c253c3f57b54
Step 3/8 : RUN adduser -S test_user
 ---> Running in 63b6750a9d45
Removing intermediate container 63b6750a9d45
 ---> e5aac9bcb95a
Step 4/8 : COPY testfile testfile
 ---> 2c6f071f0304
Step 5/8 : RUN mkdir -p /data/hostvolume
 ---> Running in ac743f22a4e1
Removing intermediate container ac743f22a4e1
 ---> 1fabeea57b79
Step 6/8 : RUN echo "Persistent data!" > /data/hostvolume/data_from_container
 ---> Running in 3958438685c9
Removing intermediate container 3958438685c9
 ---> 95f0f82333a4
Step 7/8 : VOLUME /data/hostvolume
 ---> Running in 168f2161b4b6
Removing intermediate container 168f2161b4b6
 ---> 4798beac2466
Step 8/8 : EXPOSE 80
 ---> Running in 91e256740543
Removing intermediate container 91e256740543
 ---> 308068a42f76
Successfully built 308068a42f76
Successfully tagged my_alpine_image:latest
```
After the build completes, we should have a completed Docker image saved to our host operating system.
We can confirm this using `docker image ls`:
```
[mehlj@docker dockerfile]$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
my_alpine_image     latest              a5389d833441        7 minutes ago       35.9MB
```

## Running
A Docker container is a running instance of a Docker image. To run an image, we can use `docker run`:
```
[mehlj@docker dockerfile]$ docker run -it my_alpine_image /bin/sh                                                                                                                                                                                                                        
```
The flags `-it` and positional argument `/bin/sh` are a common combination that allows you to enter an interactive
shell with your container. We will be doing this for educational purposes. 

Once in our container shell, we can confirm that several steps of the Dockerfile were completed during the 
build/runtime:
```
/ # vim --version | head -n 1
VIM - Vi IMproved 8.2 (2019 Dec 12, compiled Dec 12 2019 19:30:49)
```
```
/ # grep user /etc/passwd
test_user:x:100:65533:Linux User,,,:/home/test_user:/sbin/nologin
```
```
/ # cat testfile 
File from host
```

```
/ # cat /data/hostvolume/data_from_container 
Persistent data!
/ # 
```