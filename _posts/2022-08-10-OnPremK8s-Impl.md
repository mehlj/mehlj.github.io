---
layout: post
title: Project - On-Premises Kubernetes - Implementation
published: true
category: infrastructure
---
# Links
[Source code - Packer](https://github.com/mehlj/mehlj-packer)

[Source code - Terraform](https://github.com/mehlj/mehlj-terraform)

[Source code - Ansible](https://github.com/mehlj/mehlj-ansible)

[Source code - Application](https://github.com/mehlj/mehlj-pipeline)


# Background
In this project, I will be deploying several clustered instances of a simple REST API written in Golang. To accomplish this, I will containerize the application and deploy it into an on-prem Kubernetes cluster.

There are two major components of the underlying infrastructure for this application - the Kubernetes cluster itself, and the worker and control plane nodes that comprise the cluster.

Just as we are defining the application and application configuration as code, we will define these two major infrastructure components entirely as code, and leverage automation to provision them.

# Goal
An automated, secure, and code-defined pipeline(s) that, when run, provisions virtual machines and bootstraps a working Kubernetes cluster populated with a Golang REST API.

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
module "k8snode0" {
  source = "./modules/cluster_node/"
  
  vm_name            = "k8snode0"
  vm_domain          = "lab.io"
  vm_template        = "CentOS8_Template"
  vm_datastore       = "datastore1"
  vm_folder          = "/"
  vm_num_cpus        = 2
  vm_num_memory      = 4096
  vm_ip              = "192.168.1.210"
}

module "k8snode1" {
  source = "./modules/cluster_node/"
  
  vm_name            = "k8snode1"
  vm_domain          = "lab.io"
  vm_template        = "CentOS8_Template"
  vm_datastore       = "datastore1"
  vm_folder          = "/"
  vm_num_cpus        = 2
  vm_num_memory      = 4096
  vm_ip              = "192.168.1.211"
}

module "k8snode2" {
  source = "./modules/cluster_node/"
  
  vm_name            = "k8snode2"
  vm_domain          = "lab.io"
  vm_template        = "CentOS8_Template"
  vm_datastore       = "datastore1"
  vm_folder          = "/"
  vm_num_cpus        = 2
  vm_num_memory      = 4096
  vm_ip              = "192.168.1.212"

  # optional variables
  bootstrap_cluster  = true
}
```

To cut down on overall code, and to use [DRY techniques](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/), we use [modules](https://www.terraform.io/language/modules/develop).

All reusable code is placed in the module `cluster_node`, and we pass variables to it, much like a function.

```H
<snip>
resource "vsphere_virtual_machine" "cluster_node" {
  name             = var.vm_name
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.datastore.id
  folder           = var.vm_folder

  num_cpus               = var.vm_num_cpus
  memory                 = var.vm_num_memory
  cpu_hot_add_enabled    = true
  cpu_hot_remove_enabled = true
  memory_hot_add_enabled = true
  guest_id               = data.vsphere_virtual_machine.template.guest_id

  scsi_type = data.vsphere_virtual_machine.template.scsi_type

  network_interface {
    network_id   = data.vsphere_network.mgmt_lan.id
    adapter_type = data.vsphere_virtual_machine.template.network_interface_types[0]
    use_static_mac = var.vm_static_mac
    mac_address    = var.vm_mac_address
  }
<snip>
```

### Pipeline
In order for Terraform to deploy the infrastructure based on the code you write, you must call the binary CLI like so: `terraform apply`

We can also leverage several Terraform CLI commands to perform other best practices, like initialization, planning, and formatting.

In order to maintain the automated nature of this overall workflow, we can place all of these commands and more into a GitHub actions pipeline (since the code is stored in GitHub).

The Actions runner will perform these commands automatically when the repository is pushed to.

Note: for production usage, manual checking of Terraform code before applying is recommended. For this lab use case, I apply the code without review.

```
<snip>
  # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading modules, etc.
- name: Terraform Init
  run: terraform init

  # Checks that all Terraform configuration files adhere to a canonical format
- name: Terraform Format
  run: terraform fmt -check

  # Generates an execution plan for Terraform
- name: Terraform Plan
  run: terraform plan

  # On push to master, build or change infrastructure according to Terraform configuration files
  # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
- name: Terraform Apply
  if: github.ref == 'refs/heads/master' && github.event_name == 'push'
  run: terraform apply -auto-approve
<snip>
```



## Kubespray

When our Terraform pipeline executes, we have three instances of our CentOS template deployed into our vSphere cluster.

Our end goal, however, is to provision a container orchestration system into this infrastructure.

We can leverage [kubespray](https://github.com/kubernetes-sigs/kubespray) to bootstrap a Kubernetes cluster on X amount of servers.

Kubespray is a Kubernetes SIGs product that leverages Ansible to bootstrap a cluster entirely in an automated fashion.

In my case, I [forked the official kubespray repository](https://github.com/mehlj/kubespray) and stored my deployment-unique variables/files in my forked repo.

My variables specified several things, including the IPs of the control plane and worker nodes, network plugin, runtime engine, etc.


### Pipeline

Once all of your variables are specified, we can bootstrap the cluster using a simple `ansible-playbook` command.

However, we want a little more done than just bootstrapping the cluster. Specifically, I need:
* SELinux enabled (kubespray unfortunately disables it on all nodes)
* kubectl configured for non-admin users
* A couple of applications deployed into the cluster

A simple wrapper script for kubespray can help chain these tasks together:
```bash
#!/bin/bash

# NOTE - the ansible vault password file must contain the correct password to decrypt the Traefik private key file

# example usage:
# ./bootstrap.sh -n mehlj-cluster -f ~/.vault_pass.txt

while [[ "$#" -gt 0 ]]; do
    case $1 in
        -n|--kubespray-cluster-name) kubespray_cluster_name="$2"; shift ;;
        -f|--vault-password-file) vault_password_file="$2"; shift ;;
        *) echo "Unknown parameter passed: $1"; exit 1 ;;
    esac
    shift
done

# Bootstrap cluster with kubespray
ansible-playbook -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml kubespray/cluster.yml -b --private-key /home/runner/.ssh/github_actions

# Enable SELinux
ansible all -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml -m selinux -a "policy=targeted state=enforcing" -b --private-key /home/runner/.ssh/github_actions

# Change kubelet.env SELinux context to resolve inital issue
ansible all -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml -m shell -a "semanage fcontext -a -t etc_t -f f /etc/kubernetes/kubelet.env; restorecon /etc/kubernetes/kubelet.env" -b --private-key /home/runner/.ssh/github_actions

# Reboot all nodes and wait for them to come back up
ansible all -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml -m reboot -b --private-key /home/runner/.ssh/github_actions

# Allow non-credentialed use of kubectl (only on control plane hosts)
ansible kube_control_plane -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml -m file -a "path=/home/mehlj/.kube/ owner=mehlj group=mehlj state=directory" -b --private-key /home/runner/.ssh/github_actions
ansible kube_control_plane -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml -m copy -a "src=/etc/kubernetes/admin.conf dest=/home/mehlj/.kube/config owner=mehlj group=mehlj remote_src=yes mode=0600" -b --private-key /home/runner/.ssh/github_actions
```

Now, the shell script can be executed with some arguments, and all those tasks will be completed without human intervention. 

But how do we execute it within the perspective of the automated workflow?

There are a few ways to tackle this, but to experiment with Terraform provisioners, I chose to let Terraform handle execution of this script.

```H
locals {
  provisioner_command = "sleep 60; ./bootstrap.sh -n mehlj-cluster -f /tmp/.vault_pass.txt"
}
```

Only problem is, we don't want this to run on EVERY instantiation of the Terraform module. The cluster bootstrapping and post-tasks need only happen once.

To combat that issue, we can introduce some logic to only execute the shell script when a parent module-supplied variable `var.boostrap_cluster` is set to `True`.
```H
# -----Psuedocode:-----
# if var.bootstrap_cluster = true:
#   Provision the Kubernetes cluster using the bootstrap.sh script
# else:
#   skip the cluster bootstrapping, and only run a benign echo command to keep Terraform happy
# -----end Psuedocode-----
# 
# 
# This logic allows the cluster to be provisioned only once, to speed up the pipeline execution.
# The cluster bootstrapping is entirely idempodent, but kubespray takes some time to complete. 
# The cluster bootstrapping only needs to be run once, so omitting the step when it is not necessary allows the pipeline to be run faster.
provisioner "local-exec" {
  command = join(" && ", ["echo Bootstrapping cluster..", var.bootstrap_cluster != false ? local.provisioner_command : "echo Not bootstrapping cluster.."])
}
```

To that end, we can set that variable to `True` on only one instanatiate of the module (in my case, `k8snode2`).

## REST API Deployment
At this point, we have a working, yet empty, Kubernetes cluster.

Our next step is to deploy our simple REST API into the cluster to serve as a working proof-of-concept.

We could add another job/stage to our GitHub actions workflow, but in order to keep the pipeline code lean, we should add a final step to our bootstrap script:

``` bash
<snip>
# Deploy traefik and example hello-world applications
ansible-playbook ansible/playbooks/traefik.yml -i kubespray/inventory/$kubespray_cluster_name/hosts.yaml --limit node1 -b --vault-password-file $vault_password_file --private-key /home/runner/.ssh/github_actions
```

This snippet invokes a custom Ansible role:
```yaml
---
- name: Deploy traefik as NodePort and two example applications
  hosts: all
  become: yes
  roles:
    - ../roles/deploy-traefik
```


Which utilizes Helm to deploy Traefik and various manifests to deploy a hello-world application, as well as the Golang REST API.
```yaml
---
# tasks file for deploy-traefik
<snip>
- name: Deploy traefik
  shell: /usr/local/bin/helm install traefik --set service.type=NodePort --set ports.web.nodePort=30001 --set ports.websecure.nodePort=30002 --set ports.websecure.tls.enabled=true --set deployment.replicas=3 traefik/traefik
  args:
    chdir: /tmp/

- name: Merge YAML
  template:
    src: "{{ item }}"
    dest: "/tmp/{{ item }}"
  loop:
    - tlsstore.yml
    - app-deployment.yml
    - deployment2.yml
    - app-svc.yml
    - svc2.yml
    - app-ingress.yml
    - hello2-ingress.yml
    - nfs-pv.yml
    - nfs-pvc.yml

- name: Apply YAML
  shell: /usr/local/bin/kubectl apply -f "{{ item }}"
  loop:
    - /tmp/tlsstore.yml
    - /tmp/app-deployment.yml
    - /tmp/deployment2.yml
    - /tmp/app-svc.yml
    - /tmp/svc2.yml
    - /tmp/app-ingress.yml
    - /tmp/hello2-ingress.yml
    - /tmp/nfs-pv.yml
    - /tmp/nfs-pvc.yml
    
- name: Wait for http://app.lab.io to respond externally via ingress
  uri:
    url: "http://app.lab.io:30001"
    method: GET
    follow_redirects: none
  register: _result
  until: _result.status == 200
  retries: 30
  delay: 5

- name: Wait for https://hello2.lab.io to respond externally via ingress
  uri:
    url: "https://hello2.lab.io:30002"
    method: GET
    follow_redirects: none
    validate_certs: no
  register: _result
  until: _result.status == 200
  retries: 30
  delay: 5
```

The last two tasks in the Ansible role ensure that both applications return `200`, so once the Ansible role finishes execution, we can be assured that our application stack is deployed and functioning.

## REST API Updates
When we make changes to the REST API later down the road, we don't necessarily want the entire cluster to be destroyed and re-provisioned on every merge to `main`.

Instead, we want a rolling update to our existing cluster. That way, we can see value delivered as quickly as possible.

To account for this use-case, we can engineer a separate GitHub actions workflow, attached to our REST API GitHub repository, that builds, tests, and deploys our container image to the existing cluster.

```yaml
<snip>
jobs:

  build:
    runs-on: mehlj-lab-runners
    steps:
    - uses: actions/checkout@v2
    
    - name: Build Docker image
      run: docker build -t docker.io/mehlj/mehlj-pipeline:latest .

  test:
    runs-on: mehlj-lab-runners
    needs: build
    steps:
    - name: Grab go dependencies
      run: go get -u github.com/gorilla/mux github.com/mattn/go-sqlite3
    
    - name: Run unit tests
      run: go test -v api/main_test.go api/main.go api/sql.go
    
    - name: Lint Dockerfile
      run: hadolint Dockerfile
      
  deploy:
    runs-on: mehlj-lab-runners
    needs: [build, test]
    steps:
    - name: Push Docker image
      run: docker image push docker.io/mehlj/mehlj-pipeline:latest
      
    - name: Deploy to kubernetes
      run: ssh -o StrictHostKeyChecking=no root@k8snode0 kubectl rollout restart deployment mehlj-pipeline-deploy
```

This workflow builds our container image using the repository `Dockerfile`, performs unit testing and linting, pushes the image to Docker Hub, and performs a rolling update via the command `kubectl rollout restart deployment mehlj-pipeline-deploy`.

This command, within the `Deployment` resource, replaces instances of the old container image with instances of the new image, while ensuring that the application is never left without X number of healthy container instances.

In this manner, the application can receive live updates without users being affected by any sort of downtime.

# Outcome
After our GitHub Actions workflow executes, we are left with x3 CentOS VMs in our vSphere cluster, a working Kubernetes cluster on those VMs, and some automatic post-configuration applied.

The next phase of our workflow deploys a containerized reverse proxy and our simple golang REST API into the cluster.

[Another GitHub Actions workflow](https://github.com/mehlj/mehlj-pipeline) is present, to account for later changes to the REST API. The workflow builds, tests, and performs a rolling deployment update inside the Kubernetes cluster.
