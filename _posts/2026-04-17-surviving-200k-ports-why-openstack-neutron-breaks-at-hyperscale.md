---
layout: post
title: "Surviving 200,000 Ports: Why OpenStack Neutron Breaks at Hyperscale"
---
In this post, we are going to explore Software-Defined Networking (SDN) architectures at hyperscale - specifically, what happens when you push a cloud environment past 160,000 VMs, 200,000 virtual ports, and roughly 3,000 bare-metal hypervisors. To put that scale into perspective, OpenStack Neutron was originally designed for private enterprise environments and historically recommended a maximum of around 500 hypervisors.<label for="sn-1" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-1" class="margin-toggle"/><span class="sidenote">While aggressive tuning of RPC workers and database pools can push this limit higher, 500 nodes is widely considered the historical threshold where the default RabbitMQ message bus begins to severely bottleneck under Neutron's architecture.</span>

But as VK Cloud<label for="sn-2" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-2" class="margin-toggle"/><span class="sidenote"><a href="https://cloud.vk.com/en/main/">VK Cloud</a> is the enterprise B2B cloud division of VK (formerly Mail.ru Group), offering infrastructure, platform services, and machine learning tools. Has at least 4 Tier 3 datacenters. </span> rapidly grew into one of the largest public cloud providers in the CIS region - establishing itself as the primary alternative to Yandex<label for="sn-3" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-3" class="margin-toggle"/><span class="sidenote"><a href="https://yandex.cloud/en">Yandex Cloud</a> is the largest public cloud platform in Russia and the CIS region, often compared to an AWS or GCP equivalent for that market.</span> - we blew past those private-cloud limits. We pushed the architecture as far as it could go, until it simply couldn't go any further.

I did not design that original OpenStack architecture, nor did I write the code for our new proprietary SDN, Sprut. I joined as a Lead Solution Architect and inherited a massive system that had already suffered a catastrophic "full sync"<label for="sn-4" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-4" class="margin-toggle"/><span class="sidenote">A "full sync" occurs when a local Neutron agent loses its connection state and requests the entire network topology from the central server. At hyperscale, hundreds of agents doing this simultaneously creates a "thundering herd" that crushes the control plane.</span> outage a few years prior. When the same disaster happened a second time (now on my watch), I spearheaded the initiative to migrate our highest-paying clients away from Neutron to our new SDN (I also consulted development team on automated migration solution for small clients).

To prove to our clients that the new system would keep them safe, I spent months studying documented and extracting undocumented wisdom from our SRE and maintenance teams. I dug deep into the theoretical and practical limits of our SDN to understand exactly why the old one failed, and how the new architecture would prevent it from ever happening again.

This article is a distillation of that unique, battle-tested knowledge. Before we talk about the caveats of live-migrating a hyperscale network (which I will cover in my next post), we need to understand the fundamental design flaws of legacy systems like neutron and what are modern alternatives like OVN<label for="sn-5" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-5" class="margin-toggle"/><span class="sidenote"><a href="https://www.ovn.org/en/">OVN (Open Virtual Network)</a> replaces legacy Neutron Python agents with lightweight, C-based daemons and uses OVSDB for distributed state management, significantly improving scaling capabilities.</span> and Sprut. Here is exactly why OpenStack Neutron breaks at hyperscale, and how modern SDNs solve the bottleneck.

<br>

## Table of Contents
* [I. Reality of building scalable clouds](#i-reality)
* [II. The Anatomy of a Cloud SDN](#ii-anatomy)
* [III. Neutron architecture: Design and Scaling Considerations](#iii-neutron)
* [IV. The "Full Sync" Disaster](#iv-disaster)
* [V. The Architectural Shift: Moving to the Modern Models](#v-shift)
* [VI. Solving the Full Sync Disaster](#vi-solving)
* [VII. Conclusion and key takeaways](#vii-conclusion)
* [VIII. References & Further Reading](#viii-references)

---

## I. Reality of building scalable clouds

Cloud computing is everywhere these days. The majority prefer hyperscalers, while others go for local or custom solutions. Building a cloud from scratch is a long, demanding project, so many opt for ready-to-go platforms where the groundwork is done, like OpenStack. While OpenStack is a monumental open-source achievement, it is fundamentally better suited for private clouds. In a public cloud at hyperscale, you hit hard limits.

To understand why, we have to look at SDN (Software-Defined Networking). SDN is not just a tool for configuring many devices at once, like Ansible. It is a complete paradigm shift where the control plane is separated from the networking hardware into a single, centralized entity. SDN is the backbone of the cloud. It virtualizes hardware, isolates tenants, and provides the rapid elasticity (or autoscaling) and resource pooling required by NIST cloud standards. Without a working SDN, you have no cloud. But developing an SDN is incredibly hard, and fundamental flaws in its architecture can force you to rip it out and start over or do a very costly migration (like we did).

## II. The Anatomy of a Cloud SDN {#ii-anatomy}
Before we dive into how OpenStack Neutron fails, we need to define what a modern Software-Defined Network actually looks like.

<figure class="center-caption">
  <img src="/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_taxonomy.png" alt="SDN Taxonomy Diagram" />
  <figcaption>
    <strong>Fig. 1.</strong> Software-Defined Networks in (a) planes, (b) legacy layers, and (c) modern layers.
  </figcaption>
</figure>

At its core, SDN breaks vertical integration. Traditionally, networking vendors built routers and switches that contained both the forwarding hardware and the complex routing logic inside the same physical box. SDN rips these apart, separating the network's "brain" from its "muscle" (as seen in Figure 1, Column A).
* **The Data Plane (The Muscle)**: The underlying hardware or virtual switches become simple, fast forwarding devices. They don't think; they just move packets based on strict rules pushed down to them.
* **The Control Plane (The Brain)**: The routing logic, security groups, and intelligence are moved to a logically centralized Network Operating System (NOS).<label for="sn-6" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-6" class="margin-toggle"/><span class="sidenote">Early SDN relied heavily on the OpenFlow protocol to dictate every single traffic flow to the data plane. We quickly learned this is incredibly wasteful and unscalable in cloud environments. Modern SDNs rely more on hardware offloading and broader configuration states.</span>
* **The Management Plane (The Business Logic)**: This is where human operators and automated network applications define high-level business requirements and pass them to the Control Plane.

Because the network is now controlled by software sitting on a central server, you can program your infrastructure the same way you write an app for your phone. This makes the network flexible, adaptable, and incredibly easy to evolve.

However, there is a massive tradeoff: the system becomes vastly more complicated. By extracting the brain, we create a centralized single point of failure.<label for="sn-7" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-7" class="margin-toggle"/><span class="sidenote">In distributed systems engineering, the Control Plane is bound by the CAP theorem. To survive hardware failures, the controller must be distributed across multiple servers, introducing complex state-synchronization problems-which is exactly what kills Neutron.</span> The controller architecture must be designed flawlessly to survive at scale.

### The Two Jobs of a Hyperscaler's SDN
It is important to note that we are talking specifically about cloud SDN. A massive, distributed SDN like Neutron or Sprut is completely over-engineered for a standard enterprise datacenter with 20 hypervisors. It is also often detrimental in High-Performance Computing (HPC) clusters.<label for="sn-8" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-8" class="margin-toggle"/><span class="sidenote">In an HPC cluster, raw performance is everything. The CPU overhead and latency introduced by virtualizing the network with VXLAN encapsulation (overlays and underlays) is usually an unacceptable tradeoff. HPC relies on low-latency bare-metal technologies like InfiniBand or RoCE instead.</span>

But in a public hyperscaler, SDN is mandatory. It is the final piece of the puzzle-alongside software-defined compute and storage-that makes a cloud actually function according to NIST standards. In a cloud environment, the SDN has two massive responsibilities:

**1. Network Virtualization (Resource Pooling & Isolation)**
Before SDN, achieving multi-tenancy required a network engineer to manually configure VLANs and firewall rules. SDN dynamically virtualizes the physical network resources. This allows thousands of isolated users to share the exact same physical cables and switches safely, ensuring that one user's traffic spike doesn't cause a bottleneck for another (adhering to the NIST requirement for Measured Service).

**2. API-Driven Automation (Elasticity & Self-Service)**
Without SDN, virtual machines spin up in seconds, but the network takes hours or days to configure. By exposing a central Northbound API, the SDN allows for On-Demand Self-Service. Load balancers, firewalls, and routers can be dynamically provisioned by automated scripts or users clicking a button in a UI, providing the Rapid Elasticity that defines modern cloud computing.

## III. Neutron architecture: Design and Scaling Considerations {#iii-neutron}

The architecture of OpenStack Neutron has distinct advantages, as it was originally designed to provide network-as-a-service for private clouds and enterprise environments. In deployments with a moderate number of hypervisors, its design works brilliantly to abstract complex networking.

Fundamentally, Neutron is not a custom packet-forwarding engine. It is essentially a distributed set of Python daemons that act as a "glue" layer. It ties together and orchestrates existing, battle-tested Linux networking tools-like Open vSwitch (OVS), `iptables`, network namespaces (netns), and dnsmasq. Because it is open-source and modular, it is highly extensible and relatively easy to understand. However, to tie all these disparate tools together, Neutron relies on centralized coordination.

At hyperscale, this architecture faces severe challenges. Its core structural vulnerability is a heavy reliance on a single message queue system (RabbitMQ) to synchronize imperative state commands across thousands of distributed agents.

### The Component Layers of Neutron

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_arch.png)

The legacy Neutron architecture can be separated into four distinct layers:
1. **The Controller Layer (The API and Plugins)**
User requests hit the Neutron API. This API is bundled with core plugins-most notably the **ML2 (Modular Layer 2)** plugin-into a single Python web application known as the neutron-server. It is called a "server" because it serves the REST endpoints to the rest of the cloud, while writing persistent logical state to a central database (MySQL/MariaDB).

    ML2 was introduced to solve a massive industry problem: cloud providers wanted to use different switch vendors (Cisco, Arista, generic Linux servers) simultaneously without rewriting the entire Neutron codebase. It achieves this by splitting configuration into two parts:

    * **Type Drivers**: Define the network topology (e.g., VLAN, VXLAN, GRE).

    * **Mechanism Drivers**: Translate that topology into the specific language of the underlying hardware or software.

2. **The Transport Layer**: A message broker, typically RabbitMQ, sits between the controller and the compute nodes. It handles the heavy lifting of routing asynchronous RPC (Remote Procedure Call) messages from the ML2 plugin down to the distributed agents.

3. **Agents**: These are the Python daemons running on compute or network nodes. They listen to the message queue and translate logical RPC commands into actual dataplane rules.

    3.1. **OVS Agent (Layer 2)**: Configures switching rules on the hypervisor's Open vSwitch. It is also historically responsible for implementing Security Groups (firewall rules) locally on the compute node via `iptables`.
    
    3.2. **L3 Agent**: Handles Layer 3 protocols, routing, and Floating IPs (NAT).

    3.3. **DHCP Agent**: Manages IP address assignment and static routes for virtual machines.

    3.4. **Metadata Agent**: Proxies requests from instances to the Nova metadata service.

4. **Dataplane**: This is the underlying Linux/This is the underlying Linux network infrastructure. It is important to note that netns (Network Namespaces) and daemons like dnsmasq are not Neutron agents themselves. They are standard Linux kernel features managed by the agents. For example, the DHCP agent spawns a unique dnsmasq process inside an isolated netns to serve IP addresses to a specific tenant network without overlapping with others.<label for="sn-9" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-9" class="margin-toggle"/><span class="sidenote">This modularity is why OpenStack created OVN (Open Virtual Network) as a modern replacement. OVN offloads almost all dataplane configuration (L2, L3, DHCP, Security Groups) into highly optimized OpenFlow rules within OVS, allowing Neutron to step back and act purely as a high-level API manager.</span>

### Typical Neutron Hyperscale Deployment

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_deployment.png)

To understand why legacy OpenStack breaks at hyperscale, we must look at how it is physically deployed. The diagram above represents a standard, highly available deployment model.

Imagine this architecture stretched to its absolute limits. In a hyperscale scenario like ours, we are talking about roughly 3,000 bare-metal hypervisors hosting 160,000 VMs and 200,000 virtual ports. Here is a breakdown of how the components are physically distributed to achieve High Availability (HA).

#### 1. The Physical Underlay: Spine-Leaf Architecture
At the top of the diagram are the physical network switches arranged in a Spine-Leaf topology. Every server plugs into a Top-of-Rack (Leaf) switch, and every Leaf switch connects to every Core (Spine) switch.<label for="sn-10" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-10" class="margin-toggle"/><span class="sidenote">Traditional IT networks were built vertically (North-South) for traffic leaving the datacenter. In a cloud, the vast majority of traffic is "East-West" (VMs talking to other VMs, or compute talking to storage). Spine-leaf guarantees that any server is the exact same number of "hops" away from any other server, ensuring predictable, ultra-low latency.</span>

#### 2. The Controller Cluster (The Control Plane)
The right side of the diagram shows the "brain." In a deployment of 3,000 hypervisors, this cannot run on virtual machines. It requires a cluster of dedicated, massive bare-metal servers. To eliminate single points of failure, the control plane is stacked:
    * **HAProxy**: A load balancer that sits at the edge, distributing incoming API requests across active neutron-server workers.
    * **MySQL NeutronDB**: Deployed as a synchronous Galera cluster. Every database write is strictly replicated across the controller nodes to prevent split-brain scenarios and data loss.
    * **RabbitMQ Cluster**: The transport layer is also clustered to ensure message queues survive a hardware failure.

#### 3. Compute Nodes (The Dataplane workers)
These are the servers where the actual tenant VMs live. Every single one of these 3,000 compute nodes runs an ovs-agent daemon. This agent maintains a persistent, open connection back to the central RabbitMQ cluster, waiting for instructions to configure its local virtual switch.

#### 4. Network Nodes
Unlike compute nodes, Network Nodes do not run tenant VMs. They are dedicated, high-throughput bare-metal servers that run the L3 and DHCP agents. Routing thousands of gigabits of public internet traffic requires immense CPU overhead. If you placed this load on a standard compute node, it would steal CPU cycles from the paying tenants' VMs. High availability is achieved here using software redundancy, such as VRRP (Virtual Router Redundancy Protocol), which creates active/standby router pairs across multiple nodes.

While this hardware design is robust and effectively eliminates physical single points of failure, hardware reliability does not equal software scalability. Stretching a central RabbitMQ cluster across 3,000 hypervisors creates a ticking time bomb in the transport layer.

### Workflow Example: Port Creation (The Happy Path)

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/port_creation_workflow.png)

Now let’s look at how the software overlay manipulates the hardware. We will trace the most common operation in OpenStack: creating a virtual network port. In a healthy cloud, this relies on asynchronous message passing. Follow the numbered circles on the diagram.

**Step 1: The API Request.** When a tenant requests a new VM, the Compute service (Nova) sends a REST request to Neutron via the load balancer. The request is assigned to an ML2 plugin worker on the Controller Cluster.

**Step 2: Committing to the Database.** ML2 validates the request, generates a UUID, assigns MAC/IP addresses, and compiles security group rules. It writes this "Target State" into the central database. At this moment, the port logically exists in the control plane, but the physical dataplane knows nothing about it.

**Step 3: The Asynchronous Handoff.** ML2 packages the port configuration into an RPC message and drops it into the RabbitMQ cluster. The API worker’s job is done; it is free to handle the next user request.<label for="sn-11" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-11" class="margin-toggle"/><span class="sidenote">This asynchronous design is why OpenStack feels fast to the user. If the ML2 plugin had to establish a direct SSH connection to configure the compute node manually, API response times would skyrocket, and the central controller would quickly run out of worker threads.</span>

**Step 4: Physical Delivery and Execution.** The message travels through the physical Spine-Leaf switches until it reaches the specific hypervisor. The local ovs-agent receives the message, unpacks the JSON payload, and executes the local Linux commands to wire up the virtual interface. The VM boots, and traffic flows.

### Where the Cracks Begin to Form

This decoupled architecture is elegant on paper, but it introduces a fatal flaw at scale: it assumes the transport layer is perfectly reliable.

What happens if the RabbitMQ cluster is momentarily overwhelmed by 3,000 agents reporting their status simultaneously, and it drops the message? The local ovs-agent is blind. It has no idea Steps 1, 2, and 3 ever occurred. The central database says the port is ACTIVE, but on the hypervisor, the VM has no network connection.

Because OpenStack relies on decentralized agents, it requires a fail-safe to correct these desyncs. When an agent loses its connection to the message queue-even for a few seconds-it cannot trust its local state. To fix this, the agent is forced to trigger a blunt recovery mechanism known as a "Full Sync." In a hyperscale environment, this is often the exact mechanism that brings the entire cloud crashing down.

### Workflow Example: Network Node Full Sync

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_fullsync.png)

At its core, a cloud environment is a massive state machine. The fundamental promise of OpenStack is that the physical reality of the datacenter (Actual State) must strictly mirror the central database (Target State).

When an agent daemon is restarted-due to a crash, an upgrade, or a network blip-its local memory cache is invalidated. The agent wakes up blind. It cannot assume its local configuration is correct, because an API user might have deleted a router or added a hundred ports while the agent was offline. To reconcile this, the agent triggers a Full Sync. Let’s walk through the diagram above, using a Network Node as our example.

**Step 1**: The Call for Help. The moment the agent reconnects to RabbitMQ, it realizes it has lost synchronization. It fires off a direct RPC call to the controller: "Give me the complete configuration state for every single resource assigned to my hostname."

**Step 2**: API Reception. The request routes through HAProxy and is picked up by the ML2 Plugin.

**Step 3**: The Heavy Database Query. Unlike simple port creation, this is computationally expensive. ML2 must execute massive, complex JOIN queries against the database to gather the complete state of every logical router, floating IP, namespace, and DHCP pool scheduled to that node.

**Step 4**: Packaging the Payload. The API worker compiles all this data into a massive, monolithic JSON payload and drops it into a dedicated RabbitMQ reply queue.

**Step 5**: The Massive Delivery. The large data payload traverses the physical network. The agent downloads this state file, wipes its local caching discrepancies, and methodically configures the Linux dataplane to perfectly match the central database.

### The Advantage (And the Trap)
This mechanism exists because it makes operational recovery incredibly simple. If an SRE suspects a node has corrupted rules, they just restart the agent. The agent downloads the absolute truth from the database and flawlessly heals itself. In a 50-node private cloud, this works perfectly.

But look closely at Step 5. What happens if that massive payload takes too long to generate, or gets dropped by RabbitMQ because the queue is congested? The agent waits for a timeout period. When the message doesn't arrive, the agent assumes the request was lost, and it fiercely fires off Step 1 all over again.

Imagine a scenario where a rack switch reboots, causing 40 compute nodes to drop offline and request a full sync at the exact same second. As we will explore in the next section, this simple, self-healing loop transforms into a catastrophic, self-inflicted Denial of Service attack against the cloud's own control plane.


## IV. The "Full Sync" Disaster {#iv-disaster}

While the Full Sync mechanism is a reliable safety net for minor, everyday operational hiccups, at hyperscale, it can horribly backfire. A single incident can cost a company millions of dollars.

Let's look at a catastrophic scenario: an entire datacenter loses power. Suppose we resolve the energy grid failure quickly, but our backup power fails to engage (which actually happened to me), causing the servers to shut down hard.

Getting the compute capacity back online-even for 160,000+ VMs, including massive 128-core, 1TB RAM instances-is surprisingly fast. The compute service can boot them in about 20 minutes.<label for="sn-12" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-12" class="margin-toggle"/><span class="sidenote">OpenStack's compute service (Nova) is highly distributed. Hypervisors can independently boot local VMs from local disk caches without waiting for a massive payload from a central bottleneck.</span> But without the network, compute is useless. The users' infrastructure cannot serve external requests, computational clusters cannot communicate during MapReduce operations, and the cloud remains effectively dead.

When those 3,000+ hypervisors power back on, their local network agents wake up with wiped memories. They are blind. For the network nodes, our 200,000+ virtual Neutron ports are basically gone. To fix this, every single agent simultaneously triggers a Full Sync.

### The Thundering Herd and the Timeout Loop

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_disaster.png)

This is where the architecture crumbles. All 3,000 hypervisors simultaneously fire RPC requests into RabbitMQ, demanding the state of their respective dataplane entities.

The messaging queue is not the only problem-you cannot just throw more RAM at RabbitMQ to fix this.<label for="sn-13" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-13" class="margin-toggle"/><span class="sidenote">RabbitMQ is optimized for high-throughput, small-payload message routing. Pumping multi-megabyte JSON Full Sync state files through it instantly causes severe memory pressure and queue drops.</span> The true bottleneck is the central control plane. The ML2 Plugin has to process these thousands of concurrent requests by executing massive, complex SQL JOIN queries against the central MySQL database.

The database CPU instantly spikes to 100%. Database locks occur. The queue overflows. Because the database is locked up, the ML2 Plugin takes an excruciatingly long time to generate the payload. If it takes longer than the agent's built-in timeout setting, the agent assumes its original request was lost in the network.

What does the agent do? It aggressively fires another Full Sync request into the queue. The system effectively performs a catastrophic Denial of Service (DDoS) attack on itself. Under this self-inflicted, crushing load, the system throughput drops to a crawl-processing only about 9,000 ports per hour. Because of this architectural flaw, a 30-minute power outage instantly turns into an agonizing 20+ hour recovery nightmare.

### The VIP Client Dilemma

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_port_order_problem.png)

During a 20-hour outage, business priorities become critical. Almost all large cloud providers have VIP clients that bring in 80%+ of the revenue. They are the driving force for any provider's growth. They also have tighter SLAs. Naturally, you want to restore their networks first.

With legacy Neutron, you can't.

As shown in the diagram above, the Full Sync restoration order is completely uncontrollable. The ovs-agent configures the dataplane at Layer 2. It only sees UUIDs, MAC addresses, and Linux namespaces. It has absolutely no concept of OpenStack "Tenants," "Quotas," or "VIP Billing Status"-that metadata only exists in the locked-up MySQL database.<label for="sn-14" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-14" class="margin-toggle"/><span class="sidenote">OpenStack Keystone handles identity and multitenancy. By the time a configuration reaches the physical hypervisor agent, all tenant context is stripped away for security and simplicity, leaving only raw network primitives.</span>

The agent just blindly asks for UUIDs. Additionaly, a VIP client's VMs are scattered across hundreds of different hypervisors for fault tolerance. Because you cannot tell the decentralized agents to prioritize specific tenant workloads, the VIP client's recovery is completely held hostage by the congested global queue.

Attempting manual intervention by SREs to speed this up is incredibly risky. Manually restarting services or tweaking queues can easily break the fragile Full Sync progress, forcing the timeout loop to start all over again (which was exactly what exacerbated the outage in our case and we had to start over).

### OpenStack Cells Won't Solve Neutron Scalability
When discussing OpenStack at hyperscale, architects often point to OpenStack Cells as the ultimate scaling silver bullet. And for the compute side, they are right.

OpenStack Cells allow you to partition a massive cloud deployment into smaller, isolated "shards." Instead of putting 3,000 hypervisors on one message queue, you break them into manageable chunks-say, 10 cells of 300 hypervisors each. Each cell gets its own local database and its own local message queue for compute operations, while the end-user still interacts with a single, unified global API. Without Cells, running 160,000 VMs would be impossible; Nova would collapse under its own weight just like Neutron.

However, there is a common misconception that deploying OpenStack Cells shards your network, too. It does not. OpenStack Cells only shard Nova (Compute). They do not cover Neutron (Networking), Cinder (Storage), or Glance (Images).<label for="sn-15" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-15" class="margin-toggle"/><span class="sidenote">While there have been community discussions and experimental attempts to shard Neutron using routed provider networks or multi-region setups, there is no native, push-button "Neutron Cells" equivalent to Nova Cells v2.</span>

This means that even if you perfectly partition your 3,000 hypervisors into beautifully isolated compute cells, the ovs-agent on every single one of those 3,000 machines is still reporting back to the exact same, monolithic, global Neutron RabbitMQ cluster and MySQL database. The compute plane is distributed, but the network control plane remains a massive single point of failure.

Because Cells cannot save the network, scaling an OpenStack cloud to hyperscale sizes inevitably forces a harsh realization: you cannot fix Neutron by partitioning the infrastructure around it. You have to rip out and replace the underlying SDN entirely.

### Looking Forward
It is worth noting that these cascading failures exist only in large-scale public installations of OpenStack. Originally, OpenStack was designed as a private cloud solution where you simply don't have this massive concentration of hypervisors and ports competing for the same message queue.

But VK Cloud is a hyperscaler and must protect its reputation and its clients. That is why a separate, proprietary SDN solution called SPRUT was created to combat the limitations of Neutron. While the open-source community eventually developed OVN to solve these same issues, OVN was not mature enough when SPRUT development began. Furthermore, we needed a system that maintained the exact same API to ensure a smooth, zero-downtime migration for our clients.

## V. The Architectural Shift: Moving to the Modern Models {#v-shift}

To understand how modern solutions like OVN and Sprut solve Neutron's fatal flaws, we first need to look at how the fundamental philosophy of Software-Defined Networking has evolved over the last decade.

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_taxonomy.png)

Looking at the architectural evolution in the diagram above, legacy Neutron was built on the concepts of early network virtualization (Column B). It relied heavily on a top-down, imperative model.<label for="sn-16" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-16" class="margin-toggle"/><span class="sidenote">The classical taxonomy in columns A and B is based on Diego Kreutz's foundational 2015 paper, <em>"Software-Defined Networking: A Comprehensive Survey."</em> While the concepts of plane separation remain true, the strict layer definitions have evolved significantly in modern cloud implementations.</span> The central controller didn't just define what the network should look like; it sent explicit, transient RPC messages dictating exactly *how* to configure it, relying on agents to blindly execute heavy Linux shell commands. If a message was dropped, the state broke.

Modern SDNs have shifted to the model shown in Column C. This is the era of Intent-Based APIs, Distributed Control Planes, and Programmable Data Planes.

Instead of shouting transient commands into a fragile message queue, the modern control plane stores a declarative "Target State" (the intent) in a high-speed database or REST API. Lightweight agents on the programmable data plane independently pull this state, compare it to their actual physical reality, and natively compile the differences directly into the virtual switches.

This architectural shift-from imperative RPC messaging to declarative state reconciliation-is the absolute key to surviving hyperscale. Let's look at how the OpenStack community and VK Cloud independently implemented this shift to save their control planes.

### The New OpenStack Standard: OVN

Before we dive into the proprietary solution (Sprut) that VK Cloud ultimately built, it is crucial to examine the open-source community's answer to Neutron's scalability crisis: OVN (Open Virtual Network).<label for="sn-17" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-17" class="margin-toggle"/><span class="sidenote">For a deeper dive into the open-source implementation, refer to the <a href="https://docs.openstack.org/ovn/latest/">official OpenStack OVN documentation</a> or the foundational architectural presentations from the Open vSwitch community.</span>

A common misconception is that OVN replaces Neutron entirely. It does not. Because OpenStack clients, automation scripts (like Terraform), and other core services (like Nova) rely strictly on the standard Neutron REST API, replacing the frontend would break the entire cloud ecosystem.

Instead, OVN takes advantage of Neutron's Modular Layer 2 (ML2) design. It leaves the frontend API completely intact, but radically guts and replaces the backend. It strips out RabbitMQ and the scattered Python agents in favor of a unified, database-driven control plane.

**Historical Context:** The OVN project was announced around 2015 by the Open vSwitch community. However, achieving true feature parity with the legacy Neutron backend (supporting Distributed Virtual Routing, complex Security Groups, and hardware offloading) at a cloud scale took roughly five years. OVN finally became the default backend in the OpenStack Ussuri release in 2020. This timeline is a critical piece of the puzzle regarding why VK Cloud began developing Sprut-we simply could not wait years for OVN to mature while our datacenter was growing at a hyperscale pace.

#### The Component Layers of OVN

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/ovn_arch.png)

Let’s break down how OVN fundamentally reorganizes the control plane layers.

**1. The Controller Layer**
The top layer still sits on centralized control nodes, but it separates the "Cloud Management" state from the "Network Logic" state:

  * **Neutron API & MySQL DB:** The exact same frontend. Users notice no difference. The legacy database still stores OpenStack-specific metadata (Tenant IDs, quotas, billing info) that OVN doesn't care about.
  * **ML2/OVN Plugin:** The universal translator. It catches API requests, strips away the OpenStack metadata, and translates the request into a purely logical network configuration.
  * **OVN Northbound DB (OVSDB):** The store for the *Target State*. The ML2 plugin writes logical entities here (Logical Switches, Logical Routers). It knows what the network should look like, but not where the VMs physically live.
  * **ovn-northd:** The central compiler daemon. It constantly monitors the Northbound DB and translates high-level logical concepts into millions of low-level logical flows (MAC lookups, ACLs, routing tables).
  * **OVN Southbound DB (OVSDB):** The store for the *Actual State* and physical bindings. It contains the compiled logical flows from `ovn-northd` and tracks the physical location of every hypervisor (Chassis) and which VM is plugged into which host.

**2. The Transport Layer (State Sync, not Messaging)**
This is the most critical architectural shift. OVN completely removes the RabbitMQ asynchronous message bus. Instead, it relies on the **OVSDB protocol**, a database synchronization protocol. Rather than pushing transient messages into a queue, the control plane simply updates the Southbound DB. The agents independently connect to this database and synchronize only the rows that pertain to their specific physical hardware.

**3. The Agents Layer (The Unified Controller)**
Legacy Neutron required up to five different Python daemons fighting for resources on every hypervisor. OVN replaces all of them with a single, highly efficient C-daemon: the **`ovn-controller`**.

**4. The Dataplane Layer**
The underlying dataplane remains Open vSwitch. The `ovn-controller` receives state from the Southbound DB and programs it directly into the local OVS instance using OpenFlow rules. Because OVN handles L3 and DHCP natively inside the switch's flow tables, it largely eliminates the need to spin up messy Linux Network Namespaces (`netns`) and `dnsmasq` processes on the compute nodes.

#### The Paradigm Shift: Eliminating the Bottlenecks

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_ovn_comparison.png)

If you look at the side-by-side comparison above, you can see how OVN systematically dismantles the bottlenecks that made legacy Neutron so fragile:

  * **Pre-Calculated State vs. On-the-Fly SQL:** In Neutron, every agent request forces the ML2 plugin to hit MySQL with complex `JOIN` queries, risking a total database lock. In OVN, `ovn-northd` acts as a continuous background compiler. When an agent needs its state, it simply performs a lightweight "Read" operation of pre-calculated data from the Southbound DB.
  * **State Synchronization vs. RPC Messaging:** RPC messages are inherently two-way, locking transactions that do not scale horizontally. OVN replaces this with OVSDB, which can be deployed as a highly available Raft cluster (one Leader for writes, multiple Followers for read replicas), instantly distributing the load across the control plane.
  * **Agent Footprint & Payload Efficiency:** OVN consolidates 5 Python daemons into a single C-daemon. Furthermore, because OVN handles routing natively inside the virtual switch, the payload sent to the agent consists of tiny, highly optimized binary OpenFlow rules rather than heavy Linux shell commands.

#### The Customization Trade-off: C vs. Python

While OVN's architecture is significantly more robust, it comes with a major disadvantage for agile cloud operators: Extensibility.

Legacy Neutron was written entirely in Python. If SREs needed a custom traffic shaping rule or a hotfix for a VIP client, they could simply write a Python extension for the agent and deploy it. OVN is entirely different. The core engines that do the actual work (`ovn-northd` and `ovn-controller`) are written in C.

Adding a fundamentally new dataplane feature requires writing C code, modifying the core OVSDB schema, compiling binaries, and often submitting changes upstream to the Open vSwitch community to ensure compatibility. For a cloud provider looking to rapidly deploy proprietary features, this language barrier represents a massive loss of flexibility.

-----

### The Proprietary Solution: Sprut

*(Note: The technical architecture discussed in this section is based on engineering materials published by VK Cloud.)*

When VK Cloud realized that legacy Neutron could no longer support our hyperscale growth, we established a strict set of requirements for a replacement SDN:

1.  **100% Feature Parity:** Existing customer features could not be disabled.
2.  **L3 ToR Architecture:** Data center networks were built on an "L3 per rack" principle, requiring external traffic to be routed via BGP to the Top-of-Rack switches.<label for="sn-18" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-18" class="margin-toggle"/><span class="sidenote">L3 Top-of-Rack routing inherently breaks legacy Layer 2 broadcast protocols like VRRP, forcing the new SDN to handle High Availability routing natively without relying on broadcast domains.</span>
3.  **Massive Scalability:** The control plane had to shard or scale horizontally to eliminate bottlenecks.
4.  **Seamless Migration:** We needed to migrate 160,000+ VMs with zero downtime.
5.  **Customization:** We required an open, modifiable architecture (unlike OVN) to rapidly build proprietary features.

Open-source alternatives like Tungsten Fabric were too complex and poorly documented for our timeline. Rewriting Neutron was a dead end. And OVN was still immature. Faced with these limitations, VK Cloud built **Sprut**-a custom, highly scalable SDN designed specifically to survive hyperscale workloads, written in approximately 15,000 lines of highly efficient code.

#### The Closed Control Loop

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_sprut_comparsion.png)

Before looking at the layer design, we must look at how Sprut fundamentally redesigned the Transport Layer.

Legacy Neutron relied on an *event-driven* model. When a port changed, ML2 pushed a message into RabbitMQ hoping the agent would receive it. Sprut completely eliminates RabbitMQ, replacing the transport layer with a standard, highly scalable **HTTP REST API**.

By moving to HTTP REST, Sprut shifts from an "Event-Driven" model to a **"Closed-Loop"** control theory model:

1.  Sprut agents no longer wait passively for transient events from a central queue.
2.  Instead, they continuously poll the HTTP REST API to request their "Target State."
3.  The agent independently compares this Target State against the "Actual State" of its local hardware.
4.  If there is a difference, the agent applies *only the difference* to the local dataplane to minimize the error and bring the system into alignment.

#### The Component Layers of Sprut

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sprut_arch_detailed.png)

To simplify the complexities of debugging an Open vSwitch dataplane across thousands of nodes, Sprut adopted a strict separation of concerns.

**1. The APP Layer (The Translator)**
This layer ensures complete API compatibility with OpenStack. The **Sprut Plugin** completely replaces the legacy ML2 plugin. It receives standard Neutron requests and translates them into distinct instructions for the underlying SDN and NFV layers.

**2. The NFV Layer (Network Function Virtualization)**
This layer provides network features (primitives), not routing.

  * **NFV Agents:** These manage specific entities like DHCP servers, Virtual Routers, and Metadata Proxies.
  * **Selective Placement:** Unlike Neutron, which forces heavy agents onto every compute node, Sprut's NFV agents only run on dedicated nodes selected by operations. This conserves critical CPU and RAM on the hypervisors for paying tenants.

**3. The SDN Layer (The Connectivity Engine)**
This layer is responsible purely for moving packets from Point A to Point B. It does not know what a "Router" or a "DHCP Server" is; it only operates using "Links" and "Endpoints."

  * **The Network Graph:** SDN agents on the hypervisors create virtual switches and tunnels. They constantly monitor these tunnels by sending probe packets, reporting latency and status back to the SDN Controller in real-time. The controller uses this to build an active graph where vertices are switches and edges are tunnels.
  * **Dijkstra’s Routing:** When a connection is required, the SDN Controller uses Dijkstra's algorithm on the network graph to find the absolute shortest path. It generates traffic forwarding rules specifying the exact port sequence and pushes them to the agents, functioning similarly to the OSPF routing protocol.

#### The Workflow: Creating a Virtual Network in Sprut

By separating these concerns, provisioning a network becomes a clean, programmatic cascade:

1.  **User Request:** The user asks the standard Neutron API to create a virtual network.
2.  **Translation:** The APP Layer transforms the request into distinct entities.
3.  **Provisioning Primitives:** The APP layer instructs the NFV Layer to spin up the necessary unattached entities (e.g., a DHCP server).
4.  **Logical Wiring:** The APP layer tells the SDN layer to create "Links" and "Endpoints" to connect these entities.
5.  **Compute Attachment:** The user tells Nova to boot a VM. The Nova-compute agent starts the QEMU process and asks the APP layer to connect it to the network.
6.  **Finalizing the Route:** The SDN Controller calculates the shortest path via Dijkstra's algorithm, and the closed-loop agent applies the rules to the local Open vSwitch. The VM is instantly online.

## VI. Solving the Full Sync Disaster {#vi-solving}

In Chapter IV, we explored the nightmare scenario: a datacenter loses power, bringing down 3,000 hypervisors. When the servers reboot, the local network agents wake up with wiped memories and blindly demand their configuration state, creating a catastrophic Denial of Service loop that can take legacy Neutron over 20 hours to resolve.

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_arch_comparison.png)

Looking at the side-by-side architectural comparison above, we must ask a logical question: **If a datacenter loses power, 3,000 hypervisors wake up blind in Neutron, OVN, and Sprut alike. All three systems face the exact same stampede of agents demanding their state. So why does Neutron collapse into a 20-hour DDoS loop, while OVN and Sprut recover in minutes?**

The answer lies in how these new architectures fundamentally process, package, and transport that state. Notably, the OpenStack Foundation (with OVN) and VK Cloud (with Sprut) independently arrived at strikingly similar architectural conclusions. Here is how modern designs solve the Full Sync disaster.

### Pre-Calculated State: Stopping the Database Lock

The single biggest failure point during a Neutron recovery is the central database. When 3,000 Neutron agents ask for a Full Sync, the ML2 plugin has to construct the state on the fly. It queries the MySQL database and executes thousands of heavy, multi-table JOIN queries simultaneously to figure out how security groups, subnets, and ports relate to each specific hypervisor. MySQL immediately locks up under the CPU load.<label for="sn-19" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-19" class="margin-toggle"/><span class="sidenote">Relational databases like MySQL use row-level or table-level locking during complex read/write transactions to maintain ACID compliance. When thousands of concurrent <code>JOIN</code> queries hit the database, these locks queue up, stalling the entire application layer.</span>

Both OVN and Sprut solve this by cleanly separating "Cloud Management" (Neutron DB) from "Network Logic" (OVSDB / Sprut DB).

  * **In OVN:** The `ovn-northd` daemon acts as a continuous background compiler. Long before an outage occurs, it translates high-level API concepts into low-level OpenFlow rules and stores them statically in the Southbound DB.
  * **In Sprut:** A similar compilation process happens before the agent ever requests it.

When 3,000 agents wake up in these modern systems, there is no compiling or SQL `JOIN`ing left to do. The Southbound DB (or Sprut's HTTP REST backend) simply hands over the pre-calculated rows that belong to that specific chassis. It is a simple "Read" operation, which modern databases can perform millions of times a second without locking.

### The Right Transport: State Sync vs. RPC Queues

When OpenStack was designed in 2010, architectural standards dictated that all services must communicate via an AMQP message broker (RabbitMQ). This is excellent for asynchronous, one-off tasks-like instructing Nova to start a VM. However, configuring a dataplane requires continuous *state synchronization*. Using a task queue for massive state replication is like trying to stream a 4K movie through the postal service. Furthermore, an RPC request is inherently a two-way, locking transaction that scales poorly.

Modern SDNs abandoned the queue in favor of protocols actually designed for state replication:

  * **OVSDB Read Replicas:** OVN uses the OVSDB protocol, which operates on a Raft cluster.<label for="sn-20" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-20" class="margin-toggle"/><span class="sidenote">Raft is a consensus algorithm designed to manage a replicated log. It allows a cluster of servers to maintain a highly available state machine, guaranteeing that all nodes agree on the data even if some servers fail.</span> You have one Leader for writes and multiple Followers for reads. When 3,000 hypervisors wake up, they connect to the read-only Followers, instantly distributing the stampede load across the cluster.
  * **HTTP REST Caching:** Sprut utilizes a standard HTTP REST API. If 3,000 nodes execute standard GET requests simultaneously, the autoscaling API fleet-backed by standard load balancers and caching layers-easily serves the pre-calculated Target State.<label for="sn-21" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-21" class="margin-toggle"/><span class="sidenote">Standard REST HTTP allows for the use of ETags. If an agent requests state and the ETag hasn't changed, the server just returns a <code>304 Not Modified</code> header without sending a payload, saving massive amounts of bandwidth.</span> There are no fragile message queues to overflow and no timeout-retry loops.

### Payload Efficiency: Micro-Diffs vs. Megabytes

To restore a legacy Neutron compute node, the Full Sync payload must contain massive JSON files detailing every single `iptables` rule, DHCP lease, and routing table entry. It is megabytes of data per node, processed by Python wrappers executing heavy Linux shell commands.

OVN and Sprut program the Open vSwitch directly in C or via native OpenFlow rules. The payload difference is staggering:

  * **OVN** downloads highly optimized, binary OpenFlow tables.
  * **Sprut** downloads a lightweight topology of "Links" and "Endpoints," compares its current state to the Target State, and applies *only the differences*. A smaller payload means the physical network doesn't congest, the agent processes the diff in milliseconds, and the node is online instantly.

### Execution Efficiency and the Dataplane Abstraction

Finally, we must look at what the agents are actually forced to *do* when they receive their state.

In a legacy Neutron deployment, the 3,000 compute nodes primarily run the `ovs-agent`. While routing and DHCP are kept on separate Network Nodes, the `ovs-agent` is still responsible for port bindings and Security Groups. To restore a node, the bloated Python agent must execute hundreds of heavy Linux shell commands (like `iptables` or `ovs-ofctl`) to wire up the host.<label for="sn-22" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-22" class="margin-toggle"/><span class="sidenote">Historically, Neutron used <code>iptables</code> and Linux bridges alongside OVS to handle Security Groups because OVS lacked native stateful firewall capabilities. Modern OVS supports native connection tracking (conntrack), eliminating the heavy Linux bridge overhead.</span> This grinds the hypervisor's CPU to a halt and drastically slows down the central queue processing.

Modern SDNs bypass this messy execution model entirely:

  * **OVN’s Native OpenFlow:** OVN consolidates the agent footprint into a single, highly efficient C-daemon (`ovn-controller`). More importantly, it eliminates the need for messy Linux network namespaces and `dnsmasq` processes entirely. It implements DHCP and Routing *natively* inside the virtual switch's OpenFlow tables. The payload is small, and applying binary OpenFlow rules takes milliseconds.
  * **Sprut’s Strict Separation of Concerns:** Sprut took this optimization a step further through its strict **SDN / NFV split**. Like Neutron, Sprut isolates heavy primitives (DHCP, Routers, Metadata) onto dedicated network nodes. However, Sprut completely decouples their data models. When the 3,000 compute nodes wake up blind, their lightweight SDN agents only ask the HTTP REST API for their simple L2 switching topology graph to run Dijkstra’s algorithm. Because the compute agents are completely decoupled from the heavy NFV logic, the data they must pull is incredibly small. They download their local graph, apply the rules, and bring the nodes online instantly without dragging down the control plane.

## VII. Conclusion and key takeaways {#vii-conclusion}

A 20-hour datacenter outage is a brutal teacher. The fundamental lesson we learned at VK Cloud is that you cannot simply scale a private cloud architecture into a public hyperscaler by throwing more hardware at it. OpenStack Neutron is a phenomenal piece of software for its intended use case, but a design that relies on centralized, imperative message queues is a ticking time bomb when stretched across 3,000 hypervisors.

For infrastructure engineers, architects, and SREs building or maintaining large-scale distributed systems, the evolution from Neutron to modern SDNs like OVN and Sprut offers several critical architectural takeaways:

1. State Synchronization Beats Message Queues: Task queues (like RabbitMQ) are for asynchronous events ("Start this VM"). They are the wrong tool for continuous dataplane replication. For massive state synchronization, you must use purpose-built database replication protocols (like Raft-backed OVSDB) or heavily cached HTTP REST APIs.

2. Declarative (Closed-Loop) Beats Imperative: Systems fail when agents wait passively for transient instructions. Modern agents must be autonomous-continuously polling a declarative "Target State" and natively reconciling the differences against their local hardware.

3. Pre-Calculation Prevents Database Locks: Never design a system that generates complex state on the fly during a recovery storm. The control plane must continuously pre-compile high-level API intent into low-level dataplane rules in the background, so recovery is just a fast, lock-free database "Read."

4. Strict Separation of Concerns (SDN vs. NFV): Do not force compute hypervisors to process heavy network services (DHCP, NAT, Metadata). Keep compute agents lightweight-focused purely on simple L2 switching and Dijkstra routing-and offload the heavy stateful services to dedicated Network Nodes. This keeps the recovery payload tiny and saves CPU cycles for the paying tenants.

#### The Next Challenge: Swapping the Engine in Flight
Designing a modern SDN that solves these mathematical bottlenecks on paper is a massive engineering achievement. But architecting a new system is only half the battle. The true nightmare is operational.

How do you take a live hyperscaler with 160,000 running VMs, rip out its nervous system (Neutron), and replace it with a new one (Sprut)?<label for="sn-23" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-23" class="margin-toggle"/><span class="sidenote">In cloud infrastructure, this is the equivalent of swapping out the engine of a commercial airliner while it is flying at 30,000 feet. The clients do not care about your technical debt; they only care about their uptime.</span>

That requires meticulous live-migration strategies, custom translation tooling, and a lot of nerve. In the next article, we will dive into the exact caveats, pitfalls, and strategies we used to successfully migrate a 200,000-port network from legacy Neutron to Sprut with zero downtime.

## VIII. References & Further Reading {#viii-references}

The architectural models, technical limitations, and historical context discussed in this article are synthesized from a combination of hands-on hyperscale engineering experience and the following foundational resources.

**SDN Fundamentals & Academic Theory**
* [Software-Defined Networking: A Comprehensive Survey](https://ieeexplore.ieee.org/document/6994333) *(IEEE Xplore, 2015)* – The foundational paper by Diego Kreutz et al. that established the classic Data, Control, and Management plane taxonomy used in early SDN models.

**Legacy OpenStack & Neutron Architecture**
* [The Theory of Everything in Neutron](https://aptira.com/theory-of-everything-in-neutron/) *(Aptira)* – An excellent deep-dive into the raw mechanics and daemons that make up legacy Neutron.
* [OpenStack Neutron Networking in the Cloud Demystified](https://superuser.openinfra.org/articles/openstack-neutron-networking-in-cloud-demystified/) *(OpenInfra Superuser)* – A breakdown of the original architectural goals of OpenStack networking.
* [Understanding OpenStack Nova Cells](https://cloudification.io/cloud-blog/understanding-openstack-nova-cells-scaling-compute-across-data-centers/) *(Cloudification)* – Context on how OpenStack shards the compute plane (and why this doesn't save the network control plane).
* [OpenStack Neutron Official Wiki](https://wiki.openstack.org/wiki/Neutron) – The historical and technical documentation for the legacy networking project.

**The OVN (Open Virtual Network) Evolution**
* [OVN Architecture and Overview](https://www.openvswitch.org/support/slides/OVN_Austin.pdf) *(Open vSwitch)* – Presentation slides from the architects of OVN detailing the shift to OVSDB and logical flows.
* [Networking with Open Virtual Network](https://docs.redhat.com/de/documentation/red_hat_openstack_platform/14/html/networking_with_open_virtual_network/open_virtual_network_ovn) *(Red Hat OpenStack Platform Docs)* – Enterprise documentation on how OVN integrates with ML2 and the Southbound/Northbound databases.
* [OpenStack OVN Charm Administration Guide](https://docs.openstack.org/charm-guide/latest/admin/networking/ovn/index.html) *(OpenStack Docs)* – Practical deployment architecture for modern OVN controllers.

**Hyperscale Implementation (VK Cloud & Sprut)**
* [Architecture and Evolution of SDN at VK Cloud](https://habr.com/ru/companies/vk/articles/763760/) *(Habr)* – The primary engineering deep-dive published by the VK Cloud team detailing the proprietary Sprut architecture, the shift to HTTP REST, and the SDN/NFV split. *(Available in Russian, but diagrams are in English)*