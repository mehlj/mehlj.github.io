---
layout: post
title: Docker Swarm
published: true
category: devops
---

Docker containers are simple to manage when you have one container or even [multiple containers on one host](https://mehlj.github.io/DockerCompose/). 

But what if you have tens of containers that you have to manage? What if you had to host these containers on several Docker hosts?

That is where [container orchestration](https://www.redhat.com/en/topics/containers/what-is-container-orchestration) tools come in. Docker Swarm is the orchestrator native to the Docker toolset. 

## Use Case
You should consider using an orchestrator anytime:
* You run many containers at a time
* You intend to run many containers across several different Docker hosts
* You are concerned with the availability of the services provided by these containers

Docker Swarm and Kubernetes are both orchestrators and provide generally the same function, but Kubernetes is generally considered to be more robust, more feature rich, and more efficient at larger scale. The tradeoff is that Kubernetes is also considered to be more complex and difficult to manage. 

Docker Swarm is a good idea if you have a relatively small cluster of containers and hosts, and do not need the rich features that Kubernetes provides.

## Example
## Running
