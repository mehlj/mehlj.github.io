---
layout: post
title: DevSecOps - Culture
published: true
category: infrastructure
---
# Issue
Teams shifting to DevSecOps implementations often put a heavy focus on technical improvements, and do not strive to align their team culture with that of proper DevSecOps principles.

A strong team culture shift should enacted as quickly as any other technical improvement.

Without a cultural alignment, DevSecOps practices may not gain steam or persist beyond a few team members.



# DevSecOps Solution
Cultural shifts are hard to quantify and measure, but it helps to educate team members with overarching principles/tenants instead of overloading them with information.

Teams should, gradually, strive to align themselves with many DevSecOps cultural tenants. It helps to start with a handful of more significant tenants.

## Tenants
These tenants are taken from the official DoD Enterprise DevSecOps Fundamentals documentation:

### Continuous delivery of small incremental changes.
Small, incremental changes to a codebase allow for more constant and thorough usage of the CI/CD pipelines and code quality gates, thus ensuring that these workflows are working properly and will work properly in the future.

This also gets the development and ops team into a mindset of constant, repeated delivery, instead of a handful of large deliveries across a long time period. Ops teams are fully aware that those types of large deployments leave more room for error, and typically shell-shock the user as well.



### Bolt-on security is weaker than security baked into the fabric of the software artifact.
Security controls are more effective when closer to the product you are protecting. The offensive security landscape rapidly evolves, and attackers are, now more than ever, adept at bypassing outlying security controls, rather than taking significant time to develop an exploit kit or use clever tactics to disengage the controls.

Misconfiguration of bolt-on security tools are a significant threat, and attackers know that many teams are understaffed, undertrained, and are sprinting to make the solution work.

In this landscape, adding security controls as close to the original code as possible is the most effective defense. Attackers must be met with defense-in-depth, and cannot be allowed to bypass outlying security controls and immediately compromise their target. 

A layered defense is necessary, and the most effective manner of employing this is by adopting DevSecOps principles and shifting security as left as possible.



### Value open source software.
Open source software was historically seen as less secure than closed-source software, simply by the reasoning that "anyone" can modify open-source code.

As open source software has become more proliferate, the public has realized that this is not the case.

The source code is publicly available, but sponsoring organizations or developer teams have quality control gates in place to prevent outright malicious code from being merged to master.

With the security concern waning, we should now utilize open source software to gain the advantages it has always given: thoroughly tested, understood, and comprehensive software that can be leveraged by your organization quickly and freely.



### Engage users early and often.
If teams are responsible for delivering infrastructure or software products to end users, the end users should be engaged often and as early in the design process as possible.

Teams may get lost in the minutia of implementing complex DevSecOps solutions and lose sight of the original goal - to improve deliverables for the customer.

To ensure that the original goal is kept first and foremost, end users should be advised on significant decisions, design choices, etc. If the user brings up valid points against your implementation decisions, then more design conversations are necessary.



### Prefer user centered & Warfighter focus and design.
Likewise, in early design phases, the users should be consulted as to what they are looking to achieve from this DevSecOps implementation. Further DevSecOps efforts should be in alignment of the customer's goals, not vice versa.

This tenant should be held in even higher standing if the product/infrastructure you maintain is in direct support of business-critical customers.



### Value automating repeated manual processes to the maximum extent possible.
Automation should clearly be valued for the sake of time-saving; Significant time and effort can be saved if a tedious manual process is automated.

However, automation also has strong security posture assurance benefits as well.

Humans are prone to make mistakes, even during a process they have completed many times before. In DevSecOps environments, implementing and updating security controls are key, and such human mistakes can be detrimental. Thus, automation should be relied on as much as possible.



### Fail fast, learn fast, but don’t fail twice for the same reason.
Failure is completely necessary in complex automated workflows, and they are to be expected. What changes in DevSecOps culture is how your team reacts to them.

Failure should be detected as soon as possible (strong logging + alerting mechanisms with severity ratings) and should be understood as soon as possible. [Root cause analysis](https://sre.google/sre-book/postmortem-culture/) is an effective methodology to adhere to immediately after these failures.

Traditional ops teams may panic in such failures and attempt to restore backups or undo changes that have been made recently, without performing root cause analysis. This is ineffective - learning from failure is imperative. Learning from the failure and implementing a long-term fix is the only way to be sure the failure will not occur again.



### Fail responsibly; fail forward.
Since failure during deployments is a fated and repeating occurrence, your team must be prepared with a general strategy for restoring functionality to users.

Traditionally, when operations teams encounter critical deployment failure, the knee-jerk reaction is to roll back to the last working version and declare the deployment a failure. Sometimes, depending on the severity of the incident, this is a wise technique.

However, while this technique often helps teams stem the bleeding quickly, it does not create a culture of blame-free learning, failure adaptation, and growth. Sometimes, if backup restoration is involved, it can be time costly as well. 

The preferred technique is to fail forward. This, in essence, means that teams will react to failure by clearly recognizing its impact, performing root cause analysis, and resolving the issue by either a) relying on automated failure recovery/self-healing (preferred), b) deploying more changes to resolve the issue (i.e., another application release), or c) accepting the minor failures in the interest of the overall deployment. 



### Treat every API as a first-class citizen.
APIs are becoming increasingly important in today's technology ecosystem, as more and more workloads are becoming fully automated and devoid of any human interaction.

These automated workloads are usually powered by an Application Programming Interface (API), much like an end-user's manual workloads are powered by Graphical User Interfaces (GUIs). 

APIs can often be forgotten and deprioritized due to the fact that they have no human-visible interface, and often do not service end-users directly (only indirectly). 

In reality, APIs should be monitored and built with just as much reliability and durability as your typical web application with a graphical frontend. Otherwise, your automated workflows may slowly fail and it may fall upon slow, manual, human intervention to accommodate the workload gap.



### Good code always has documentation as close to the code as possible.
Applications, and the code they are comprised of, are notoriously difficult to document. Some believe that code comments are sufficient, while others believe that entire handbooks should be written for each microservice. Some write code comments in extreme verbosity, while others are vehemently against code commenting in the first place.

A sufficient solution is a sort of balance between all mindsets. Code comments should be concise and clear, only describing the why something was done, not the what. The what can be determined from simply reading and interpreting the code. Interpreting the author's thinking at the time of writing, however, is a more difficult challenge that can be alleviated with code comments.

In complex systems, code should have additional supporting documentation. In such scenarios, it is important that the documentation be kept as "close" to the code as possible. For example, if the code is stored in a git repository, then the supporting documentation, perhaps written in Markdown, is well suited to being stored in the same git repository, underneath a directory called /docs. 

The reason this is important is due to the longtime issue of documentation maintenance - awareness. Documentation can be numerous and sprawling, with many team members not aware of some documentation's existence. To better promote team awareness of documentation, put the docs as close to the code as possible. This ensures that the docs will be read more frequently, because team members will always be working with the code.



### Recognize the strategic value of data; ensure its potential is not unintentionally compromised.
Data, and the historical records kept within, is the key to continuous improvement and agility - the overall mantra of DevSecOps.

As previously mentioned, learning from failure, past incidents, and simply tracking overall metrics is key to continuous improvement. Growth must be tracked, to determine if the team is on the right path, and incidents must be studied, to better prevent them in the future. 

As such, data must be aggregated and utilized fully. Plenty of teams are sufficient at data aggregation, but falter when it comes to utilization. Recent visualization technology (Grafana, ELK stack, etc.) can help in this regard.

This data is also very useful to adversaries, so data-at-rest protection is of upmost importance. Disk encryption, strong key lifecycle management, and access protections are key.

