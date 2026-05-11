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
    <strong>Fig. 3.</strong> Basic cloud networking port migration.
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
    <strong>Fig. 4.</strong> SDN Migration decision matrix.
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

## III. The Server-Level Migration: Hot-Swapping the Dataplane

While the Client-level migration utilizing the Northbound API was the safest route for our VIPs, it required active coordination and client engineering effort. For the thousands of smaller tenants remaining in the cloud, we needed a completely invisible backend solution. This is the Server-level migration. 

The core concept of a Server-level migration is based on cloning the client's network infrastructure from the source SDN (Neutron) to the destination SDN (Sprut) in a 1-to-1 ratio. We temporarily create duplicates while preserving the original values of all significant parameters, most importantly the OpenStack UUIDs. By bypassing the standard public APIs, we ensure the guest virtual machines-and the clients managing them-notice absolutely nothing other than a brief network disconnect.

### Mapping Migration to the Two Jobs of SDN

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/server-level-access-to-control-and-data-planes.png" alt="Server-level Migration Access Map" />
  <figcaption>
    <strong>Fig. 5.</strong> Server-level migration access across the Control and Data planes.
  </figcaption>
</figure>

In the previous article, we established that a modern cloud SDN performs two primary jobs: Entity Management (Control Plane) and Network Function Virtualization (Data Plane). The Server-level migration is cleanly split into two corresponding phases:

1. **Logical Migration:** This handles the Entity Management. It involves cloning the cloud entities (Networks, Subnets, Ports, Security Groups) directly in the databases.
2. **Physical Migration:** This handles the Data Plane. It involves mechanically rewiring the virtual machine's network interface on the bare-metal hypervisor.

As shown in Figure 4, executing this requires Admin-level access and direct database manipulation (e.g., via SQL CLI), completely bypassing the standard Northbound API. One of the major advantages of this approach is speed; by interacting directly with the databases and network nodes, we bypass the slow RabbitMQ RPC queues that bottleneck legacy Neutron.<label for="sn-7" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-7" class="margin-toggle"/><span class="sidenote">Avoiding the RPC queue provides a massive speedup, but it is inherently dangerous. Manual database manipulation risks breaking the control loop, corrupting the synchronized state, or leaving the hypervisor with invalid local configurations.</span>

### The Logical Migration

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/server-level-access-to-control-plane.png" alt="Logical Migration Access" />
  <figcaption>
    <strong>Fig. 6.</strong> Logical migration directly accesses private APIs and SQL databases.
  </figcaption>
</figure>

The logical migration phase focuses entirely on the cloud databases. Because an SDN encompasses much more than just VM ports-managing routers, firewall rules, and metadata-the backend script (`sdn_cp_migrator`) must perfectly clone these relationships. The process consists of three main tasks:

**1. Analysis of Resources:** The utility scans the source SDN to determine if migration is possible. Because Server-level migration is an irreversible process, we cannot perform partial migrations (e.g., migrating only half of a subnet). Once a tenant is migrated, legacy Neutron support for that tenant is permanently blocked. Due to this irreversibility, L2 support engineers often create a mock copy of the client's topology to perform a "dry run" rehearsal before touching production.

**2. Preparation and Transfer:**
The script injects the cloned state into the Sprut database. A critical challenge during this phase was the migration of Floating IPs (Public IPs). You cannot simply move a public IP from a Neutron edge router to a Sprut edge router without causing severe BGP routing downtime. Previously, we relied on DNS round-robin hacks, but for this migration, we developed a specialized internal REST API handle that allowed administrators to seamlessly transfer the Floating IP bindings between the two SDNs at the backend level.

**3. Deletion in the Source SDN:**
Once the transfer is verified, the logical entities in the Neutron database are purged to prevent the control plane from managing duplicate states.

### The Physical Migration: The QEMU/KVM Dataplane

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/neutron_sprut_physical_path.png" alt="Dataplane Comparison" />
  <figcaption>
    <strong>Fig. 7.</strong> The physical network paths of a VM in Neutron vs. Sprut.
  </figcaption>
</figure>

With the databases cloned, we must perform the physical migration. This involves dropping down to the Southbound interface (the Linux shell of the hypervisor) to manually hot-swap the network cables. To understand how we do this without shutting down the VM, we must look at how a virtual machine connects to the network in a QEMU/KVM environment.

When an application inside a VM sends data, the guest OS encapsulates it into an Ethernet frame and pushes it to its virtual driver. QEMU intercepts this write operation in the guest's memory space and injects it into a `tap` interface on the host machine. 

From the `tap` interface, the paths diverge depending on the SDN (as seen in Figure 6):
* **Neutron's Path:** The frame enters a small Linux bridge (`qbr`). Here, the host kernel applies `iptables` rules to enforce OpenStack Security Groups. It then traverses a virtual patch cable (`veth` pair) into the Open vSwitch integration bridge (`br-int`), where it is tagged and sent to the tunnel bridge.
* **Sprut's Path:** The architecture is significantly simpler. Sprut drops the intermediate Linux bridge entirely. The `tap` interface plugs directly into the Sprut OVS bridge (`br-sprut`).<label for="sn-8" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-8" class="margin-toggle"/><span class="sidenote">Neutron required the intermediate Linux bridge because older versions of OVS did not natively support stateful firewalls. Sprut utilizes modern OVS connection tracking (conntrack), allowing security groups to be enforced directly within the OpenFlow tables.</span>

### Step-by-Step: Hot-Swapping the Port

The physical migration (`sdn_dp_migrator`) executes the swap in three stages.

**Stage 1: Preparation**
<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/physical_migration_preparation.png" alt="Migration Preparation" />
  <figcaption>
    <strong>Fig. 8.</strong> The pre-created Sprut target state awaits connection.
  </figcaption>
</figure>
Thanks to the logical migration phase, the target Sprut OVS bridge and network topology already exist on the hypervisor. The VM is currently operating on the original Neutron path.

**Stage 2: The Switch**
<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/physical_migration_port_swap.png" alt="The Port Swap" />
  <figcaption>
    <strong>Fig. 9.</strong> Disconnecting the tap interface and binding it to Sprut.
  </figcaption>
</figure>
The script executes a sequence of strict Southbound commands:
* **2.1:** It forcefully unplugs the VM's `tap` interface from the legacy Linux bridge. (Network connectivity drops here).
* **2.2:** It instantly plugs the `tap` interface into `br-sprut`.
* **2.3:** It calls the Sprut API to officially bind the port to the VM in the control plane.
* **2.4:** It calls the Neutron API to unbind the old port. (Network connectivity is restored).

**Stage 3: The Cleanup**
<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/physical_migration_dataplane_cleanup.png" alt="Neutron Dataplane Cleanup" />
  <figcaption>
    <strong>Fig. 10.</strong> Disassembling the legacy Neutron virtual interface.
  </figcaption>
</figure>
With the VM successfully passing traffic over Sprut, the script cleans up the host hypervisor. It detaches the `veth` pairs (3.1, 3.4), disables and deletes the Linux bridge (3.2, 3.3), and completely removes the virtual cables (3.5).

### The Inconsistent State (The Libvirt Paradox)

If we stop after Stage 3, the packets flow perfectly, but we have created a ticking time bomb. 

While we successfully hot-swapped the network on the hypervisor, the VM's persistent `libvirt` XML configuration file (which defines the VM's hardware for QEMU) still believes the VM is connected to the old Neutron Linux bridge. The server is now in an inconsistent state: the logical database says Sprut, the physical dataplane says Sprut, but the hypervisor's local definition says Neutron. If the VM were to be hard-rebooted, it would attempt to connect to a Linux bridge that no longer exists, resulting in a fatal crash.

We cannot simply edit the `libvirt` XML file manually without risking severe hypervisor corruption. The only safe way to resolve this inconsistency is to perform a **Live Migration**. By instructing Nova to live-migrate the VM to a different bare-metal host, Nova is forced to read the *new* database state and generate a fresh, accurate `libvirt` XML file on the destination host, fully finalizing the transition.

### Downtime Expectations

Despite operating on the backend, Server-level migration still incurs downtime, split into two distinct windows:
1. **Downtime 1 (The Dataplane Swap):** The time between unplugging the `tap` interface from Neutron and the Sprut OpenFlow rules fully propagating. This typically lasts 10 to 30 seconds.
2. **Downtime 2 (The Live Migration):** The standard memory cutover time required by QEMU when moving the VM between physical hosts to fix the `libvirt` XML.

### The Server-Level Migration Algorithm

To successfully safely manage this across thousands of tenants, the complete Server-level migration utility follows this strict algorithmic checklist:

1. **State Retrieval:** Queries the source API to download the complete state of the client's project.
2. **Health Check:** Validates the state for known issues (e.g., VMs in transitioning states or missing gateway IPs).
3. **Soft Limitation Warning:** Flags non-critical limitations and prompts the operator to accept the risks.
4. **State Compilation:** Compiles a clean Target State file suitable for import into the Sprut database.
5. **Sprut Discovery:** Analyzes the current state of the project in the Sprut SDN (if any prior resources exist) and backs it up.
6. **External Network Mapping:** Identifies external subnets to correctly remap Floating IPs and external gateways.
7. **Execution Plan Generation:** Compiles and outputs the strict sequence of transfer operations.
8. **Anomaly Detection:** Halts and alerts the operator if unknown entities exist in the Sprut target destination.
9. **Conflict Resolution:** Alerts the operator if UUID conflicts are detected between Neutron and Sprut.
10. **Logical Execution:** Commits the migration to the databases. If the default security group ID differs, it forcefully recreates it to match the Neutron UUID.
11. **Retry Mechanism:** If the logical migration drops a connection, the script is fully idempotent and can be safely re-run with the same arguments.
12. **Physical Execution:** Triggers the `sdn_dp_migrator` to perform the Southbound `tap` interface hot-swap.
13. **Consistency Enforcement:** Triggers the Nova Live Migration to rewrite the `libvirt` XML and finalize the transition.

## IV. The Client-Level Migration: Safe Tooling for VIPs

While the internal backend tools were efficient for smaller workloads, the sheer complexity of our VIP clients' environments demanded a different approach. We needed surgical tools that respected the OpenStack state machine, utilized the public APIs, and could be audited and executed directly by the client's engineering teams. 

All the tools, scripts, and Terraform test stands developed for this project are available in the public [VK Cloud neutron-2-sprut repository](https://github.com/vk-cs/neutron-2-sprut/tree/main). 

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/client-level-migration-arch.png" alt="Client-level Migration API Access" />
  <figcaption>
    <strong>Fig. 11.</strong> Client-level migration interacts strictly with the Northbound REST API, preserving validation and state checks.
  </figcaption>
</figure>

### The Tooling Choice: Why Bash and CLI?

The first architectural decision was choosing the right language for the migration tools. OpenStack is written in Python, and its Python SDKs (`openstacksdk`) are powerful and expressive. A standard engineering instinct would be to write the migration utility as a comprehensive Python application. We explicitly chose not to do this.

Instead, I developed the entire toolset using Bash, standard OpenStack CLI commands, `curl`, and `jq` (for JSON processing). 

This decision was driven by client friction. This tool was designed to be run by the client's DevOps engineers under the scrutiny of their InfoSec teams. Almost all engineers operating within an OpenStack cloud are already intimately familiar with standard OpenStack CLI commands. Providing a Python application meant forcing the client to review thousands of lines of unfamiliar code, manage virtual environments, resolve Python version discrepancies, and handle dependency conflicts on their jump hosts.<label for="sn-9" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-9" class="margin-toggle"/><span class="sidenote">In enterprise environments, introducing new Python dependencies to a hardened jump host often requires InfoSec approval, which can delay a critical maintenance window by weeks. Bash and Curl are universally available.</span>

Bash scripts utilizing standard CLI commands are transparent. A client engineer can read the script, instantly understand the API calls being made, and trust the execution. There was one caveat: because the OpenStack CLI was not originally designed to manage overlapping SDNs natively, some resources (like creating a base Network) defaulted to Neutron. To bypass this, we used `curl` to make direct REST API calls for specific Sprut entity creation, while relying on the CLI for everything else.

### The Core Mechanics: Migrating the VM Port

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/sequence_of_swapping_ports.png" alt="Port Migration Logic" />
  <figcaption>
    <strong>Fig. 12.</strong> The sequence of swapping a virtual interface.
  </figcaption>
</figure>

The foundation of the migration is the virtual machine port swap. One of the greatest advantages of OpenStack is the ability to create a networking port with a specifically requested MAC and IP address. This capability is the linchpin of our strategy, as it allows us to swap the underlying network without forcing the guest OS to reconfigure its internal network interfaces (e.g., `netplan` or `systemd-networkd`). The only action required by the guest OS post-migration is a simple DHCP renewal (`dhclient`).

We iterated through several versions of this core script. The initial proof-of-concept, [`migrator.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/migrator.sh), was interactive, prompting the user for the VM name and destination subnets. While it proved the methodology was safe, interactivity is an anti-pattern for mass automation.

The production version, [`migrator-multiple.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/migrator-multiple.sh), accepts a CSV configuration file and processes VMs sequentially. Crucially, it includes idempotency checks. If a migration fails mid-execution, the pre-created ports are not deleted; the script can simply be re-run, detect the existing resources, and pick up where it left off.

Here is the core logic sequence executed by the script:

```bash
# 1. Capture original MAC and IP
capture_info_full 

# 2. Create the new port on Sprut with the exact same MAC/IP
create_port_with_mac_ip 

# 3. Disconnect the old port from the Nova instance
detach_source_port 

# 4. Attach the newly created Sprut port
attach_new_port 

# 5. Apply original Security Groups and Floating IPs to the new port
set_security_groups
attach_floating_ip
```

While most IaaS VMs only have a single port, specialized instances (like Next-Generation Firewalls or virtual routers) often possess multiple interfaces. For these complex, multi-port edge cases, we recommended that clients execute the migration manually via the CLI to ensure proper interface ordering and routing preservation.

### Dependency Graphing: Migrating Complex Services

Migrating flat networks and basic VMs is trivial. The true complexity of a client-level migration emerges when dealing with specialized services like VPNaaS (IPsec) and LBaaS (Load Balancers). You cannot simply "copy" an IPsec tunnel.

To automate the migration of a complex service, you must execute strict **Dependency Graphing**.

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/entity_relation_diagram_ipsec.png" alt="IPsec Dependency Graph" />
  <figcaption>
    <strong>Fig. 13.</strong> Mapping the hierarchical dependencies of an IPsec Site Connection.
  </figcaption>
</figure>

As illustrated in the Entity Relation Diagram (Figure 13), when you query the API for an `IPSEC SITE CONNECTION`, the response doesn't just contain string fields (like the Pre-Shared Key). It also contains IDs referencing other discrete OpenStack entities (e.g., `ikepolicy_id`, `ipsecpolicy_id`, `local_ep_group_id`, `vpnservice_id`). Those child entities must be queried, and they often contain their own dependencies. 

If you attempt to create the top-level IPsec tunnel on the destination SDN before creating its child policies, the API will reject the request. Therefore, the migration algorithm must collect data top-down, but execute creation bottom-up.

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/ipcsec-migration-stages.png" alt="IPsec Migration Algorithm" />
  <figcaption>
    <strong>Fig. 14.</strong> The 4-stage algorithm for migrating complex services (IPsec example).
  </figcaption>
</figure>

This dependency logic forms a universal 4-stage architecture used in almost all of our service migration scripts, as demonstrated in [`copy-ipsec-v2.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/copy-ipsec-v2.sh) and Figure 14:

1. **Stage 1: Collection.** Query the source Neutron entities and all hierarchical children.
2. **Stage 2: Target Discovery.** Query the Sprut environment to see if any of these entities have already been created (preventing duplicates and conflicts).
3. **Stage 3: Delta Creation.** Create the missing child objects (IKE policies, Endpoint groups) in Sprut.
4. **Stage 4: Final Assembly.** Create the top-level IPsec connection binding the new child objects together.

```bash
# Example from copy-ipsec-v2.sh: Stage 4 Assembly via REST
curl_response=$(curl -s -X POST "${sprut_api_base}/vpn/ipsec-site-connections" \
    -H "Content-Type: application/json" \
    -H "X-SDN:SPRUT" \
    -d '{
          "ipsec_site_connection": {
              "psk": "'$psk'",
              "ikepolicy_id": "'$ikepolicy_id'",
              "ipsecpolicy_id": "'$ipsecpolicy_id'",
              "peer_address": "'$peer_address'",
              "name": "'$name'"
          }
        }')

```

Arguably the most complex entity to map was the Load Balancer (LBaaS). OpenStack Octavia load balancers are provisioned as active/standby "Amphora" VMs. Because spinning up these VMs takes time, we split the logic into two scripts. First, [`copy-loadbalancer.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/copy-loadbalancer.sh) pre-provisions the heavy Amphora instances on Sprut days before the maintenance window. Later, [`copy-loadbalancer-rules.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/copy-loadbalancer-rules.sh) is executed during the downtime to instantly attach the migrated backend VMs to the new Sprut load balancer pools.

### Platform Services (PaaS)

Migrating PaaS instances (like Managed Kubernetes or DBaaS) presents a unique challenge. Because PaaS orchestration engines (like Magnum or Trove) maintain their own state databases that synchronize with Nova and Neutron, manually swapping the underlying network ports via scripts immediately corrupts the PaaS controller's state.

To migrate PaaS safely, the client must use the native PaaS backup/restore tools (e.g., Velero for Kubernetes) to spin up a completely new instance directly on the Sprut network. 

<figure class="center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/paas_migration.png" alt="PaaS Migration Networking Scenario" />
  <figcaption>
    <strong>Fig. 15.</strong> Resolving connectivity when PaaS instances and VMs reside in the same subnets.
  </figcaption>
</figure>

This introduces a routing dilemma, highlighted in Figure 15. If a group of VMs and a PaaS instance share the same legacy subnet, moving them asynchronously breaks connectivity. If half of a client's project (the VMs) is migrated to Sprut, and the other half (the PaaS cluster) remains on Neutron, how do they communicate? 

VK Cloud’s "Advanced Router" utilizes BGP peering to seamlessly bridge Sprut and Neutron subnets, allowing partial project migrations.<label for="sn-10" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-10" class="margin-toggle"/><span class="sidenote">A router cannot bridge two networks possessing the exact same IP address space. To utilize cross-SDN routing during a partial migration, the target Sprut network must be provisioned with a different CIDR block than the original Neutron network.</span> By establishing static routing on the bridge, we allowed clients to decouple their VM migrations from their heavy PaaS rebuilds, mitigating the risk of a massive "all-at-once" cutover.

### The IaC Trap: Fixing Terraform State

The final hurdle of the Client-level migration was Infrastructure-as-Code (IaC).

For massive clients like Uchi.ru, deleting and recreating thousands of resources via Terraform was operationally impossible. While our scripts successfully migrated the physical network, Terraform was completely unaware of the changes. If a client ran `terraform apply` after our scripts finished, Terraform would detect that the VM was no longer attached to the Neutron port defined in its code, and it would violently attempt to recreate the VM to "fix" the discrepancy.

Terraform, like the cloud itself, is a state machine. To prevent catastrophic recreations, we had to manually manipulate the `.tfstate` file to force it to recognize the new Sprut UUIDs.

To automate this surgery, I wrote [`modify_terraform_state.sh`](https://github.com/vk-cs/neutron-2-sprut/blob/main/modify_terraform_state.sh). The client updates their `.tf` files to declare `sdn = "sprut"` on their network resources. The script then executes a sequence of `terraform state rm` (to untrack the old Neutron ports) and `terraform import` commands to pull the newly generated Sprut UUIDs directly into the local state file.

```bash
# Removing the legacy tracking
terraform state rm vkcs_networking_port.port_1

# Importing the new Sprut UUID into the state file
terraform import vkcs_networking_port.port_1 <NEW_SPRUT_PORT_UUID>
```

By manually updating the state file, the client regains total declarative control over their newly migrated Sprut infrastructure, completing the migration cycle with zero loss of automation.

### The Comprehensive Migration Runbook

While building the underlying automation scripts is a massive engineering hurdle, safely executing them across 200,000 ports requires strict operational discipline. You cannot rely on ad-hoc decision-making during a live infrastructure swap. Just as technical support uses strict runbooks to resolve incidents, we needed to design a definitive "Workbook" for the client-level migration. Each step needed to be clear, sequential, and entirely unambiguous so that any vendor engineer or client DevOps team could follow it flawlessly.

<figure class="fullwidth center-caption">
  <img src="/assets/img/2026-04-27-migrating-200-000-network-ports/comprehensive_sdn_migration_diagram.png" alt="Comprehensive OpenStack Tenant Migration Workflow" />
  <figcaption>
    <strong>Fig. 16.</strong> The end-to-end operational runbook for migrating a complete OpenStack tenant.
  </figcaption>
</figure>

To achieve this, we developed the comprehensive flowchart shown in Figure 16, categorizing the entire migration lifecycle into four distinct, color-coded stages:

* **White (Planning & Inventory):** This is the foundational prep work. No scripts are executed here. Teams generate configurations, assess the tenant's inventory, verify quotas, and plan the migration strategy.
* **Blue (Zero-Downtime Preparation):** This stage can be executed days or weeks in advance. Using the dependency-graphing scripts discussed earlier, engineers copy networks, subnets, routers, security groups, empty LBaaS instances, and IPsec tunnels to the Sprut SDN. Because these operations only create parallel infrastructure, they cause absolutely zero downtime to the active Neutron environment.
* **Green (Specialized PaaS/Custom Prep):** If the tenant utilizes DBaaS, KaaS, or highly customized deployments, these clusters are rebuilt and synchronized on the newly created Sprut networks during this phase, utilizing the BGP bridging strategies mentioned above.
* **Red (The Maintenance Window):** This is the critical downtime event. The operations team executes `migrator-multiple.sh` to forcefully hot-swap the VM ports, runs `copy-loadbalancer-rules.sh` to attach the newly migrated VMs to the Sprut load balancers, and finally runs `modify_terraform_state.sh` to update the client's IaC backend. 

By strictly adhering to this color-coded runbook, we decoupled the complex, time-consuming API creation tasks (Blue) from the high-stress port swapping tasks (Red). This ensured that when the actual maintenance window opened, the only thing left to do was flip the switch.

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
