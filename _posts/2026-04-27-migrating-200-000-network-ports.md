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


## II. The Philosophy of Migration: State Machines vs. Database Intervention

Before writing a single script or scheduling a maintenance window, we had to determine the architectural philosophy of the migration. When moving 200,000 virtual ports, the decision of *how* to interact with the cloud-via public APIs or through direct database manipulation-dictates the entire risk profile of the project.

### The Cloud as a State Machine

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/cloud_state_machine.png" alt="Cloud state machine" />
  <figcaption>
    <strong>Fig. 1.</strong> The cloud as a State Machine which synchronizes Cloud State and physical infrastructure.
  </figcaption>
</figure>

At its core, a cloud environment can be viewed as a massive state machine. As shown in the simplified model above, external users interact with a Public API to declare their desired infrastructure state (e.g., "Create a network with this CIDR block"). The Cloud Control Plane accepts these requests, writes the "Target State" to the central database, and continuously works to synchronize the physical hardware with that database.

The safest way to change the state of physical infrastructure is strictly through the Public API. When a request comes through, the Control Plane executes a list of safety checks: it verifies quotas, checks for physical resource availability, validates parameter formatting, and prevents security exploits. This rigorous validation is what guarantees the reliability of the cloud and ensures the database remains uncorrupted. 

However, safety comes at a cost. At hyperscale, executing millions of API calls and waiting for validation checks takes significant time. To speed up operations, engineers are often tempted to bypass the Public API and interact directly with the physical infrastructure or private, unrestricted administrative APIs. While faster, this direct intervention removes the safety net, significantly increasing the risk of state corruption.

### Mapping Migration to the SDN Layers


<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/sdn_layers.png" alt="SDN Layered Diagram" />
  <figcaption>
    <strong>Fig. 2.</strong> Software-Defined Networks in planes and layers.
  </figcaption>
</figure>

This state machine philosophy maps directly onto the architecture of a Software-Defined Network. As established in the previous article, SDN separates the Control Plane from the Data Plane. Crucially, it introduces different levels of access:

* **Northbound Interfaces (The Safe Path):** This is the Neutron REST API. It is the highest level of abstraction, used by users and network applications. It is slow but mathematically safe.
* **Southbound Interfaces (The Hardware Path):** This is the protocol (like OpenFlow or OVSDB) used by the controller to configure the physical hardware, such as the Open vSwitch (OVS) on bare-metal hypervisors. Users and administrators are generally not supposed to send requests directly to the Southbound interface.<label for="sn-4" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-4" class="margin-toggle"/><span class="sidenote">Bypassing the Northbound API to manually inject rules into a Southbound interface creates an immediate desync between the physical reality of the hypervisor and the logical "Target State" stored in the central database.</span>

When deciding how to migrate the network, we had to choose which interface level to target.

### The Universal Mechanics of an SDN Migration

Before detailing the specific execution paths, it is crucial to understand what "migrating" virtual machines to a new network actually means. 

Despite existing in the same cloud, Neutron and Sprut do not share a common backend. Each SDN is an entirely isolated service with its own database, its own resource space, and its own operational principles. Therefore, you cannot simply update a database flag to transition a network from Neutron to Sprut. 

Regardless of whether you use the Northbound API or Southbound database hacks, the fundamental physical reality of the migration requires a three-step cloning process:

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/basic_port_migration.png" alt="Basic port migration" />
  <figcaption>
    <strong>Fig. 4.</strong> Basic port cloud networking port migration.
  </figcaption>
</figure>


1.  **The Original State:** The virtual machine is actively connected to the legacy Neutron network, serving traffic.
2.  **Cloning the Target State:** A parallel, isolated duplicate of the network infrastructure is created in the Sprut SDN. This includes matching subnets, security groups, virtual routers and routing rules.
3.  **The Port Swap & Cleanup:** The VM's virtual network interface is disconnected from Neutron and forcefully reattached to the newly created Sprut network. This is the only moment during the entire process where network connectivity is lost. Once the VM is successfully operating on Sprut, the legacy Neutron resources are deleted.

While this three-step physical sequence remains the same, the *method* used to execute it changes drastically depending on which SDN interface layer we target.

### The Three Paths of Migration

Based on the access levels described earlier, We defined three internal paths for migrating OpenStack tenants to the Sprut SDN.<label for="sn-5" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-5" class="margin-toggle"/><span class="sidenote">In OpenStack, a "Tenant" (or Project) is an isolated set of cloud resources, identical to an AWS Account or a GCP Project. Migration was executed strictly on a tenant-by-tenant basis.</span> It is important to note the "-level" postfix here: we are not migrating servers; we are migrating *networks* at different operational levels.

1.  **Client-Level Migration:** A set of actions performed exclusively through the standard, existing Northbound API (e.g., recreating a virtual router, requesting a new port, reconnecting a VM interface). This approach is the most natural, predictable, and secure. However, because new entities are created via the API, the OpenStack UUIDs of those network resources will change. The guest operating systems must adapt to the switch (e.g., renewing DHCP leases), which often requires coordination with the client.
2.  **Server-Level Migration:** A process performed entirely on the backend by operations engineers. It utilizes service-level access, direct queries to private administrative APIs, and low-level Southbound configurations. The primary goal is for the guest VMs to "notice nothing"-UUIDs remain the same, and no client involvement is required. In theory, this is the ultimate seamless migration. In practice, achieving this safely requires monumental, customized engineering effort that often exceeds the budget and timeline of a critical migration.
3.  **Hybrid-Level Migration:** A blend of the two, utilizing Client-level API scripts for entity creation, but assisted by specific Server-level tools to bypass certain API limitations.

### The Decision Matrix and Execution

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/migration_decision_matrix.png" alt="SDN Layered Diagram" />
  <figcaption>
    <strong>Fig. 3.</strong> SDN Migration decision matrix.
  </figcaption>
</figure>

With a massive cloud footprint and limited engineering resources, we had to prioritize ruthlessly. We developed a strict decision matrix (illustrated above) to determine exactly how and when an OpenStack tenant would be migrated. 

**Step 1: Analysis and Prioritization**
We started by listing and ranking all existing OpenStack tenants. Our primary goal was to reduce the "blast radius" of a potential Neutron Full Sync disaster as quickly as possible. The fewer ports left on legacy Neutron, the faster the recovery time for everyone. Therefore, we prioritized the largest projects and our highest-paying VIP clients. 

**Step 2: Checking for Specialized Entities**
We systematically analyzed each tenant for complex services. VIP clients rarely use simple IaaS (Infrastructure as a Service) VMs. They utilize complex PaaS environments, DBaaS, shared networks, VPNaaS, Floating IPs, and highly customized solutions. The presence of these entities immediately dictated the technical limitations of the migration.

**Step 3: Choosing Priority and Migration Level**
Based on the analysis, we assigned the tenant to a priority queue (first, second, third) and selected the migration branch. The presence of complex services forced the routing between Server-Level and Client-Level migration. 

For the vast majority of our VIP clients, **Client-Level Migration** was the only viable choice, offering distinct advantages:
* **Total Feature Support:** It seamlessly handles DBaaS, PaaS, shared networks, and custom solutions because it relies on native API creation.
* **Architectural Control:** The process was entirely guided by a Solution Architect working directly with the customer’s engineering team.
* **The Disadvantage:** It introduces friction. It requires convincing a CTO to allocate their engineering resources for an infrastructure upgrade. Additionally, changing UUIDs requires clients to update their Infrastructure-as-Code (Terraform) states.

Conversely, **Server-Level Migration** was utilized primarily for smaller, simpler tenants. 
* **The Advantage:** It is incredibly fast, preserves all OpenStack UUIDs (saving Terraform configurations), and requires zero involvement from the end-user.
* **The Disadvantage:** It explicitly breaks if the tenant uses PaaS or shared networks, making it unsuitable for complex enterprise environments.

**Step 4: Migration Preparation**
Preparation diverged entirely depending on the chosen path.
* **Server-Level Preparation:** We allocated an internal L2/L3 support engineer, scheduled a downtime window, and notified the Solution Architect if the tenant belonged to a VIP client.
* **Client-Level Preparation:** Preparation was highly collaborative. The customer’s InfoSec team audited our open-source migration scripts. We then trained the customer's engineers on how to execute the tools, allocated a Solution Architect to oversee the process, and agreed on a strict downtime window. To minimize that window, our scripts pre-created all necessary Sprut subnets, security groups, load balancers, and IPsec tunnels days in advance without impacting the live environment.

**Step 5: Migration Execution and Troubleshooting**
* **Server-Level Execution:** L2/L3 engineers autonomously executed the migration using backend utilities, bypassing the Northbound API to preserve OpenStack UUIDs. If problems arose, they followed strict internal runbooks. Because this method manipulates the database directly, rolling back to Neutron in the event of a failure was strictly impossible.
* **Client-Level Execution:** The customer's engineer executed the migration steps. During the maintenance window, the only action required was the physical port swap-disconnecting the Neutron port and attaching the pre-created Sprut port. This method's greatest strength was its rollback capability. If an obscure application-level bug appeared after moving to Sprut, we could instantly roll the VM's port back to Neutron using the same scripts.<label for="sn-6" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-6" class="margin-toggle"/><span class="sidenote">This saved us during a rare edge case where Sprut initially mishandled the dual-port connection requirements (control vs. file ports) of legacy FTP traffic. We safely rolled the affected VMs back to Neutron while our SDN developers patched Sprut.</span>

While clients could theoretically run these API scripts in parallel to migrate dozens of VMs in a few seconds, most preferred a sequential, one-VM-at-a-time approach, taking about 20 seconds per server to allow for immediate health checks. Interestingly, as the script repository matured with comprehensive documentation, a shift occurred. Large enterprise clients managing dozens of OpenStack tenants-such as Philip Morris International-became increasingly independent. Once they grew comfortable with the methodology, they began executing migrations entirely on their own, and in some cases, built specialized internal tools based on our scripts. Eventually, simply handing over the repository link was enough for many to self-serve, though our Solution Architects always remained on standby to assist if needed.

Ultimately, every successful migration-regardless of the level chosen-accelerated our goal. It moved critical workloads onto a much faster, declarative SDN, while simultaneously reducing the port-count burden on the legacy Neutron database, making the cloud safer for everyone.

## III. The Server-Side Migration: Hot-Swapping the Dataplane

Focus: A deep-dive into how the internal SRE tool worked for the masses. (We explain this to show we understand the backend, contrasting it with your VIP solution).

Logical Migration (Control Plane): Hacking the OpenStack MySQL databases to clone the network (sdn_cp_migrator).

Physical Migration (Data Plane): Dropping down to the Southbound interface to manually unplug Linux bridges and plug into OVS.

The Libvirt Paradox: Explaining the inconsistency between the physical wiring and the VM's XML configuration.

The Fix: Using Nova Live-Migration to force the hypervisor to rewrite the XML and achieve consistency.

The two types of downtime (Southbound swap vs. Live Migration).

## IV. The Client-Side Migration: Surgical Tooling for VIPs

Focus: Your actual project. How you built the bash/CLI tools for the biggest clients.

Why Bash/CLI instead of Python (minimizing client friction and dependency hell).

### The Engineering Methodology: Dependency Graphing
The Dependency Graphing Methodology (Mapping Neutron fields to Sprut fields, figuring out the strict creation order).
* How to build a migration tool from scratch.

* API field mapping and determining the strict execution order.

* Why we chose Bash + OpenStack CLI over Python to reduce client friction.

### How ports swap

The 45-Second Disconnect Window: How the script actually swapped the ports.

The FIP and DNS workaround.


## V. Entity Caveats: IPsec, Advanced Routers

* The technical realities of the migration.

* Handling Security Groups with -sprut postfixes.  

* The IPsec / Advanced Router DNAT architecture changes.

* The 45-second VM disconnect/reconnect window.  

## VI. The Infrastructure-as-Code Trap: Fixing Terraform State

* You can't just delete and recreate a 5,000 VM deployment.

* How we allowed clients to adopt the new SDN without destroying their existing Terraform state files using terraform import

## VII. The Human Element: Selling Downtime to Angry CEO and CTOs

* The "Shoot the Messenger" dynamic.

* Translating technical debt into executive business value.

## VIII. Translating technical debt into executive business value.

* Key takeaways

* What I learned, and what I would do differently today.
