---
layout: post
title: Project - On-Premises Kubernetes - Overview
published: true
category: infrastructure
---
# Links
https://github.com/users/mehlj/projects/2

https://github.com/mehlj/mehlj-packer

https://github.com/mehlj/mehlj-terraform

https://github.com/mehlj/mehlj-ansible

https://github.com/mehlj/mehlj-pipeline

# Background
Kubernetes seems to have won the container orchestration competition, and [even Mirantis agrees](https://www.mirantis.com/blog/mirantis-acquires-docker-enterprise-platform-business/). 

Kubernetes may have added complexity when compared to Swarm, but, if utilized properly alongside a well-engineered microservice software architecture, it can add significant business value.

As such, I wanted a project to test out on-premise Kubernetes cluster bootstrapping, as some work scenarios may not merit cloud deployments.

# Goal
An automated, secure, and code-defined pipeline(s) that, when run, provisions virtual machines, bootstraps a production-ready Kubernetes cluster, and deploys a simple REST API behind a clustered reverse proxy. 

# Strategy
## Tools
### vSphere
I will be leveraging my existing vSphere cluster in my home lab, licensed with vSphere Enterprise and vCenter through [VMUG Advantage](https://www.vmug.com/).

### Infrastructure as Code
`terraform` will be used for all infrastructure provisioning - in this case, three Kubernetes cluster nodes.

### Templating
The vSphere provider for `terraform` generally requires a vSphere template to clone from, so `packer` will be used to provision and apply configuration management to a CentOS template.

### Configuration
`ansible` will be used for all configuration management. CM code should be applied to baseline images only, to promote the principles of immutable infrastructure.

### Kubernetes
[`kubespray`](https://github.com/kubernetes-sigs/kubespray) will be used for Kubernetes cluster bootstrapping. It is better to leverage tried and true methodologies for complex tasks such as this.

### REST API
The REST API will be written in `go`, with the appropriate `Dockerfile` to facilitate `docker` image creation.

### Version Control
All code or configuration files will be stored in GitHub repositories. These repositories should be made public, as there will be no secrets stored in the code itself.

### CI/CD
Since all code will be stored in GitHub - I will leverage GitHub Actions for all CI pipelines.

## Separation
Generally, I will try to promote logical separation of three major tenants in tech environments:
1. Infrastructure
2. Configuration
3. Applications

As such, I will keep these elements independent of each other, while still employing CI pipelines to bridge those gaps and automate deployment start to finish.


# Overview
![](/images/clusterflow.png)

