---
layout: post
title: Project - On-Premises Kubernetes - Infrastructure
published: true
category: infrastructure
---
# Links
[Source code - Packer](https://github.com/mehlj/mehlj-packer)

[Source code - Terraform](https://github.com/mehlj/mehlj-terraform)

[Source code - Ansible](https://github.com/mehlj/mehlj-ansible)


# Background
In this project, I will be deploying several clustered instances of a simple REST API written in Golang. To accomplish this, I will containerize the application and deploy it into an on-prem Kubernetes cluster.

There are two major components of the underlying infrastructure for this application - the Kubernetes cluster itself, and the worker and control plane nodes that comprise the cluster.

Just as we are defining the application and application configuration as code, we will define these two major infrastructure components entirely as code, and leverage automation to provision them.

# Goal
An automated, secure, and code-defined pipeline(s) that, when run, provisions virtual machines and bootstraps a production-ready Kubernetes cluster.

# Implementation
## Templating
### Background
In theory, servers could be deployed from scratch (fresh, new install) every time you need to deploy your infrastructure or scale. This could be done by hand, or via scripts.

That, of course, would mean a lot of wasted time. Since that OS install process is the same for every server you deploy, shouldn't you just do that time-costly process once? 

The solution is through the usage of templating techniques, or creating *machine images*. 

Machine image software allows you to define a *template* as code. You define the virtual machine specifications (CPU, RAM, etc.), operating system, and supply some basic configuration tasks to perform once the OS is installed (grab a DHCP address, NFS mounts, etc.).

Once you define you template entirely as code, the machine image software *builds* the template and stores it in your specified location.

Once the template is created and present in the intended location, then servers can be deployed *from* the template ad infitium. This saves a huge amount of time, since servers deployed from the template will already have an OS and an expected configuration.

### Packer
[Hashicorp Packer](https://www.packer.io/) is one such machine image software, and the most popular. 

It allows you to define several types of templates via HCL2 or JSON, and *creates* the templates using the binary CLI - i.e., `packer build my-code.pkr.hcl`.

In my project situation, I need x3 CentOS virtual machines to comprise the Kubernetes cluster, the ability to define them via code, and the ability to deploy them via automation.

To that end, I used Packer HCL2 to define a single CentOS template with standard Kubernetes node resource specifications. This template would be ultimately stored in my on-premise vSphere cluster.

```H
source "vsphere-iso" "k8stemplate" {
  CPUs                 = "${var.vm-cpu-num}"
  RAM                  = "${var.vm-mem-size}"
  RAM_reserve_all      = false
  boot_command         = ["<tab> text ks=https://raw.githubusercontent.com/mehlj/mehlj-packer/master/ks.cfg<enter><wait>"]
  boot_order           = "disk,cdrom"
  boot_wait            = "10s"
  convert_to_template  = true
  datastore            = "${var.vsphere-datastore}"
  guest_os_type        = "centos8_64Guest"
  host                 = "${var.host}"
  insecure_connection  = "true"
  iso_paths            = ["${var.iso_url}"]
  network_adapters {
    network      = "${var.vsphere-network}"
    network_card = "vmxnet3"
  }
  notes        = "Kubernetes cluster node template. Built via Packer."
  password     = "${local.admin-password}"
  ssh_password = "${local.ssh-password}"
  ssh_username = "mehlj"
  storage {
    disk_size             = "${var.vm-disk-size}"
    disk_thin_provisioned = true
  }
  username       = "${var.vsphere-user}"
  vcenter_server = "${var.vsphere-server}"
  vm_name        = "${var.vm-name}"
}
```

### Kickstart
In my above example, I define a virtual machine as code that will ultimately become a vSphere template. 

However, how do we handle the installation of the operating system itself? CentOS usually installs via an interactive wizard after booting to an install disc.

To avoid any manual intervention, we can leverage the Red Hat product *Kickstart*. Kickstart, similarly, allows you to define a RHEL or CentOS OS installation process entirely as code. You need only supply a `ks.cfg` file as a kernel boot parameter.

How do we supply that file as a boot parameter after we boot to a fresh CentOS installation ISO? That is where `boot_command` comes in. That Packer variable determines what keystrokes to pass to the virtual machine as soon as it boots.

In my above example:
```H
boot_command         = ["<tab> text ks=https://raw.githubusercontent.com/mehlj/mehlj-packer/master/ks.cfg<enter><wait>"]
```

The virtual machine types in my Kickstart boot parameter as raw text, presses *Enter* to boot, and *Waits* for the rest of the installation.

Kickstart is capable of pulling in configuration files via HTTP as well, so I pull directly from the GitHub repo URL to maintain accuracy over time.

### Ansible
After the operating system installs automatically (thanks to Kickstart), we are left with a fresh CentOS virtual machine that is about to be converted to a template.

What if we don't want our template to be completely bare-bones? What if we have a common configuration that should be applied to ALL machines in our fleet? It would be a waste of time to apply that configuration each time we instantiate from that template, right?

We should aim to apply fleet-common configuration only once - to our baseline template. For any non-fleet common configuration, i.e., unique configuration that must be done to that one template instantiation, we rely on the IaC tooling later down the process.

Examples of fleet-common configuration could be installing your corporate anti-virus agent, OS hardening, HTTP proxy settings, etc. *Any configuration that should be common across your entire fleet*.

Applying this fleet-common configuration is theoretically possible with pure Packer, as it allows for simple shell commands to be run on your template:
```H
  provisioner "shell" {
    inline = [
      "echo 'it is alive !'"
    ]
  }
```

However, this is a sub-standard way of defining configuration-as-code (CaC). It is better to leverage a configuration management tool like [Ansible](https://www.ansible.com/).

In my case, I've written all my Ansible code in [another repo](https://github.com/mehlj/mehlj-ansible). To chain together Packer and Ansible, you can leverage the [Ansible provisioner](https://www.packer.io/plugins/provisioners/ansible/ansible).

```H
build {
  sources = ["source.vsphere-iso.k8stemplate"]

  # stage the ansible vault pass file for later usage
  provisioner "file" {
    content         = "${local.vault-password}"
    destination     = "/root/.vault_pass.txt"
  }

  # configure SSH keys
  provisioner "ansible" {
    command         = "./ansible-wrapper.sh"
    playbook_file   = "./mehlj-ansible/playbooks/ssh.yml"
  }

  # configure kubernetes prerequisites
  provisioner "ansible" {
    command         = "./ansible-wrapper.sh"
    playbook_file   = "./mehlj-ansible/playbooks/kubernetes.yml"
    extra_arguments = ["--vault-password-file=/root/.vault_pass.txt"]
  }
}
```

In the above code snippet, I leverage the Ansible provisioner to execute playbooks stored in another GitHub repository - locally extracted to `./mehlj-ansible/`.

How does the repository get extracted locally? I leverage a simple shell wrapper (`ansible-wrapper.sh`) around Ansible that clones the target repo before ever running the CLI, thus ensuring it's always there.
```bash
#!/bin/bash

git clone https://github.com/mehlj/mehlj-ansible.git && ansible-playbook "$@"
```

After Packer bootstraps Ansible and all my playbooks are run, the virtual machine now has proper SSH keys, Ansible vault passwords (for secret management), and Kubernetes node configuration requirements completed. 

It is now ready to be converted to a vSphere template.

### Pipeline
In order for Packer to actually *build* the template, you must call the binary CLI like so: `packer build my-code.pkr.hcl`

Normally, in a git repository like this, you would define a CI pipeline that performs basic linting of the HCL2 files, and upon success, runs the above command to create the template.

In my case, this template is extremely static, so I ran the command manually once. In production scenarios, a CI pipeline that runs automatically on commits may be necessary.

## Infrastructure as Code
### Background
Now that our vSphere template is present in the cluster, and is at an expected configuration, we can begin instantiating servers from said template.

We can always instantiate these servers manually via the web interface, but that would introduce a manual intervention into the overarching workflow, bringing automation to a halt.

Additionally, we may be scaling or re-deploying this infrastructure many times over. Wouldn't it be helpful to define this process ahead of time, so that the same methods of work are performed to deploy these server instantiations?

Of course, we could define the process via a static, hand-written document. When we scale or rebuild our infrastructure, we could refer back to the documentation and deploy the infrastructure in the same manner each time.

However, we all know that humans are not perfect and make mistakes. Especially when it comes to rote, repeated work such as this.

Additionally, documentation of this nature tends not to be scalable. What if you are scaling multiple times in a day? Or when necessary changes to the deployment methodology are found on the fly? Your documentation quickly becomes out of date.

When your pre-defined infrastructure configuration does not match what is actually deployed in real-time, you have encountered *configuration drift*.

When your infrastructure drifts, your servers morph into a [snowflake configuration](https://www.sumologic.com/blog/snowflake-configurations-and-devops-automation/) and become entirely impossible to reproduce. 

When your infrastructure becomes impossible to reproduce, rebuilding to resolve issues becomes impossible. 

When your infrastructure becomes impossible to reproduce, scaling becomes impossible.

When your infrastructure becomes impossible to reproduce, somewhere, a baby cries.

The answer to all of these problems is Infrastructure as Code (IaC). 

IaC tooling takes many forms, but they all share the same goal - to define your infrastructure in a cohesive manner, and track lifecycles accordingly.

### Terraform
Since we want to keep this workflow entirely automated, and for [all the other benefits](https://www.redhat.com/en/topics/automation/what-is-infrastructure-as-code-iac), we should leverage Infrastructure as Code (IaC) tooling.

[Terraform](https://www.terraform.io/) is the most popular IaC tools in the market today, and complements nicely with Packer since they are both made by Hashicorp.

It allows you to define your infrastructure as idempotent, human-readable, declarative code. You simple write your own code in HCL, and the Terraform binary handles provisioning and state-management of the infrastructure.

As mentioned previously, for this project, I need x3 virtual machines provisioned/managed in my on-prem vSphere cluster. Through the use of Packer, I have a vSphere template available for use.

We can leverage Terraform to instantiate the template three times, with slightly differing specifications for each instantiation (hostname, IP, etc.). 

```H

```






### Pipeline

## Kubespray








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
