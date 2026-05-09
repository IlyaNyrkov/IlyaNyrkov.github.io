---
layout: post
title: "Migrating 200,000 Network Ports: How to Swap SDNs in a Live Public Cloud"
---

In my [previous post](/2026/04/17/surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale.md), I provided an  analysis on why legacy Software-Defined Networking (SDN) architectures like OpenStack Neutron break at hyperscale and why we need to replace them. However, architecting and deploying a modern SDN is only half the solution. The second half is the migration itself. How do you take a live public cloud with over 160,000 running VMs, 3,000 bare metal hypervisors and roughly 200,000 virtual ports, and replace its underlying networking without causing catastrophic downtime? This post details the technical methodology, the operational realities required to execute a hyperscale infrastructure migration. This is a real challenge that was also faced by AWS, GCP, Azure and other hyperscalers.

To be clear, migrating an environment of this size is a massive, cross-functional effort involving Product Managers, Software Developers, L1/L2 Tech Support, Site Reliability Engineers (SREs), and Solution Architects. As a Lead Solution Architect, I led the urgent migration of our top clients responsible for the majority of the cloud's revenue. I developed Public API-based [migration tools](https://github.com/vk-cs/neutron-2-sprut/tree/main) and methodology for these customers. Using the Public API is the safest way to migrate stateful infrastructure. It respects the cloud's built-in safety checks and quotas, unlike unstable backend database modifications that bypass user validation and risk silent corruption. These tools and the edge-cases discovered during their use then served as the core requirements and blueprint for the internal, automated pipelines used by the SRE team to migrate the remaining thousands of smaller clients seamlessly.

While we will look specifically at the migration from OpenStack Neutron to VK Cloud's proprietary Sprut SDN, the core methodologies discussed here are not exclusive to OpenStack. The principles of dependency graphing, state reconciliation, and blast-radius reduction can be applied to any large-scale cloud provider or bare-metal migration.

We will also explore why swapping an SDN is not just an IaaS (Infrastructure-as-a-Service) problem. Higher-level PaaS offerings-like Database-as-a-Service (DBaaS) and Kubernetes-as-a-Service (KaaS) - are built directly on top of this networking layer and require specialized care during the transition. PaaS environments often involve complex, dynamically scaling resources and shared multi-tenant networks (like load balancers and ingress controllers) that easily break if the underlying network engine changes unexpectedly.

This post details experience gained in this massive project. We will break it down step-by-step, how we approached the problem, from laying out all network dependencies and fixing Terraform state files, to the politics of convincing enterprise CTOs and CEO to accept scheduled maintenance windows.

## Table of Contents

## I. Reality of Major Public Cloud Migrations

Replacing the underlying network or control plane of a live cloud is a monumental engineering challenge. It is a high-risk maneuver that vendors avoid for as long as possible, yet the physics of hyperscale growth eventually force everyone's hand. To understand the operational reality of our migration, we first need to look at why these migrations happen and how the rest of the industry handles them.

### Why Hyperscalers Replace Foundational Architectures

Replacing the foundational architecture of a live cloud is rare, nevertheless, almost every major hyperscaler has faced it. When cloud providers decide to execute migrations of this scale, the decision is driven by unavoidable architectural limitations:

1. **Hitting the "Flat" Scalability Ceiling:** Early clouds started with "flat" networks. Flat architectures are fast to build but fail at scale because broadcast domains become too large, IP exhaustion occurs, and routing tables become too complex to compute. Migrations are required to introduce hierarchical, subnet-based VPC architectures.
2. **Distributed State & Reliability:** Early control planes relied on centralized databases. As the cloud grows in server and datacenter count, a single database lock causes a cascading failure. Control planes must be rewritten to use highly distributed, eventually-consistent state storage (like Paxos or Raft consensus algorithms) to ensure regional failures don't take down the entire cloud.
3. **Reducing Computational Overhead:** Software-based control planes running on the host server steal valuable CPU cycles from the customer. Migrations are driven by the need to push network and storage processing down to the hardware level (via SmartNICs or Data Processing Units), dramatically lowering latency.
4. **The Feature-Parity Trap:** Architectural debt prevents revenue. If an old SDN cannot natively support IPv6, overlapping IP spaces, or VPC Peering, the vendor loses enterprise clients. Rebuilding the SDN unblocks the product roadmap.
5. **Operational Manageability & Blast Radius:** At hyperscale, hardware statistically malfunctions or dies every minute. Modern SDNs are redesigned to limit the "blast radius." If a switch dies, the new architecture ensures it only impacts a strictly isolated micro-segment of the network.<label for="sn-1" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-1" class="margin-toggle"/><span class="sidenote">This principle of blast-radius reduction applies equally to modern Software-Defined Compute (SDC) and Software-Defined Storage (SDS) architectures.</span>
6. **Tenant Isolation and Security:** Early clouds were multi-tenant but heavily relied on software firewalls. As cyber threats evolved, rebuilding the control plane was necessary to implement Zero Trust architectures natively at the packet level.

### Why VK Cloud Replaced Its Foundational Architecture

As discussed in my previous article, the catalyst for VK Cloud was the "200k port" Neutron disaster. Our cloud had grown to a scale where the central RabbitMQ message bus could no longer synchronize the dataplane state across 3,000 hypervisors, resulting in catastrophic "Full Sync" timeout loops during datacenter recoveries.

The engineering solution was VK Cloud's proprietary SDN, "Sprut." By abandoning the fragile message queue in favor of a declarative HTTP REST architecture, the performance gains were massive. Our benchmarks showed a 65% speedup in full network creation, an 87% speedup in network removal, and an 84% speedup in mass deletion (a critical metric for clients running automated Terraform CI/CD environments).

### The Hyperscaler Playbook: Externalizing the Risk

When global hyperscalers execute these foundational upgrades, their standard playbook is to externalize the labor and risk. To be fair, at their scale, there is rarely another option. They set a date, provide a script, and tell the customer: *"Figure it out, or lose support."* Here are historical cases of hyperscalers handling major migrations similar to VK Cloud's:

1. **AWS: The Retirement of EC2-Classic:** In August 2021, AWS announced the deprecation of their original flat network, EC2-Classic. While they provided an automation runbook, AWS explicitly required customers to hunt down their legacy resources, run the scripts, rewrite security groups, and absorb the downtime themselves. There was no managed tool for smoothly replacing a VM's underlying network without causing that VM to become unavailable for an extended period.
2. **Google Cloud: Legacy Networks to VPCs:** GCP's deprecation of their subnet-less "Legacy Networks" is perhaps the starkest contrast to a managed migration. Because legacy networks spanned multiple regions by default, Google's official documentation literally states: *"There is no automated solution to convert multiple regions... Recreate your resources with the same configurations... Delete your old resources."* Enterprise customers were told to build a new network from scratch. For a large enterprise, this is incredibly problematic because different departments typically have varying subnetwork architectures. If the legacy network isn't perfectly documented (which is almost always the case), it is almost impossible to execute this manually without errors.
3. **Azure: ASM to ARM Control Plane Shift:** Azure’s shift from Azure Service Management (Classic) to Azure Resource Manager (ARM) forced a massive architectural change. Customers were given hard deadlines to migrate via PowerShell workflows. Crucially, any legacy automation scripts written for the ASM API became useless and had to be completely rewritten into ARM JSON templates.
4. **AWS: Xen to Nitro Hypervisor Swap:** Moving from legacy Xen to the hardware-offloaded Nitro system required deep OS-level changes. The AWS migration runbook details brutal prerequisites: updating ENA network drivers, editing `/etc/fstab` to use UUIDs, and modifying GRUB boot parameters. If a client's custom Linux kernel wasn't prepared, the migration simply failed.

The consistent thread across all of these cases is the transfer of operational burden. The downtime, the troubleshooting of edge cases, and the overall migration strategy development were dumped squarely onto the clients' Operations teams.

### The Architectural Evolution: Flat Networks to Modern SDNs

A common thread across the hyperscaler cases described above is the replacement of flat networks and aging control plane components. (VK Cloud also used to have a flat network before Neutron, though that transition occurred before I joined the team).

It is worth noting that the reason all hyperscalers started with flat networks was not a lack of experience. Cloud is a capital-intensive business requiring massive hardware investments that become obsolete in just a few years. In this environment, time-to-market and simplicity were paramount. In the early days of the cloud, engineers simply replicated what they knew: physical data centers were mostly flat. SDN was still an emerging paradigm. Overlay networks (like L2 over L3) were not prevalent, and encapsulation was slow and required heavy CPU usage. Running a flat network on physical switches, however, was hardware-accelerated and lightning-fast. 

Building a system that can dynamically create thousands of isolated virtual routers and subnets across thousands of physical hosts is an incredibly hard distributed systems problem. In the "Move Fast and Break Things" era, a flat network was the path of least resistance. 

There were distinct advantages to these older networking solutions. They offered ultra-low latency with no encapsulation overhead, and debugging was simple: if a packet wasn't moving, it was a standard L2 issue. However, the disadvantages ultimately proved fatal at scale:
* **The "ARP Storm":** As a cloud hits 100,000+ VMs, the amount of broadcast traffic becomes so dense that the network spends all its time processing background noise instead of actual data.<label for="sn-2" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-2" class="margin-toggle"/><span class="sidenote">An ARP storm occurs when a massive number of devices simultaneously broadcast Address Resolution Protocol requests across a flat L2 network, saturating the switches and crippling overall network throughput.</span>
* **Security Risks:** If a tenant managed to spoof a MAC address or ARP response, they could potentially intercept traffic from other tenants on the same flat segment.
* **Geographic Limits:** You cannot stretch a flat L2 network across the globe without it breaking.

Modern SDNs (like OVN and Sprut) were designed to mitigate these exact issues at hyperscale. The advantages are clear: infinite scale allowing millions of tenants with overlapping IP ranges, and deep programmability allowing entire network topologies to change via a single API call without touching a physical wire. Live migration also became seamless; because the network is software, you can move a VM from one physical rack to another and it keeps its IP address because the overlay simply updates its tunnel map. 

However, modern SDNs also introduced new complexities:
* **MTU Issues:** Because you are wrapping a packet inside another packet for the overlay tunnel, the inner packet has to be smaller, causing MTU (Maximum Transmission Unit) fragmentation overhead.<label for="sn-3" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-3" class="margin-toggle"/><span class="sidenote">We once faced a severe MTU-related bug with a direct L2 channel established between a VK Cloud datacenter in Moscow and an AWS datacenter in Stockholm, which I will detail in a future article.</span>
* **Control Plane Reliance:** If the SDN control plane hangs, the entire network can go "blind," even if the physical wires are perfectly fine.

### The VK Cloud Difference: A Partnership Approach

As noted earlier, international hyperscalers operate at a magnitude that makes personalized migrations nearly impossible. When AWS shut down EC2-Classic, they were coordinating the migration of 15 years of technical debt across millions of active servers. At that scale, automated self-service is the only viable path.

However, for a regional hyperscaler like VK Cloud (managing several data centers across Russia and Kazakhstan), telling a massive enterprise client to "recreate your resources on your own" is unacceptable.

We chose a partnership approach. We built automated backend pipelines to migrate smaller tenants invisibly. But for our VIP clients-which included global giants like Philip Morris International and top-tier EdTech platforms like Uchi.ru-we engineered surgical, transparent migration tools.

My team and I sat directly with their engineers. We developed the [Public API-based tooling](https://github.com/vk-cs/neutron-2-sprut/tree/main), hosted production webinars, and even built [Terraform sandbox environments](https://github.com/vk-cs/neutron-2-sprut/tree/main/examples) so clients could safely test the migration on dummy infrastructure before touching production. We gathered their feedback, refined the tools in real-time, and fed that data directly back to our internal engineering teams to perfect the mass-migration tool for the rest of the cloud.

This migration required late-night maintenance windows, manipulating Terraform state files, and convincing CTOs that a brief 20-second network disconnect for their VMs was a necessary trade-off for the long-term survival and speed of their infrastructure.


## II. The Philosophy of Migration: State Machines vs. Database intervention

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
