---
layout: post
title: DevSecOps Overview
published: true
category: infrastructure
---

# Issue

Software engineering within today's technological climate is fraught with challenges. 
## Velocity

To stay competitive in today's market, features must be released to customers rapidly. 

Not only that, but these features must have direct fiscal gain for the company. Features that cause fiscal loss for the company must be tracked and rescinded accordingly.

## Software Vulnerabilities

With organizations releasing software as quickly as possible, it is only natural that software bugs will emerge.

Even small, seemingly benign bugs can be chained together and result in a serious software vulnerability. 

Such vulnerabilities could result in significant fiscal loss for the organization, in the form of customer data compromise (reparations), fines, and intangible reputation damage. 

## Bolt-on Security

As a response to increased deployment of bugs to production, companies often opt to implement more security controls in production. Implementations like perimeter firewalls, IDS/IPS, host-based AV, etc.

While these are effective controls, they cannot be relied upon fully to prevent cybersecurity incidents.

Companies may neglect to implement security controls in their software product itself, due to the costly nature of it, in both time and effort. 

The end result is attackers leveraging a bug in the companies' application itself, and bypassing all perimeter/production security while doing so.


## Rigid Infrastructure

Companies not only need to be able to deliver features quickly, but they must be able to deliver these features reliably and with very limited downtime for customers.

With high-visibility features being released, production environments must be able to adapt to rapid fluctuations of traffic. 

If features do not function responsively and smoothly in production after launch, then the feature is a failure, despite however much work went into the software engineering of it.

As such, infrastructure must be elastic, scalable, and high performing. 

The issue is: traditional I.T. infrastructure is rigid and inefficient, especially in high-security industries (banking, healthcare, government, etc.).

The nature of said infrastructure is in the interest of cybersecurity teams, as static, unchanging infrastructure is significantly easier to account for/track over a long period of time.


## Team Boundaries

Issues like rigid infrastructure lead to resentment among team members. Development teams may work extremely hard on a feature release, and rigid infrastructure maintained by the operations team causes the release to be a flop. Or, vice versa.

Resentment leads to strong dividing lines between operation and development teams. A culture of separation, and not of cooperation.

The introduction of cybersecurity teams can exacerbate this culture shift - as security professionals may be seen as an unmoving, uncooperative entity that only dumps more work on other teams, due to the last-minute bolt-on security implementations.

This culture shift leads to a classic deployment method of "throwing it over the fence", so to speak, wherein developers finish writing and testing a feature/release, and simply hand the software update to the operations team.

The operations team, without much context to the situation, has to deploy that feature to production, without the advice or help of the experts writing the code. 

This ultimately leads to long and painful deployments during maintenance periods, which causes more animosity, stress, burnout, and financial loss for the organization. 

## Competition

On top of all of those challenges, teams must be aware of their competitors. Competitors may already have adapted to the current landscape, and could be leaving your organization in the dust.


## Summary

To sum it all up, organizations are moving too quickly for software to be released frequently, securely, and in a scalable manner.

There needs to be a methodology that allows organizations to implement the frequent-release tenants of DevOps without sacrificing confidentiality, integrity, and reliability.



# Solution

In order to not sacrifice security while meeting DevOps fundamentals, security controls need to be introduced at every stage of the software's lifecycle - which is the essence of DevSecOps.

## Velocity

Organizations may or may not be releasing rapidly already, but DevSecOps compiles with the standard DevOps practice of frequent and small releases. 

## Software Vulnerabilities

DevSecOps should not slow you down, if implemented correctly. DevSecOps is all about introducing security control implementation and testing early, so if vulnerabilities are discovered, they are typically found early on. 

Organizations can respond to vulnerabilities during build and test phases, instead of in staging or production. Before the vulnerability becomes costly to the organization.

For example, say a development team discovers a backdoor in one of their critical libraries that is easily exploited, similar to the infamous VSFTPD 2.3.4 backdoor.

If the development team discovered it during static code analysis (SCA) scanning during their build or test phases, they can simply patch the library and run it through test again, no harm, no foul. Barely a schedule alteration for a significant vulnerability.

If the organization discovered the vulnerability late into the release stages, like in staging or production, significant time would be lost, as the release would have to backtrack the process and end up back at dev → test → staging again.

## Bolt-on Security

Take our previous example. The organization may determine it's too costly (time or otherwise) to backtrack and fix the vulnerability now, so they push the release to production anyways. They implement a bolt-on security control, like an IPS signature for that particular exploit. 

However, the offensive security landscape is rapidly evolving. They have found a method of masquerading their traffic to avoid common signature detection, and have exploited the production software, compromised customer data, and cost the company dearly.

DevSecOps promotes Defense-in-Depth (DiD), and implores organizations to implement security controls early (like the aforementioned patch), while at the same time, implementing boundary protection for production (like the aforementioned IPS). 

Both methods are helpful for avoiding security incidents, but it is best practice to combine them - hence defense in depth.

## Rigid Infrastructure

DevSecOps promotes transient and immutable infrastructure, as stagnant and long-lived infrastructure not only is unable to adapt to the rapid development climate, but can lead to security incidents. 

The longer infrastructure exists on the network, the longer it has a valid attack surface. Infrastructure should be provisioned as needed, and promptly destroyed when the engineering effort is complete.

This does cause an issue with asset tracking, however. Traditionally, to maintain awareness of the overall risk of the organizations' network, each infrastructure piece is documented and reported on over long periods of time. However, this is in direct opposition with transient infrastructure.

Instead, all infrastructure should be instantiated from a repository of vetted and approved images. From Docker images to VM templates built via Packer, all images should have security controls baked-in during build time. After a successful build, security control reports should be generated and attached to the pipeline outcome as an artifact.

Cybersecurity teams can then be assured that their infrastructure is all built from these baseline images. They need only track the amount of instantiations that the network currently has. I.e., 30 docker containers of XYZ image, 20 Linux VM deployments, etc. From this information, they can gather exactly 1) what your infrastructure is comprised of, and 2) how large your infrastructure is - at any point.

One wrench in this method is the existence of configuration drift. This is the process of an image instantiation becoming less and less resemblant of the baseline image from which it was initially formed. This usually occurs due to human intervention - fast, undocumented changes to quickly resolve issues. Drift causes a rift between a) what cybersecurity teams are reporting up the chain, and b) what actually exists on the network.

Having transient infrastructure helps to combat drift - as short-lived infrastructure is less likely to encounter drift at the human hand.

Another method of combating, and likely eliminating, drift is implementing immutable infrastructure. Immutable infrastructure prevents humans, even administrators, from manually implementing changes to image instantiations themselves (i.e., via ssh). Instead, the only method of changing the infrastructure is to update the code that forms that image, and then replacing that server/container with the updated image.

Immutable infrastructure allows for 100% accurate reporting from cybersecurity teams, as the risk landscape can be assessed via the code that comprises the images. They can be sure that the code matches what is deployed in production, since the infrastructure is immutable.

## Team Boundaries

Introducing all teams (development, security, and operations) early in the software development lifecycle is imperative to preventing resentment and promoting inter-team cooperation.

Constant cooperation is key, as no-one wants to be forced to work with a seemingly outside entity that is brought in at the last minute. Teams need to be aware of responsibilities, intent, and plan accordingly as early as possible. 

Implementing security controls early also improves cooperation with cybersecurity teams, as less incidents occur, and devs become more used to working with quality control gates that relate to security.

## Competition

If an organization is implementing DevSecOps, and are releasing software frequently and securely, they have already made significant strides to stay competitive in the current market. 
