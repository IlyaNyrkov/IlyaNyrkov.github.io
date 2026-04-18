---
layout: post
title: "Surviving 200,000 Ports: Why OpenStack Neutron Breaks at Hyperscale"
---
In this post, we are going to explore Software-Defined Networking (SDN) architectures at hyperscale - specifically, what happens when you push a cloud environment past 160,000 VMs, 200,000 virtual ports, and roughly 3,000 bare-metal hypervisors. To put that scale into perspective, OpenStack Neutron was originally designed for private enterprise environments and historically recommended a maximum of around 500 hypervisors.<label for="sn-1" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-1" class="margin-toggle"/><span class="sidenote">While aggressive tuning of RPC workers and database pools can push this limit higher, 500 nodes is widely considered the historical threshold where the default RabbitMQ message bus begins to severely bottleneck under Neutron's architecture.</span>

But as VK Cloud<label for="sn-2" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-2" class="margin-toggle"/><span class="sidenote"><a href="https://cloud.vk.com/en/main/">VK Cloud</a> is the enterprise B2B cloud division of VK (formerly Mail.ru Group), offering infrastructure, platform services, and machine learning tools. Has at least 4 Tier 3 datacenters. </span> rapidly grew into one of the largest public cloud providers in the CIS region - establishing itself as the primary alternative to Yandex<label for="sn-3" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-3" class="margin-toggle"/><span class="sidenote"><a href="https://yandex.cloud/en">Yandex Cloud</a> is the largest public cloud platform in Russia and the CIS region, often compared to an AWS or GCP equivalent for that market.</span> - we blew past those private-cloud limits. We pushed the architecture as far as it could go, until it simply couldn't go any further.

I did not design that original OpenStack architecture, nor did I write the code for our new proprietary SDN, Sprut. I joined as a Lead Solution Architect and inherited a massive system that had already suffered a catastrophic "full sync"<label for="sn-4" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-4" class="margin-toggle"/><span class="sidenote">A "full sync" occurs when a local Neutron agent loses its connection state and requests the entire network topology from the central server. At hyperscale, hundreds of agents doing this simultaneously creates a "thundering herd" that crushes the control plane.</span> outage a few years prior. When the same disaster happened a second time (now on my watch), I spearheaded the initiative to migrate our highest-paying clients away from Neutron to our new SDN (I also consulted development team on automated migration solution for small clients).

To prove to our clients that the new system would keep them safe, I spent months studying documented and extracting undocumented wisdom from our SRE and maintenance teams. I dug deep into the theoretical and practical limits of our SDN to understand exactly why the old one failed, and how the new architecture would prevent it from ever happening again.

This article is a distillation of that unique, battle-tested knowledge. Before we talk about the caveats of live-migrating a hyperscale network (which I will cover in my next post), we need to understand the fundamental design flaws of legacy systems like neutron and what are modern alternatives like OVN<label for="sn-5" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-5" class="margin-toggle"/><span class="sidenote"><a href="https://www.ovn.org/en/">OVN (Open Virtual Network)</a> replaces legacy Neutron Python agents with lightweight, C-based daemons and uses OVSDB for distributed state management, significantly improving scaling capabilities.</span> and Sprut. Here is exactly why OpenStack Neutron breaks at hyperscale, and how modern SDNs solve the bottleneck.

# I. Reality of building scalable clouds

Cloud computing is everywhere these days. The majority prefer hyperscalers, while others go for local or custom solutions. Building a cloud from scratch is a long, demanding project, so many opt for ready-to-go platforms where the groundwork is done, like OpenStack. While OpenStack is a monumental open-source achievement, it is fundamentally better suited for private clouds. In a public cloud at hyperscale, you hit hard limits.

To understand why, we have to look at SDN (Software-Defined Networking). SDN is not just a tool for configuring many devices at once, like Ansible. It is a complete paradigm shift where the control plane is separated from the networking hardware into a single, centralized entity. SDN is the backbone of the cloud. It virtualizes hardware, isolates tenants, and provides the rapid elasticity (or autoscaling) and resource pooling required by NIST cloud standards. Without a working SDN, you have no cloud. But developing an SDN is incredibly hard, and fundamental flaws in its architecture can force you to rip it out and start over or do a very costly migration (like we did).

# II. The Anatomy of a Cloud SDN

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_taxonomy.png)

Before we dive into how OpenStack Neutron fails, we need to define what a modern Software-Defined Network actually looks like.

At its core, SDN breaks vertical integration. Traditionally, networking vendors built routers and switches that contained both the forwarding hardware and the complex routing logic inside the same physical box. SDN rips these apart, separating the network's "brain" from its "muscle" (as seen in Figure 1, Column A).
* **The Data Plane (The Muscle)**: The underlying hardware or virtual switches become simple, fast forwarding devices. They don't think; they just move packets based on strict rules pushed down to them.
* **The Control Plane (The Brain)**: The routing logic, security groups, and intelligence are moved to a logically centralized Network Operating System (NOS).<label for="sn-6" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-6" class="margin-toggle"/><span class="sidenote">Early SDN relied heavily on the OpenFlow protocol to dictate every single traffic flow to the data plane. We quickly learned this is incredibly wasteful and unscalable in cloud environments. Modern SDNs rely more on hardware offloading and broader configuration states.</span>
* **The Management Plane (The Business Logic)**: This is where human operators and automated network applications define high-level business requirements and pass them to the Control Plane.

Because the network is now controlled by software sitting on a central server, you can program your infrastructure the same way you write an app for your phone. This makes the network flexible, adaptable, and incredibly easy to evolve.

However, there is a massive tradeoff: the system becomes vastly more complicated. By extracting the brain, we create a centralized single point of failure.<label for="sn-7" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-7" class="margin-toggle"/><span class="sidenote">In distributed systems engineering, the Control Plane is bound by the CAP theorem. To survive hardware failures, the controller must be distributed across multiple servers, introducing complex state-synchronization problems-which is exactly what kills Neutron.</span> The controller architecture must be designed flawlessly to survive at scale.

## The Two Jobs of a Hyperscaler's SDN
It is important to note that we are talking specifically about cloud SDN. A massive, distributed SDN like Neutron or Sprut is completely over-engineered for a standard enterprise datacenter with 20 hypervisors. It is also often detrimental in High-Performance Computing (HPC) clusters.<label for="sn-8" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-8" class="margin-toggle"/><span class="sidenote">In an HPC cluster, raw performance is everything. The CPU overhead and latency introduced by virtualizing the network with VXLAN encapsulation (overlays and underlays) is usually an unacceptable tradeoff. HPC relies on low-latency bare-metal technologies like InfiniBand or RoCE instead.</span>

But in a public hyperscaler, SDN is mandatory. It is the final piece of the puzzle-alongside software-defined compute and storage-that makes a cloud actually function according to NIST standards. In a cloud environment, the SDN has two massive responsibilities:

**1. Network Virtualization (Resource Pooling & Isolation)**
Before SDN, achieving multi-tenancy required a network engineer to manually configure VLANs and firewall rules. SDN dynamically virtualizes the physical network resources. This allows thousands of isolated users to share the exact same physical cables and switches safely, ensuring that one user's traffic spike doesn't cause a bottleneck for another (adhering to the NIST requirement for Measured Service).

**2. API-Driven Automation (Elasticity & Self-Service)**
Without SDN, virtual machines spin up in seconds, but the network takes hours or days to configure. By exposing a central Northbound API, the SDN allows for On-Demand Self-Service. Load balancers, firewalls, and routers can be dynamically provisioned by automated scripts or users clicking a button in a UI, providing the Rapid Elasticity that defines modern cloud computing.

# III. Neutron architecture: Design and Scaling Considerations

The architecture of OpenStack Neutron has distinct advantages, as it was originally designed to provide network-as-a-service for private clouds and enterprise environments. In deployments with a moderate number of hypervisors, its design works brilliantly to abstract complex networking.

Fundamentally, Neutron is not a custom packet-forwarding engine. It is essentially a distributed set of Python daemons that act as a "glue" layer. It ties together and orchestrates existing, battle-tested Linux networking tools-like Open vSwitch (OVS), `iptables`, network namespaces (netns), and dnsmasq. Because it is open-source and modular, it is highly extensible and relatively easy to understand. However, to tie all these disparate tools together, Neutron relies on centralized coordination.

At hyperscale, this architecture faces severe challenges. Its core structural vulnerability is a heavy reliance on a single message queue system (RabbitMQ) to synchronize imperative state commands across thousands of distributed agents.

## The Component Layers of Neutron

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

## Typical Neutron Hyperscale Deployment

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_deployment.png)

To understand why legacy OpenStack breaks at hyperscale, we must look at how it is physically deployed. The diagram above represents a standard, highly available deployment model.

Imagine this architecture stretched to its absolute limits. In a hyperscale scenario like ours, we are talking about roughly 3,000 bare-metal hypervisors hosting 160,000 VMs and 200,000 virtual ports. Here is a breakdown of how the components are physically distributed to achieve High Availability (HA).

### **1. The Physical Underlay: Spine-Leaf Architecture**
At the top of the diagram are the physical network switches arranged in a Spine-Leaf topology. Every server plugs into a Top-of-Rack (Leaf) switch, and every Leaf switch connects to every Core (Spine) switch.<label for="sn-10" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-10" class="margin-toggle"/><span class="sidenote">Traditional IT networks were built vertically (North-South) for traffic leaving the datacenter. In a cloud, the vast majority of traffic is "East-West" (VMs talking to other VMs, or compute talking to storage). Spine-leaf guarantees that any server is the exact same number of "hops" away from any other server, ensuring predictable, ultra-low latency.</span>

### **2. The Controller Cluster (The Control Plane)**
The right side of the diagram shows the "brain." In a deployment of 3,000 hypervisors, this cannot run on virtual machines. It requires a cluster of dedicated, massive bare-metal servers. To eliminate single points of failure, the control plane is stacked:
    * **HAProxy**: A load balancer that sits at the edge, distributing incoming API requests across active neutron-server workers.
    * **MySQL NeutronDB**: Deployed as a synchronous Galera cluster. Every database write is strictly replicated across the controller nodes to prevent split-brain scenarios and data loss.
    * **RabbitMQ Cluster**: The transport layer is also clustered to ensure message queues survive a hardware failure.

### **3. Compute Nodes (The Dataplane workers)**
These are the servers where the actual tenant VMs live. Every single one of these 3,000 compute nodes runs an ovs-agent daemon. This agent maintains a persistent, open connection back to the central RabbitMQ cluster, waiting for instructions to configure its local virtual switch.

### **4. Network Nodes**
Unlike compute nodes, Network Nodes do not run tenant VMs. They are dedicated, high-throughput bare-metal servers that run the L3 and DHCP agents. Routing thousands of gigabits of public internet traffic requires immense CPU overhead. If you placed this load on a standard compute node, it would steal CPU cycles from the paying tenants' VMs. High availability is achieved here using software redundancy, such as VRRP (Virtual Router Redundancy Protocol), which creates active/standby router pairs across multiple nodes.

While this hardware design is robust and effectively eliminates physical single points of failure, hardware reliability does not equal software scalability. Stretching a central RabbitMQ cluster across 3,000 hypervisors creates a ticking time bomb in the transport layer.

## Workflow Example: Port Creation (The Happy Path)

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/port_creation_workflow.png)

Now let’s look at how the software overlay manipulates the hardware. We will trace the most common operation in OpenStack: creating a virtual network port. In a healthy cloud, this relies on asynchronous message passing. Follow the numbered circles on the diagram.

**Step 1: The API Request.** When a tenant requests a new VM, the Compute service (Nova) sends a REST request to Neutron via the load balancer. The request is assigned to an ML2 plugin worker on the Controller Cluster.

**Step 2: Committing to the Database.** ML2 validates the request, generates a UUID, assigns MAC/IP addresses, and compiles security group rules. It writes this "Target State" into the central database. At this moment, the port logically exists in the control plane, but the physical dataplane knows nothing about it.

**Step 3: The Asynchronous Handoff.** ML2 packages the port configuration into an RPC message and drops it into the RabbitMQ cluster. The API worker’s job is done; it is free to handle the next user request.<label for="sn-11" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-11" class="margin-toggle"/><span class="sidenote">This asynchronous design is why OpenStack feels fast to the user. If the ML2 plugin had to establish a direct SSH connection to configure the compute node manually, API response times would skyrocket, and the central controller would quickly run out of worker threads.</span>

**Step 4: Physical Delivery and Execution.** The message travels through the physical Spine-Leaf switches until it reaches the specific hypervisor. The local ovs-agent receives the message, unpacks the JSON payload, and executes the local Linux commands to wire up the virtual interface. The VM boots, and traffic flows.

## Where the Cracks Begin to Form

This decoupled architecture is elegant on paper, but it introduces a fatal flaw at scale: it assumes the transport layer is perfectly reliable.

What happens if the RabbitMQ cluster is momentarily overwhelmed by 3,000 agents reporting their status simultaneously, and it drops the message? The local ovs-agent is blind. It has no idea Steps 1, 2, and 3 ever occurred. The central database says the port is ACTIVE, but on the hypervisor, the VM has no network connection.

Because OpenStack relies on decentralized agents, it requires a fail-safe to correct these desyncs. When an agent loses its connection to the message queue-even for a few seconds-it cannot trust its local state. To fix this, the agent is forced to trigger a blunt recovery mechanism known as a "Full Sync." In a hyperscale environment, this is often the exact mechanism that brings the entire cloud crashing down.

## Workflow Example: Network Node Full Sync

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_fullsync.png)

At its core, a cloud environment is a massive state machine. The fundamental promise of OpenStack is that the physical reality of the datacenter (Actual State) must strictly mirror the central database (Target State).

When an agent daemon is restarted-due to a crash, an upgrade, or a network blip-its local memory cache is invalidated. The agent wakes up blind. It cannot assume its local configuration is correct, because an API user might have deleted a router or added a hundred ports while the agent was offline. To reconcile this, the agent triggers a Full Sync. Let’s walk through the diagram above, using a Network Node as our example.

**Step 1**: The Call for Help. The moment the agent reconnects to RabbitMQ, it realizes it has lost synchronization. It fires off a direct RPC call to the controller: "Give me the complete configuration state for every single resource assigned to my hostname."

**Step 2**: API Reception. The request routes through HAProxy and is picked up by the ML2 Plugin.

**Step 3**: The Heavy Database Query. Unlike simple port creation, this is computationally expensive. ML2 must execute massive, complex JOIN queries against the database to gather the complete state of every logical router, floating IP, namespace, and DHCP pool scheduled to that node.

**Step 4**: Packaging the Payload. The API worker compiles all this data into a massive, monolithic JSON payload and drops it into a dedicated RabbitMQ reply queue.

**Step 5**: The Massive Delivery. The large data payload traverses the physical network. The agent downloads this state file, wipes its local caching discrepancies, and methodically configures the Linux dataplane to perfectly match the central database.

## The Advantage (And the Trap)
This mechanism exists because it makes operational recovery incredibly simple. If an SRE suspects a node has corrupted rules, they just restart the agent. The agent downloads the absolute truth from the database and flawlessly heals itself. In a 50-node private cloud, this works perfectly.

But look closely at Step 5. What happens if that massive payload takes too long to generate, or gets dropped by RabbitMQ because the queue is congested? The agent waits for a timeout period. When the message doesn't arrive, the agent assumes the request was lost, and it fiercely fires off Step 1 all over again.

Imagine a scenario where a rack switch reboots, causing 40 compute nodes to drop offline and request a full sync at the exact same second. As we will explore in the next section, this simple, self-healing loop transforms into a catastrophic, self-inflicted Denial of Service attack against the cloud's own control plane.


# IV. The "Full Sync" Disaster

While the Full Sync mechanism is a reliable safety net for minor, everyday operational hiccups, at hyperscale, it can horribly backfire. A single incident can cost a company millions of dollars.

Let's look at a catastrophic scenario: an entire datacenter loses power. Suppose we resolve the energy grid failure quickly, but our backup power fails to engage (which actually happened to me), causing the servers to shut down hard.

Getting the compute capacity back online-even for 160,000+ VMs, including massive 128-core, 1TB RAM instances-is surprisingly fast. The compute service can boot them in about 20 minutes.<label for="sn-12" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-12" class="margin-toggle"/><span class="sidenote">OpenStack's compute service (Nova) is highly distributed. Hypervisors can independently boot local VMs from local disk caches without waiting for a massive payload from a central bottleneck.</span> But without the network, compute is useless. The users' infrastructure cannot serve external requests, computational clusters cannot communicate during MapReduce operations, and the cloud remains effectively dead.

When those 3,000+ hypervisors power back on, their local network agents wake up with wiped memories. They are blind. For the network nodes, our 200,000+ virtual Neutron ports are basically gone. To fix this, every single agent simultaneously triggers a Full Sync.

## The Thundering Herd and the Timeout Loop

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_disaster.png)

This is where the architecture crumbles. All 3,000 hypervisors simultaneously fire RPC requests into RabbitMQ, demanding the state of their respective dataplane entities.

The messaging queue is not the only problem-you cannot just throw more RAM at RabbitMQ to fix this.<label for="sn-13" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-13" class="margin-toggle"/><span class="sidenote">RabbitMQ is optimized for high-throughput, small-payload message routing. Pumping multi-megabyte JSON Full Sync state files through it instantly causes severe memory pressure and queue drops.</span> The true bottleneck is the central control plane. The ML2 Plugin has to process these thousands of concurrent requests by executing massive, complex SQL JOIN queries against the central MySQL database.

The database CPU instantly spikes to 100%. Database locks occur. The queue overflows. Because the database is locked up, the ML2 Plugin takes an excruciatingly long time to generate the payload. If it takes longer than the agent's built-in timeout setting, the agent assumes its original request was lost in the network.

What does the agent do? It aggressively fires another Full Sync request into the queue. The system effectively performs a catastrophic Denial of Service (DDoS) attack on itself. Under this self-inflicted, crushing load, the system throughput drops to a crawl-processing only about 9,000 ports per hour. Because of this architectural flaw, a 30-minute power outage instantly turns into an agonizing 20+ hour recovery nightmare.

## The VIP Client Dilemma

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_port_order_problem.png)

During a 20-hour outage, business priorities become critical. Almost all large cloud providers have VIP clients that bring in 80%+ of the revenue. They are the driving force for any provider's growth. They also have tighter SLAs. Naturally, you want to restore their networks first.

With legacy Neutron, you can't.

As shown in the diagram above, the Full Sync restoration order is completely uncontrollable. The ovs-agent configures the dataplane at Layer 2. It only sees UUIDs, MAC addresses, and Linux namespaces. It has absolutely no concept of OpenStack "Tenants," "Quotas," or "VIP Billing Status"-that metadata only exists in the locked-up MySQL database.<label for="sn-14" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-14" class="margin-toggle"/><span class="sidenote">OpenStack Keystone handles identity and multitenancy. By the time a configuration reaches the physical hypervisor agent, all tenant context is stripped away for security and simplicity, leaving only raw network primitives.</span>

The agent just blindly asks for UUIDs. Additionaly, a VIP client's VMs are scattered across hundreds of different hypervisors for fault tolerance. Because you cannot tell the decentralized agents to prioritize specific tenant workloads, the VIP client's recovery is completely held hostage by the congested global queue.

Attempting manual intervention by SREs to speed this up is incredibly risky. Manually restarting services or tweaking queues can easily break the fragile Full Sync progress, forcing the timeout loop to start all over again (which was exactly what exacerbated the outage in our case and we had to start over).

## OpenStack Cells Won't Solve Neutron Scalability
When discussing OpenStack at hyperscale, architects often point to OpenStack Cells as the ultimate scaling silver bullet. And for the compute side, they are right.

OpenStack Cells allow you to partition a massive cloud deployment into smaller, isolated "shards." Instead of putting 3,000 hypervisors on one message queue, you break them into manageable chunks-say, 10 cells of 300 hypervisors each. Each cell gets its own local database and its own local message queue for compute operations, while the end-user still interacts with a single, unified global API. Without Cells, running 160,000 VMs would be impossible; Nova would collapse under its own weight just like Neutron.

However, there is a common misconception that deploying OpenStack Cells shards your network, too. It does not. OpenStack Cells only shard Nova (Compute). They do not cover Neutron (Networking), Cinder (Storage), or Glance (Images).<label for="sn-15" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-15" class="margin-toggle"/><span class="sidenote">While there have been community discussions and experimental attempts to shard Neutron using routed provider networks or multi-region setups, there is no native, push-button "Neutron Cells" equivalent to Nova Cells v2.</span>

This means that even if you perfectly partition your 3,000 hypervisors into beautifully isolated compute cells, the ovs-agent on every single one of those 3,000 machines is still reporting back to the exact same, monolithic, global Neutron RabbitMQ cluster and MySQL database. The compute plane is distributed, but the network control plane remains a massive single point of failure.

Because Cells cannot save the network, scaling an OpenStack cloud to hyperscale sizes inevitably forces a harsh realization: you cannot fix Neutron by partitioning the infrastructure around it. You have to rip out and replace the underlying SDN entirely.

## Looking Forward
It is worth noting that these cascading failures exist only in large-scale public installations of OpenStack. Originally, OpenStack was designed as a private cloud solution where you simply don't have this massive concentration of hypervisors and ports competing for the same message queue.

But VK Cloud is a hyperscaler and must protect its reputation and its clients. That is why a separate, proprietary SDN solution called SPRUT was created to combat the limitations of Neutron. While the open-source community eventually developed OVN to solve these same issues, OVN was not mature enough when SPRUT development began. Furthermore, we needed a system that maintained the exact same API to ensure a smooth, zero-downtime migration for our clients.

# V. The Architectural Shift: Moving to the Modern Models

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_sprut_comparsion.png)
Reference difference in neutron and sprut architectures (image 6) and how sprut solves sprut bottleneck.

Explain how sprut works in more details and it's architecture which exactly does 2 jobs we mentioned in the beginning! NFV + SDN (it's a really good abstraction, ref. image 7)

- Reference Column C ("The New Layer Model") in your diagram.

- Explain the shift from imperative RPC message queues to declarative REST APIs and state reconciliation (Intent-Based APIs).

- Introduce Sprut: Explain how migrating to a proprietary solution like VK Cloud's Sprut solved this. (You don't need code; focus on the architecture).

- Highlight the transition: Instead of a heavy central controller pushing massive state via RabbitMQ, modern systems separate the control plane into lightweight, distributed services that constantly poll their "Target State" vs. "Actual State" via HTTP REST APIs.

- ovn architectures

## Modern SDN design
(reference first diagram c)

## New Openstack standard: OVN

Before we dive into the proprietary solution (Sprut) that VK Cloud ultimately built, it is crucial to examine the open-source community's answer to Neutron's scalability crisis: OVN (Open Virtual Network).

First, let's clear up a common misconception: OVN does not replace Neutron entirely. Because OpenStack clients, automation scripts, and other services (like Nova) rely on the standard Neutron REST API, replacing the API would break the entire ecosystem.

Instead, OVN takes advantage of Neutron's Modular Layer 2 (ML2) design. It leaves the frontend API completely intact and radically guts and replaces the backend-stripping out RabbitMQ and the scattered Python agents in favor of a unified, database-driven control plane.

Historical Context: The OVN project was announced around 2015 by the Open vSwitch community. However, networking at a cloud scale is incredibly complex. It took roughly five years of intense development for OVN to achieve true feature parity with the legacy Neutron backend (supporting advanced features like Distributed Virtual Routing, complex Security Groups, and hardware offloading). It finally became the default backend in the OpenStack Ussuri release in 2020. This timeline is a critical piece of the puzzle regarding why VK Cloud began developing Sprut-we simply could not wait years for OVN to mature while our datacenter was growing at a hyperscale pace.

### The Component Layers of OVN
![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/ovn_arch.png)
Let’s break down how OVN fundamentally reorganizes the control plane layers, referencing the diagram above.

#### 1. The Controller Layer
The top layer still sits on your centralized control nodes, but it is vastly more sophisticated than legacy Neutron. It separates the "Cloud Management" state from the "Network Logic" state.
* **Neutron API**: The exact same Python REST API frontend. Users notice no difference.
* **Neutron DB (MySQL)**: Still present. It stores OpenStack-specific metadata (Tenant IDs, quotas, user billing info) that OVN doesn't care about.

* **ML2/OVN Plugin**: The universal translator. It catches API requests, reads the necessary metadata from MySQL, and translates the request into a purely logical network configuration.

* **OVN Northbound DB (OVSDB)**: The store for the Target State. The ML2 plugin writes logical entities here (Logical Switches, Logical Routers, Logical Ports). It knows what the network should look like, but not where the VMs physically live.

* **ovn-northd**: The central compiler daemon. It constantly monitors the Northbound DB and translates those high-level logical concepts into millions of low-level logical flows (MAC lookups, ACLs, routing tables).

* **OVN Southbound DB (OVSDB)**: The store for the Actual State and physical bindings. It contains the compiled logical flows from ovn-northd and tracks the physical location of every hypervisor (Chassis) and which VM is plugged into which host (Port Bindings).

#### 2. The Transport Layer (State Sync, not Messaging)
* **OVSDB Protocol**: This is the most critical architectural shift. OVN completely removes the RabbitMQ asynchronous message bus. Instead, it relies on the OVSDB protocol. This is a database synchronization protocol. Rather than shouting transient messages into a queue and hoping agents hear them, the control plane simply updates the Southbound DB. The agents independently connect to this database and synchronize only the rows that pertain to their specific physical hardware.

#### 3. The Agents Layer (The Unified Controller)
* **OVN Controller (`ovn-controller`)**: Legacy Neutron required four or five different Python daemons (ovs-agent, l3-agent, dhcp-agent, metadata-agent) fighting for resources on every single hypervisor. OVN replaces all of them with a single, highly efficient daemon written in C. This single agent connects to the Southbound DB, downloads the physical flows, and handles L2 switching, L3 routing, DHCP, and NAT all by itself.

#### 4. The Dataplane Layer
* **OVS (ovs-vswitchd & ovsdb-server)**: The underlying dataplane remains Open vSwitch. The `ovn-controller` receives the state from the Southbound DB and programs it directly into the local OVS instance using OpenFlow rules. Because OVN handles L3 and DHCP natively inside the switch's flow tables, it largely eliminates the need to spin up hundreds of messy Linux Network Namespaces (NetNS) and dnsmasq processes on the compute nodes.

### The Paradigm Shift: Eliminating the Architectural Bottlenecks
![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_ovn_comparison.png)

If you look at the side-by-side comparison in the diagram above, you can see how OVN systematically dismantles the bottlenecks that made legacy Neutron so fragile at scale. The improvements boil down to a fundamental shift in how state is calculated, transported, and applied:
* **Pre-Calculated State vs. On-the-Fly SQL**: In legacy Neutron, the ML2 plugin acts as a middleman that has to generate configuration states dynamically. Every time an agent requests an update, ML2 hits the MySQL database with complex JOIN queries, risking a total database lock. In OVN, this logic is decoupled. The ovn-northd daemon acts as a continuous background compiler. It translates high-level network logic into low-level OpenFlow rules before the agents even ask for it, storing the finalized rules in the Southbound DB. When an agent needs its state, the database simply performs a lightweight "Read" operation of pre-calculated data.
* **State Synchronization vs. RPC Messaging**: Neutron used RabbitMQ—an asynchronous message broker—to distribute state. RPC (Remote Procedure Call) messages are inherently two-way, locking transactions that do not scale horizontally well for massive state synchronization. OVN replaces this with the OVSDB protocol, which is designed purely for database synchronization. More importantly, OVSDB can be deployed as a highly available Raft cluster with many replicas. You can have one Leader for writes and multiple Followers for read replicas, instantly distributing the load across the control plane.
* **Payload Efficiency and Dataplane Complexity**: To configure a Neutron compute node, the control plane had to send massive JSON payloads detailing every single `iptables` rule, DHCP lease, and routing table entry. The Python agents then translated these into heavy Linux commands. OVN bypasses this completely. Because OVN handles L3 routing and DHCP natively inside the virtual switch, the payload sent to the agent consists of tiny, highly optimized binary OpenFlow rules.
* **Agent Footprint**: Finally, as shown in the diagram, placing 4–5 heavy Python daemons (OVS, L3, DHCP, Metadata) on a single node created massive CPU overhead and multiplied the number of connections hitting the control plane. OVN consolidates this entire footprint into a single, lightweight C-daemon (`ovn-controller`).

By separating the "Cloud Management" state from the "Network Logic," and replacing a fragile message queue with a resilient, replicable database protocol, OVN created an architecture capable of supporting thousands of hypervisors without collapsing under its own weight.

### The Customization Trade-off: C vs. Python
While OVN's architecture is significantly more robust and scalable than legacy Neutron, it comes with a major disadvantage for cloud operators: Extensibility.

Legacy Neutron was written entirely in Python. If a cloud provider needed a custom networking feature, a proprietary traffic shaping rule, or a quick hotfix for a VIP client, their Site Reliability Engineers (SREs) could simply write a Python extension for the l3-agent or ovs-agent and deploy it.

OVN is entirely different. While the ML2/OVN integration plugin is Python, the core engines that actually do the work (ovn-northd and `ovn-controller`) are written in C.

Adding a fundamentally new dataplane feature is no longer a matter of writing a quick Python script. It requires writing C code, modifying the core OVN schema, compiling binaries, and often submitting the changes upstream to the Linux Foundation’s Open vSwitch community to ensure compatibility. For an agile cloud provider looking to rapidly deploy proprietary features, this language and architecture barrier represents a massive loss of flexibility.

## Proprietary solution: Sprut

*(Note: The technical architecture and diagrams discussed in this section are based on engineering materials published by VK Cloud, specifically their deep-dive on the Habr platform.)*

### Why VK Cloud Built Sprut (The Alternatives)

When VK Cloud realized that legacy Neutron could no longer support our hyperscale growth, we established a strict set of requirements for a replacement SDN:

1.  **100% Feature Parity:** Existing customer features could not be disabled; the new solution needed identical functionality out of the gate.
2.  **L3 ToR Architecture:** Data center networks were built on an "L3 per rack" principle, requiring external traffic to be routed via BGP to the Top-of-Rack switches (which breaks legacy L2 broadcast protocols like VRRP).
3.  **Massive Scalability:** The control plane had to shard or scale horizontally to eliminate bottlenecks and simplify incident resolution.
4.  **Seamless Migration:** We needed to migrate 160,000+ VMs to the new SDN with zero downtime and no impact on the clients.
5.  **Customization:** We required an open, modifiable architecture to rapidly build proprietary cloud features.

We evaluated the available open-source options against these requirements:

  * **Tungsten Fabric:** While it supported L3 ToR routing and had broad functionality, its massive, poorly documented codebase and complex technology stack made rapid implementation and operation nearly impossible for our timeline.
  * **Reworking Neutron:** A dead end. The necessary features were already there, but the fundamental architecture (centralized RPC queues) was inherently flawed for our scale. Reworking it would require a total rewrite of the core.
  * **OVN:** As discussed earlier, OVN is now the industry standard. However, Sprut's development began around 2018–2019, long before OVN had reached maturity or feature parity. Furthermore, OVN's core is written in C, making rapid Python-based customization difficult. Finally, VK Cloud's OpenStack deployment had diverged from the main upstream branch over the years to serve specific business needs, making it too difficult to cleanly integrate a bleeding-edge OVN build.

Faced with these limitations, VK Cloud made the decision to build **Sprut**—a custom, highly scalable SDN designed specifically to survive hyperscale workloads, written in approximately 15,000 lines of efficient code.

### The Paradigm Shift: From Events to a Closed Control Loop

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_sprut_comparsion.png)

Before looking at the final layer design, we must look at how Sprut fundamentally redesigned the Transport Layer (as seen in the comparison diagram above).

**The Death of the Queue (RabbitMQ -\> HTTP REST):**
Legacy Neutron relied on an *event-driven* model. When a port changed, the ML2 plugin pushed a message into a RabbitMQ queue hoping the agent would receive it. Sprut completely eliminates RabbitMQ. It replaces the transport layer with a standard, highly scalable **HTTP REST API**.

**The Closed Control Loop:**
By moving to HTTP REST, Sprut shifts from an "Event-Driven" model to a **"Closed-Loop"** control theory model:

1.  Sprut agents no longer wait passively for transient events to arrive from a central queue.
2.  Instead, they continuously poll the HTTP REST API to request their "Target State."
3.  The agent independently compares this Target State from the API against the "Actual State" of its local hardware.
4.  If there is a difference (e.g., a new port needs to be wired, or a deleted rule needs to be removed), the agent applies *only the difference* to the local dataplane to minimize the error and bring the system into alignment.

### The Component Layers of Sprut

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sprut_arch_detailed.png)

While the HTTP REST loop fixed the transport layer, the underlying Open vSwitch dataplane can still be incredibly complex to debug across thousands of nodes. To simplify this, Sprut adopted a strict separation of concerns, splitting the architecture into three distinct layers (as seen in the finalized diagram).

##### 1. The APP Layer (The Translator)

This layer ensures complete API compatibility with OpenStack, allowing for a seamless migration.

  * **Neutron API:** The standard OpenStack frontend (identical to the user experience in Neutron or OVN).
  * **Sprut Plugin:** This completely replaces the legacy ML2 plugin. It receives user requests (e.g., "Create a network") and translates them into a set of distinct instructions for the underlying SDN and NFV layers. It also includes a Nova Driver to ensure stable operation with the compute service.

##### 2. The NFV Layer (Network Function Virtualization)

This layer is responsible strictly for providing network features (primitives), not routing traffic.

  * **NFV Agents & Controller:** These components manage specific network entities like DHCP servers, Virtual Routers, Floating IPs, and Metadata Proxies.
  * **Selective Placement:** Unlike Neutron, which forces heavy agents (like the OVS and L3 agents) onto every single compute node, Sprut's NFV agents only run on dedicated nodes selected by the operations team. This conserves critical network resources, CPU, and RAM on the hypervisors for the paying tenants' VMs.

##### 3. The SDN Layer (The Connectivity Engine)

This layer is responsible purely for moving packets from Point A to Point B. It does not know what a "Router" or a "DHCP Server" is; it only operates using "Links" and "Endpoints."

  * **Virtual Switches & Tunnels:** When SDN agents launch on the bare-metal hypervisors, they create virtual switches and establish tunnels between them. Tunnels can be configured for full connectivity, partial connectivity, or separated regions.
  * **The Network Graph:** The SDN agents constantly monitor the tunnels by sending probe packets through them. They report the latency and status in real-time back to the SDN Controller. The controller uses this to build and constantly update a real-time graph where vertices are switches and edges are tunnels.
  * **Dijkstra’s Routing:** An Endpoint is a connection point to the switch system, and a Link connects exactly two Endpoints. When a connection is required, the SDN Controller uses Dijkstra's algorithm on the network graph to find the absolute shortest path. It generates traffic forwarding rules specifying the port sequence and pushes them to the agents, functioning very similarly to the OSPF routing protocol.

#### The Workflow: Creating a Virtual Network in Sprut

By separating these concerns, creating a network becomes a clean, programmatic cascade:

1.  **User Request:** A user asks the standard Neutron API to create a virtual network.
2.  **Translation:** The APP Layer (Sprut Plugin) receives the request and transforms it into a set of entities.
3.  **Provisioning Primitives:** The APP layer tells the NFV Layer to spin up the necessary unattached entities (like a DHCP server).
4.  **Logical Wiring:** The APP layer tells the SDN layer to create "Links" and "Endpoints" to connect these NFV entities together. At this point, the network exists, but it has no virtual machines.
5.  **Compute Attachment:** The user tells Nova to boot a VM. The Nova-compute agent starts the QEMU process and asks the APP layer to connect the VM to the network.
6.  **Finalizing the Route:** The APP layer generates a set of Link and Endpoint parameters to connect the VM to the switch. The SDN Controller determines the shortest path, and the closed-loop agent applies the rules to the local Open vSwitch. The VM is instantly online.

# VI. Solving the Full Sync Disaster

In Chapter IV, we explored the nightmare scenario: a datacenter loses power, bringing down 3,000 hypervisors. When the servers reboot, the local network agents wake up with wiped memories and blindly demand their configuration state, creating a catastrophic Denial of Service loop that can take legacy Neutron over 20 hours to resolve.

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_arch_comparison.png)

Looking at the side-by-side architectural comparison above, we must ask a logical question: **If a datacenter loses power, 3,000 hypervisors wake up blind in Neutron, OVN, and Sprut alike. All three systems face the exact same stampede of 3,000 agents demanding their state. So why does Neutron collapse into a 20-hour DDoS loop, while OVN and Sprut recover in minutes?**

The answer lies in how these new architectures fundamentally process, package, and transport that state. Notably, the OpenStack Foundation (with OVN) and VK Cloud (with Sprut) arrived at strikingly similar architectural conclusions independently. Here is how these modern designs solve the Full Sync Disaster.

## Pre-Calculated State: Stopping the Database Lock

The single biggest failure point during a Neutron recovery is the central database. When 3,000 Neutron agents ask for a Full Sync, the ML2 plugin has to construct the state *on the fly*. It hits the MySQL database and executes thousands of heavy, multi-table `JOIN` queries simultaneously to figure out how security groups, subnets, and ports relate to each specific hypervisor. MySQL immediately locks up under the CPU load.

Both OVN and Sprut solve this by cleanly separating "Cloud Management" (NeutronDB) from "Network Logic" (OVSDB / Sprut DB).

  * **In OVN:** The `ovn-northd` daemon acts as a continuous background compiler. Long before the outage occurs, it translates high-level API concepts into low-level OpenFlow rules and stores them statically in the Southbound DB.
  * **In Sprut:** A similar compilation happens before the agent ever requests it.

When 3,000 agents wake up in these modern systems, there is no compiling or SQL `JOIN`ing to do. The Southbound DB (or Sprut's HTTP REST backend) simply hands over the pre-calculated rows that belong to that chassis. It is a simple "Read" operation, which modern databases can perform millions of times a second without locking.

## The Right Transport: State Sync vs. RPC Queues

When OpenStack was designed in 2010, the architectural standard dictated that all services must communicate via an AMQP message bus (RabbitMQ). This is excellent for asynchronous, one-off tasks (e.g., "Start this VM"). However, configuring a dataplane is continuous state synchronization. Using a task queue for state sync is like trying to use a mailbox to stream a 4K movie. Furthermore, an RPC request is inherently a two-way, locking transaction that scales poorly.

Modern SDNs abandoned the queue in favor of protocols actually designed for state replication:

  * **OVSDB Read Replicas:** OVN uses the OVSDB protocol, which operates on a Raft cluster. You have one Leader for writes and multiple Followers for reads. When 3,000 hypervisors wake up, they connect to the read-only Followers, instantly distributing the stampede load across the cluster.
  * **HTTP REST Caching:** Sprut utilizes a standard HTTP REST API. If 3,000 nodes execute standard `GET` requests simultaneously, the autoscaling API fleet—backed by standard load balancers and caching layers—easily serves the pre-calculated Target State. There are no fragile message queues to overflow and no timeout-retry loops.

## Payload Efficiency: Micro-Diffs vs. Megabytes

To restore a legacy Neutron compute node, the Full Sync payload must contain massive JSON files detailing every single `iptables` rule, DHCP lease, and routing table entry. It is megabytes of data per node, processed by bloated Python wrappers executing standard Linux shell commands.

OVN and Sprut program the Open vSwitch directly in C using native OpenFlow rules. The payload difference is staggering.
  * OVN simply downloads highly optimized binary flow tables.
  * Sprut downloads a lightweight topology of "Links" and "Endpoints," compares its current state to the Target State, and applies *only the differences*. A smaller payload means the physical network doesn't congest, the agent processes the diff in milliseconds, and the node is online instantly.

## Agent Consolidation and the SDN/NFV Split

Finally, the physical footprint of the agents drastically changes the recovery math. Neutron required four bloated Python daemons (OVS, L3, DHCP, Metadata) on every hypervisor, multiplying the number of connections hitting the control plane.

OVN consolidates this entire footprint into a single, highly efficient C-daemon (`ovn-controller`). You still have 3,000 agents waking up, but they consume a fraction of the host's CPU while pulling a tiny payload.

Sprut took this footprint reduction a step further through its strict **SDN / NFV split**. By isolating the NFV agents (DHCP, Routers, Metadata) onto a few dozen dedicated network nodes, the 3,000 compute nodes *only* run the lightweight SDN agent. When the 3,000 compute nodes wake up blind, they only ask the API for their simple L2 switching topology. They do not have to sync complex L3 routing or DHCP configurations. By moving the heaviest network primitives off the hypervisors entirely, Sprut drastically reduces the stampede impact on the API, ensuring a smooth, predictable recovery at hyperscale.

# VII. Conclusion and key takeaways

- Summarize the main takeaway: You cannot run a hyperscale cloud on an architecture designed for enterprise data centers.

- The Teaser: End with a bridge to your next article. (e.g., "Fixing the communication bottleneck is only half the battle. In the next article, we will look at the Data Plane-how separating SDN from NFV prevents your virtual switches from collapsing under the weight of complex network services.")

## What is SDN? 8 layer model

- use OS analogy from https://www.cs.utsa.edu/~korkmaz/teaching/ds-resources/sharvari-papers/survey-2015-Kreutz-sdn-comp-survey.pdf
- Control and data plane
-- Controller is centralized but reserved
- Forwarding devices and net os
- Northbound southbound interfaces
- Packet walkthrough
- A practical real world example of setting up same rule in datacenter via SDN vs Classical Networking vs Ansible + Networking
-- Data + control plane is stored within one device (unlike SDN)
-- Configure via CLI which is limited
-- Individual networking configurations
-- Control plane needs to communicate and there can be a lot of racks
-- SDN architecture is complex too though.
- How to make more reliable
-- Clustering
-- Seperation by regions each with separate controller using east west protocol
-- Hierarchal networking controllers

We define an SDN as a network architecture with four
pillars.
1) The control and data planes are decoupled. Control functionality is removed from network devices
that will become simple (packet) forwarding
elements.
2) Forwarding decisions are flow based, instead of
destination based. A flow is broadly defined by a
set of packet field values acting as a match (filter)
criterion and a set of actions (instructions). In the
SDN/OpenFlow context, a flow is a sequence of
packets between a source and a destination. All
packets of a flow receive identical service policies
at the forwarding devices [25], [26]. The flow
abstraction allows unifying the behavior of different types of network devices, including routers,
switches, firewalls, and middleboxes [27]. Flow
programming enables unprecedented flexibility,
limited only to the capabilities of the implemented flow tables [9].
3) Control logic is moved to an external entity, the
so-called SDN controller or NOS. The NOS is a
software platform that runs on commodity server
technology and provides the essential resources
and abstractions to facilitate the programming of
forwarding devices based on a logically centralized, abstract network view. Its purpose is therefore similar to that of a traditional operating system.
4) The network is programmable through software
applications running on top of the NOS that interacts with the underlying data plane devices.
This is a fundamental characteristic of SDN, considered as its main value proposition

## How SDN is built? Neutron
- Describe neutron architecture and how it maps to general model
- Describe vk implementation
- Neutron architectural decisions and why they were made
-- advantages
-- disadvantages

## Neutron disaster scenario
- Why disaster happened and how Nova Cells could've solved this

## Proprietary decisions
- Sprut as a solution.
- More durable
- Simplier dataplane
- Requires more resources to operate


## Sources
https://habr.com/ru/companies/vktech/articles/1000332/

https://github.com/vktechdev/evpn_connector

https://cloudification.io/cloud-blog/understanding-openstack-nova-cells-scaling-compute-across-data-centers/

https://superuser.openinfra.org/articles/openstack-neutron-networking-in-cloud-demystified/

https://aptira.com/theory-of-everything-in-neutron/

https://wiki.openstack.org/wiki/Neutron

comprehensive survey on sdn - https://www.cs.utsa.edu/~korkmaz/teaching/ds-resources/sharvari-papers/survey-2015-Kreutz-sdn-comp-survey.pdf

