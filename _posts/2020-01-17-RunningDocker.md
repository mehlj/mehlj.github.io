---
layout: post
title: Docker Quick Start
published: true
category: devops
---

As explained in [another post](https://mehlj.github.io/Docker/), Docker is software designed to 'containerize' applications. While it can be complex to implement
on a larger scale, it is quite easy to quickly launch and test on a single machine.

## Environment
Containers are platform-independent - a major boost to portability in containerized applications. 

However, to exactly replicate these steps, here are the specifications: 
- CentOS 8
- `docker-ce-3:19.03.5-3.el7.x86_64` 

These steps will largely be adapted from the official CentOS/Docker documentation:
https://docs.docker.com/install/linux/docker-ce/centos/

## Prerequisites
Begin by installing some utilities needed to setup our `docker` `yum` repository, and
some drivers needed by the `docker` engine.
```
[mehlj@docker ~]$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
Last metadata expiration check: 0:11:58 ago on Tue 21 Jan 2020 07:45:35 PM EST.
Package device-mapper-persistent-data-0.8.5-2.el8.x86_64 is already installed.
Package lvm2-8:2.03.05-5.el8.0.1.x86_64 is already installed.
Dependencies resolved.
==========================================================================================================================================================================================================================================================================================
 Package                                                              Architecture                                                      Version                                                                   Repository                                                         Size
==========================================================================================================================================================================================================================================================================================
Installing:
 yum-utils                                                            noarch                                                            4.0.8-3.el8                                                               BaseOS                                                             64 k

Transaction Summary
==========================================================================================================================================================================================================================================================================================
Install  1 Package

Total download size: 64 k
Installed size: 19 k
Downloading Packages:
yum-utils-4.0.8-3.el8.noarch.rpm                                                                                                                                                                                                                          399 kB/s |  64 kB     00:00    
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                                                                     154 kB/s |  64 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                                                                  1/1 
  Installing       : yum-utils-4.0.8-3.el8.noarch                                                                                                                                                                                                                                     1/1 
  Running scriptlet: yum-utils-4.0.8-3.el8.noarch                                                                                                                                                                                                                                     1/1 
  Verifying        : yum-utils-4.0.8-3.el8.noarch                                                                                                                                                                                                                                     1/1 

Installed:
  yum-utils-4.0.8-3.el8.noarch                                                                                                                                                                                                                                                            

Complete!
[mehlj@docker ~]$ 
```

Add the stable repository for `docker` to our machine:
```
[mehlj@docker ~]$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
[mehlj@docker ~]$ 
```

Install the `docker` engine and `containerd`. Note that the `--nobest` flag must be passed, as there was, at the time 
of writing this, a dependency issue with CentOS 8 and `containerd`.
```
[mehlj@docker ~]$ sudo yum install docker-ce docker-ce-cli containerd.io --nobest -y
Last metadata expiration check: 0:00:27 ago on Tue 21 Jan 2020 08:14:11 PM EST.
Dependencies resolved.

 Problem: package docker-ce-3:19.03.5-3.el7.x86_64 requires containerd.io >= 1.2.2-3, but none of the providers can be installed
  - cannot install the best candidate for the job
  - package containerd.io-1.2.10-3.2.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.3.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.el7.x86_64 is excluded
  - package containerd.io-1.2.4-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.5-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.6-3.3.el7.x86_64 is excluded
==========================================================================================================================================================================================================================================================================================
 Package                                                              Architecture                                                  Version                                                                 Repository                                                               Size
==========================================================================================================================================================================================================================================================================================
Installing:
 containerd.io                                                        x86_64                                                        1.2.0-3.el7                                                             docker-ce-stable                                                         22 M
 docker-ce                                                            x86_64                                                        3:18.09.1-3.el7                                                         docker-ce-stable                                                         19 M
 docker-ce-cli                                                        x86_64                                                        1:19.03.5-3.el7                                                         docker-ce-stable                                                         39 M
Installing dependencies:
 libcgroup                                                            x86_64                                                        0.41-19.el8                                                             BaseOS                                                                   70 k
Skipping packages with broken dependencies:
 docker-ce                                                            x86_64                                                        3:19.03.5-3.el7                                                         docker-ce-stable                                                         24 M

Transaction Summary
==========================================================================================================================================================================================================================================================================================
Install  4 Packages
Skip     1 Package

Total size: 80 M
Installed size: 338 M
Downloading Packages:
[SKIPPED] libcgroup-0.41-19.el8.x86_64.rpm: Already downloaded                                                                                                                                                                                                                           
[SKIPPED] containerd.io-1.2.0-3.el7.x86_64.rpm: Already downloaded                                                                                                                                                                                                                       
[SKIPPED] docker-ce-18.09.1-3.el7.x86_64.rpm: Already downloaded                                                                                                                                                                                                                         
[SKIPPED] docker-ce-cli-19.03.5-3.el7.x86_64.rpm: Already downloaded                                                                                                                                                                                                                     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                                                                  1/1 
  Installing       : docker-ce-cli-1:19.03.5-3.el7.x86_64                                                                                                                                                                                                                             1/4 
  Running scriptlet: docker-ce-cli-1:19.03.5-3.el7.x86_64                                                                                                                                                                                                                             1/4 
  Installing       : containerd.io-1.2.0-3.el7.x86_64                                                                                                                                                                                                                                 2/4 
  Running scriptlet: containerd.io-1.2.0-3.el7.x86_64                                                                                                                                                                                                                                 2/4 
  Running scriptlet: libcgroup-0.41-19.el8.x86_64                                                                                                                                                                                                                                     3/4 
  Installing       : libcgroup-0.41-19.el8.x86_64                                                                                                                                                                                                                                     3/4 
  Running scriptlet: libcgroup-0.41-19.el8.x86_64                                                                                                                                                                                                                                     3/4 
  Running scriptlet: docker-ce-3:18.09.1-3.el7.x86_64                                                                                                                                                                                                                                 4/4 
  Installing       : docker-ce-3:18.09.1-3.el7.x86_64                                                                                                                                                                                                                                 4/4 
  Running scriptlet: docker-ce-3:18.09.1-3.el7.x86_64                                                                                                                                                                                                                                 4/4 
  Verifying        : libcgroup-0.41-19.el8.x86_64                                                                                                                                                                                                                                     1/4 
  Verifying        : containerd.io-1.2.0-3.el7.x86_64                                                                                                                                                                                                                                 2/4 
  Verifying        : docker-ce-3:18.09.1-3.el7.x86_64                                                                                                                                                                                                                                 3/4 
  Verifying        : docker-ce-cli-1:19.03.5-3.el7.x86_64                                                                                                                                                                                                                             4/4 

Installed:
  containerd.io-1.2.0-3.el7.x86_64                                      docker-ce-3:18.09.1-3.el7.x86_64                                      docker-ce-cli-1:19.03.5-3.el7.x86_64                                      libcgroup-0.41-19.el8.x86_64                                     

Skipped:
  docker-ce-3:19.03.5-3.el7.x86_64                                                                                                                                                                                                                                                        

Complete!
[mehlj@docker ~]$ 
```
Start the `docker` service:
```
[mehlj@docker ~]$ sudo service docker start
Redirecting to /bin/systemctl start docker.service
```

## Testing
Now that `docker` is installed, we can use a proof-of-concept image from Docker Hub known as `hello-world` to verify
our installation. Docker Hub is a public collection of pre-made image shared by the community. This `hello-world`
container is provided by the company Docker itself.

To run this container, we simple need to use `docker run`:
```
[mehlj@docker ~]$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:9572f7cdcee8591948c2963463447a53466950b3fc15a247fcad1917ca215a2f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

[mehlj@docker ~]$ 
```
As we notice from the output, `docker` first checks if the image was present on the local machine:

`Unable to find image 'hello-world:latest' locally`

After it determines that the image is not present locally, it will automatically download the image from Docker Hub.
Once it downloads, it runs in the `docker` engine.

As the output states, we can `try something more ambitious` by running a full Ubuntu container, instead of the simple 
`hello-world` container:
```
[mehlj@docker ~]$ sudo docker run -it ubuntu bash
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
5c939e3a4d10: Pull complete 
c63719cdbe7a: Pull complete 
19a861ea6baf: Pull complete 
651c9d2d6c4f: Pull complete 
Digest: sha256:8d31dad0c58f552e890d68bbfb735588b6b820a46e459672d96e585871acc110
Status: Downloaded newer image for ubuntu:latest

root@8efc1921398e:/# uname -a
Linux 8efc1921398e 4.18.0-147.3.1.el8_1.x86_64 #1 SMP Fri Jan 3 23:55:26 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
root@8efc1921398e:/# 
```
The container was created from the image simply known as `ubuntu`, and our flags `-it` and our call to `bash` allowed 
us to enter an interactive terminal session with it. 

From this, we can determine that our Docker installation is working as intended, and this system is ready for 
more practical usage of Docker.