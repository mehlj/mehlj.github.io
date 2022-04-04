---
layout: post
title: DevSecOps - Cattle, not Pets
published: true
category: infrastructure
---

# Issue
One shift from traditional software development practices to DevSecOps practices is the life cycle management of back-end infrastructure, such as servers.

Traditionally, server infrastructure are treated as pets, each with their own quirks. They are kept online as long as possible, or until the customer formally requests a decommission. This leads to a major issue: configuration drift.

Configuration drift is the process of a system slowly becoming unrecognizable from its original configuration - the state at which it was at immediately after being presented to customers (the "gold" configuration).

We want servers to stay as close to the gold configuration as possible, because the gold configuration is easily reproducible (from documentation or automation) in the event that the server crashes or needs a full rebuild.

With "pet" servers staying online or in production for months or years, configuration drift slowly occurs from manual changes made by administrators, standard patching/hardening, or by routine maintenance. This ultimately results in slower scaling and response to critical issues.


# DevSecOps Solution
The general DevSecOps approach to server infrastructure is to the treat them as cattle - not pets.

In practice, this means deploying your infrastructure such that anything can be destroyed and recreated identically, through an automated workflow.

Infrastructure should not be long-lived, they should scale up and down or be destroyed and provisioned only as they are needed.

If clustering and automatic failover is available, rebuilding the entire server to receive software updates is another effective method of combating drift.

## Templates
Since servers will be deployed routinely and rapidly, you must build from a "gold" image - or template.

The template is an artifact that you can clone or build from. All server configurations will stem from this minimal, gold image.

When a unique server configuration is required, i.e., an Apache httpd server, configuration management code is written that brings an instance of that gold image → a fully functioning Apache httpd server.

That configuration management code (i.e., Ansible) is very important, as this is what allows that unique server workload to be reproducible. If that workload needed to scale up (in response to heightened user traffic), or needed to be rebuilt all together, an automated pipeline would:

* Clone/build a new server from the gold template
* Apply the configuration management code to configure the server with this unique workload
* Deploy the new server to the appropriate location (VLAN, etc.)

## Immutable
Where possible, deployed servers should be individually immutable.

In practice, this means that server's running configuration CANNOT be changed manually, the only way to alter a server configuration is by changing the code and going through the standard GitOps workflow again to deploy the changes (replacing the original).

This is a difficult practice, but it essentially prohibits drift from occurring, and enforces the optimal GitOps workflow that all DevOps practitioners should be following anyways. Change the code, redeploy the template instance, don't change the server while it is running.

The approach has a couple of downsides:

* Takes more time to deploy changes (but not effort).
  * Since you cannot change servers that are currently running, you must destroy them + redeploy them using a newer template every time you want to make any change - large or small.
  * The process of updating the template source code, destroying the running instances, and deploying them again using the new template - costs time. In theory, it would be much quicker to simply make the small change directly to the running instances in production, but this is what causes drift.
  * This process seems like it would result in more effort from maintainers, but it should not, because this entire template updating workflow should be automated. The maintainer should edit the template source code, and CI/CD pipelines should automatically handle the rest.
* Requires strong clustering support.
  * It was previously mentioned that the process of updating immutable infrastructure involves destroying the old running instances and deploying new ones. This would obviously cause downtime for our users, so this must be mitigated through the use of load balancers and server clustering - i.e., EC2 auto scaling groups and Application Load Balancers (ALBs).
  * In such a clustered/load balanced environment, the introduction of a new template would be handled in a slow, methodical manner that causes zero downtime for users - such as the blue green deployment method.
