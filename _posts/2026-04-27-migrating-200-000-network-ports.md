---
layout: post
title: "Migrating 200,000 Network Ports: How to Swap SDNs in a Live Public Cloud"
---

In my previous post, we explored why legacy network architectures mathematically collapse at hyperscale. But architecting a modern Software-Defined Network on a whiteboard is only half the battle. How do you take a live public cloud with over 160,000 running VMs and replace its underlying network without affecting clients? This post details the technical methodology and the operational realities of swapping the networking engine of a live public cloud.

## I. Introduction: Migrating a Live Public Cloud

* Recap the "200k port" Neutron disaster.

* Why do we need migration in first place.

* The hyperscaler dilemma: How VK Cloud’s client-oriented approach differed from the standard "migrate it yourself" industry playbook.

## II. The Philosophy of Migration: State Machines vs. Database Hacks

* Why we abandoned the "perfect" backend SRE tool for Public API scripts.

* Treating the cloud as a State Machine.

* Reducing the blast radius (why moving VIPs saved the legacy Neutron users too).

## III. The Engineering Methodology: Dependency Graphing

* How to build a migration tool from scratch.

* API field mapping and determining the strict execution order.

* Why we chose Bash + OpenStack CLI over Python to reduce client friction.

## IV. Entity Caveats: IPsec, Advanced Routers, and 45-Second Windows

* The technical realities of the migration.

* Handling Security Groups with -sprut postfixes.  

* The IPsec / Advanced Router DNAT architecture changes.

* The 45-second VM disconnect/reconnect window.  

## V. The Infrastructure-as-Code Trap: Fixing Terraform State

* You can't just delete and recreate a 5,000 VM deployment.

* How we allowed clients to adopt the new SDN without destroying their existing Terraform state files using terraform import

## VI. The Human Element: Selling Downtime to Angry CEO and CTOs

* The "Shoot the Messenger" dynamic.

* Translating technical debt into executive business value.

## Translating technical debt into executive business value.

* Key takeaways

* What I learned, and what I would do differently today.
