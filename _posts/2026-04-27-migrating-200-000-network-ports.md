---
layout: post
title: "Migrating 200,000 Network Ports: How to Swap SDNs in a Live Public Cloud"
---

In my previous post, I provided an  analysis on why legacy Software-Defined Networking (SDN) architectures like OpenStack Neutron break at scale. However, architecting and deploying a modern SDN is only half the solution. The second half is the migration itself. How do you take a live public cloud with over 160,000 running VMs and roughly 200,000 virtual ports, and replace its underlying networking without causing catastrophic downtime? This post details the technical methodology, the operational realities required to execute a hyperscale infrastructure swap.

To be clear, migrating an environment of this size is a massive, cross-functional effort involving Product Managers, Software Developers, L1/L2 Tech Support, Site Reliability Engineers (SREs), and Solution Architects. As a Lead Solution Architect, I led the urgent migration of our highest-paying clients responsible for the majority of the cloud's revenue. I developed Public API-based migration tools for these top-tier customers. Using the Public API is the safest way to migrate stateful infrastructure. It respects the cloud's built - in safety checks and quotas, unlike dangerous backend database modifications that bypass user validation and risk silent corruption. These tools and the edge-cases discovered during their use then served as the core requirements and blueprint for the internal, automated pipelines used by the SRE team to migrate the remaining thousands of smaller clients seamlessly.

While we will look specifically at the migration from OpenStack Neutron to VK Cloud's proprietary Sprut SDN, the core methodologies discussed here are not exclusive to OpenStack. The principles of dependency graphing, state reconciliation, and blast-radius reduction can be applied to any large-scale cloud provider or bare-metal migration.

We will also explore why swapping an SDN is not just an IaaS (Infrastructure-as-a-Service) problem. Higher-level PaaS offerings-like Database-as-a-Service (DBaaS) and Kubernetes-as-a-Service (KaaS) - are built directly on top of this networking layer and require highly specialized care during the transition. PaaS environments often involve complex, dynamically scaling resources and shared multi-tenant networks (like load balancers and ingress controllers) that easily break if the underlying network topology changes unexpectedly.

This post details experience gained in this massive project. We will break it down step-by-step, how we approached the problem, from laying out all network dependencies and fixing Terraform state files, to the politics of convincing enterprise CTOs and CEO to accept scheduled maintenance windows.

## Table of Contents

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

## VII. Translating technical debt into executive business value.

* Key takeaways

* What I learned, and what I would do differently today.
