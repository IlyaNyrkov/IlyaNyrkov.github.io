---
layout: post
title: "Surviving 200,000 Ports: Why OpenStack Neutron Breaks at Hyperscale"
---
In this post, we are going to explore Software-Defined Networking (SDN) architectures at hyperscale - specifically, what happens when you push a cloud environment past 160,000 VMs, 200,000 virtual ports, and roughly 3,000 bare-metal hypervisors. To put that scale into perspective, OpenStack Neutron was originally designed for private enterprise environments and historically recommended a maximum of around 500 hypervisors.<label for="sn-1" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-1" class="margin-toggle"/><span class="sidenote">While aggressive tuning of RPC workers and database pools can push this limit higher, 500 nodes is widely considered the historical threshold where the default RabbitMQ message bus begins to severely bottleneck under Neutron's architecture.</span>

But as VK Cloud<label for="sn-2" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-2" class="margin-toggle"/><span class="sidenote"><a href="https://cloud.vk.com/en/main/">VK Cloud</a> is the enterprise B2B cloud division of VK (formerly Mail.ru Group), offering infrastructure, platform services, and machine learning tools.</span> rapidly grew into one of the largest public cloud providers in the CIS region - establishing itself as the primary alternative to Yandex<label for="sn-3" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-3" class="margin-toggle"/><span class="sidenote"><a href="https://yandex.cloud/en">Yandex Cloud</a> is the largest public cloud platform in Russia and the CIS region, often compared to an AWS or GCP equivalent for that market.</span> - we blew past those private-cloud limits. We pushed the architecture as far as it could go, until it simply couldn't go any further.

I did not design that original OpenStack architecture, nor did I write the code for our new proprietary SDN, Sprut. I joined as a Lead Solution Architect and inherited a massive system that had already suffered a catastrophic "full sync"<label for="sn-4" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-4" class="margin-toggle"/><span class="sidenote">A "full sync" occurs when a local Neutron agent loses its connection state and requests the entire network topology from the central server. At hyperscale, hundreds of agents doing this simultaneously creates a "thundering herd" that crushes the control plane.</span> outage a few years prior. When the same disaster happened a second time (now on my watch), I spearheaded the initiative to migrate our highest-paying clients away from Neutron to our new SDN (I also consulted development team on automated migration solution for small clients).

To prove to our clients that the new system would keep them safe, I spent months studying documented and extracting undocumented wisdom from our SRE and maintenance teams. I dug deep into the theoretical and practical limits of our SDN to understand exactly why the old one failed, and how the new architecture would prevent it from ever happening again.

This article is a distillation of that unique, battle-tested knowledge. Before we talk about the caveats of live-migrating a hyperscale network (which I will cover in my next post), we need to understand the fundamental design flaws of legacy systems like neutron and what are modern alternatives like OVN<label for="sn-5" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-5" class="margin-toggle"/><span class="sidenote"><a href="https://www.ovn.org/en/">OVN (Open Virtual Network)</a> replaces legacy Neutron Python agents with lightweight, C-based daemons and uses OVSDB for distributed state management, significantly improving scaling capabilities.</span> and Sprut. Here is exactly why OpenStack Neutron breaks at hyperscale, and how modern SDNs solve the bottleneck.

## I. Reality of building scalable clouds

Cloud computing is everywhere these days. The majority prefer hyperscalers, while others go for local or custom solutions. Building a cloud from scratch is a long, demanding project, so many opt for ready-to-go platforms where the groundwork is done, like OpenStack. While OpenStack is a monumental open-source achievement, it is fundamentally better suited for private clouds. In a public cloud at hyperscale, you hit hard limits.

To understand why, we have to look at SDN (Software-Defined Networking). SDN is not just a tool for configuring many devices at once, like Ansible. It is a complete paradigm shift where the control plane is separated from the networking hardware into a single, centralized entity. SDN is the backbone of the cloud. It virtualizes hardware, isolates tenants, and provides the rapid elasticity (or autoscaling) and resource pooling required by NIST cloud standards. Without a working SDN, you have no cloud. But developing an SDN is incredibly hard, and fundamental flaws in its architecture can force you to rip it out and start over or do a very costly migration (like we did).

## II. The Anatomy of a Cloud SDN

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sdn_taxonomy.png)

Before we dive into how OpenStack Neutron fails, we need to define what a modern Software-Defined Network actually looks like.

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

## III. Neutron architecture: Design and Scaling Considerations

The architecture of OpenStack Neutron has distinct advantages, as it was originally designed to provide network-as-a-service for private clouds and enterprise environments. In deployments with a moderate number of hypervisors, its design works brilliantly to abstract complex networking.

Fundamentally, Neutron is not a custom packet-forwarding engine. It is essentially a distributed set of Python daemons that act as a "glue" layer. It ties together and orchestrates existing, battle-tested Linux networking tools-like Open vSwitch (OVS), iptables, network namespaces (netns), and dnsmasq. Because it is open-source and modular, it is highly extensible and relatively easy to understand. However, to tie all these disparate tools together, Neutron relies on centralized coordination.

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

    3.1. **OVS Agent (Layer 2)**: Configures switching rules on the hypervisor's Open vSwitch. It is also historically responsible for implementing Security Groups (firewall rules) locally on the compute node via iptables.
    
    3.2. **L3 Agent**: Handles Layer 3 protocols, routing, and Floating IPs (NAT).

    3.3. **DHCP Agent**: Manages IP address assignment and static routes for virtual machines.

    3.4. **Metadata Agent**: Proxies requests from instances to the Nova metadata service.

4. **Dataplane**: This is the underlying Linux/This is the underlying Linux network infrastructure. It is important to note that netns (Network Namespaces) and daemons like dnsmasq are not Neutron agents themselves. They are standard Linux kernel features managed by the agents. For example, the DHCP agent spawns a unique dnsmasq process inside an isolated netns to serve IP addresses to a specific tenant network without overlapping with others.<label for="sn-9" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-9" class="margin-toggle"/><span class="sidenote">This modularity is why OpenStack created OVN (Open Virtual Network) as a modern replacement. OVN offloads almost all dataplane configuration (L2, L3, DHCP, Security Groups) into highly optimized OpenFlow rules within OVS, allowing Neutron to step back and act purely as a high-level API manager.</span>

### Typical Neutron Hyperscale Deployment

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_deployment.png)

To understand why legacy OpenStack breaks at hyperscale, we must look at how it is physically deployed. The diagram above represents a standard, highly available deployment model.

Imagine this architecture stretched to its absolute limits. In a hyperscale scenario like ours, we are talking about roughly 3,000 bare-metal hypervisors hosting 160,000 VMs and 200,000 virtual ports. Here is a breakdown of how the components are physically distributed to achieve High Availability (HA).

#### **1. The Physical Underlay: Spine-Leaf Architecture**
At the top of the diagram are the physical network switches arranged in a Spine-Leaf topology. Every server plugs into a Top-of-Rack (Leaf) switch, and every Leaf switch connects to every Core (Spine) switch.<label for="sn-10" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-10" class="margin-toggle"/><span class="sidenote">Traditional IT networks were built vertically (North-South) for traffic leaving the datacenter. In a cloud, the vast majority of traffic is "East-West" (VMs talking to other VMs, or compute talking to storage). Spine-leaf guarantees that any server is the exact same number of "hops" away from any other server, ensuring predictable, ultra-low latency.</span>

#### **2. The Controller Cluster (The Control Plane)**
The right side of the diagram shows the "brain." In a deployment of 3,000 hypervisors, this cannot run on virtual machines. It requires a cluster of dedicated, massive bare-metal servers. To eliminate single points of failure, the control plane is stacked:
    * **HAProxy**: A load balancer that sits at the edge, distributing incoming API requests across active neutron-server workers.
    * **MySQL NeutronDB**: Deployed as a synchronous Galera cluster. Every database write is strictly replicated across the controller nodes to prevent split-brain scenarios and data loss.
    * **RabbitMQ Cluster**: The transport layer is also clustered to ensure message queues survive a hardware failure.

#### **3. Compute Nodes (The Dataplane workers)**
These are the servers where the actual tenant VMs live. Every single one of these 3,000 compute nodes runs an ovs-agent daemon. This agent maintains a persistent, open connection back to the central RabbitMQ cluster, waiting for instructions to configure its local virtual switch.

#### **4. Network Nodes**
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


## IV. The "Full Sync" Problem

While the Full Sync mechanism is a reliable safety net for minor, everyday operational hiccups, at hyperscale, it can horribly backfire. A single incident can cost a company millions of dollars.

Let's look at a catastrophic scenario: an entire datacenter loses power. Suppose we resolve the energy grid failure quickly, but our backup power fails to engage (which actually happened to me), causing the servers to shut down hard.

Getting the compute capacity back online—even for 160,000+ VMs, including massive 128-core, 1TB RAM instances—is surprisingly fast. The compute service can boot them in about 20 minutes.<label for="sn-12" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-12" class="margin-toggle"/><span class="sidenote">OpenStack's compute service (Nova) is highly distributed. Hypervisors can independently boot local VMs from local disk caches without waiting for a massive payload from a central bottleneck.</span> But without the network, compute is useless. The users' infrastructure cannot serve external requests, computational clusters cannot communicate during MapReduce operations, and the cloud remains effectively dead.

When those 3,000+ hypervisors power back on, their local network agents wake up with wiped memories. They are blind. For the network nodes, our 200,000+ virtual Neutron ports are basically gone. To fix this, every single agent simultaneously triggers a Full Sync.

### The Thundering Herd and the Timeout Loop

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_disaster.png)

This is where the architecture crumbles. All 3,000 hypervisors simultaneously fire RPC requests into RabbitMQ, demanding the state of their respective dataplane entities.

The messaging queue is not the only problem—you cannot just throw more RAM at RabbitMQ to fix this.<label for="sn-13" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-13" class="margin-toggle"/><span class="sidenote">RabbitMQ is optimized for high-throughput, small-payload message routing. Pumping multi-megabyte JSON Full Sync state files through it instantly causes severe memory pressure and queue drops.</span> The true bottleneck is the central control plane. The ML2 Plugin has to process these thousands of concurrent requests by executing massive, complex SQL JOIN queries against the central MySQL database.

The database CPU instantly spikes to 100%. Database locks occur. The queue overflows. Because the database is locked up, the ML2 Plugin takes an excruciatingly long time to generate the payload. If it takes longer than the agent's built-in timeout setting, the agent assumes its original request was lost in the network.

What does the agent do? It aggressively fires another Full Sync request into the queue. The system effectively performs a catastrophic Denial of Service (DDoS) attack on itself. Under this self-inflicted, crushing load, the system throughput drops to a crawl—processing only about 9,000 ports per hour. Because of this architectural flaw, a 30-minute power outage instantly turns into an agonizing 20+ hour recovery nightmare.

### The VIP Client Dilemma

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/full_sync_port_order_problem.png)

During a 20-hour outage, business priorities become critical. Almost all large cloud providers have VIP clients that bring in 80%+ of the revenue. They are the driving force for the provider's growth. Naturally, you want to restore their networks first.

With legacy Neutron, you can't.

As shown in the diagram above, the Full Sync restoration order is completely uncontrollable. The ovs-agent configures the dataplane at Layer 2. It only sees UUIDs, MAC addresses, and Linux namespaces. It has absolutely no concept of OpenStack "Tenants," "Quotas," or "VIP Billing Status"—that metadata only exists in the locked-up MySQL database.<label for="sn-14" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-14" class="margin-toggle"/><span class="sidenote">OpenStack Keystone handles identity and multitenancy. By the time a configuration reaches the physical hypervisor agent, all tenant context is stripped away for security and simplicity, leaving only raw network primitives.</span>

The agent just blindly asks for UUIDs. Furthermore, a VIP client's VMs are scattered across hundreds of different hypervisors for fault tolerance. Because you cannot tell the decentralized agents to prioritize specific tenant workloads, the VIP client's recovery is completely held hostage by the congested global queue.

Attempting manual intervention by SREs to speed this up is incredibly risky. Manually restarting services or tweaking queues can easily break the fragile Full Sync progress, forcing the timeout loop to start all over again (which was exactly what exacerbated the outage in my case).

### OpenStack Cells Won't Solve Neutron Scalability
When discussing OpenStack at hyperscale, architects often point to OpenStack Cells as the ultimate scaling silver bullet. And for the compute side, they are right.

OpenStack Cells allow you to partition a massive cloud deployment into smaller, isolated "shards." Instead of putting 3,000 hypervisors on one message queue, you break them into manageable chunks—say, 10 cells of 300 hypervisors each. Each cell gets its own local database and its own local message queue for compute operations, while the end-user still interacts with a single, unified global API. Without Cells, running 160,000 VMs would be impossible; Nova would collapse under its own weight just like Neutron.

However, there is a common misconception that deploying OpenStack Cells shards your network, too. It does not. OpenStack Cells only shard Nova (Compute). They do not cover Neutron (Networking), Cinder (Storage), or Glance (Images).<label for="sn-15" class="margin-toggle sidenote-number"></label><input type="checkbox" id="sn-15" class="margin-toggle"/><span class="sidenote">While there have been community discussions and experimental attempts to shard Neutron using routed provider networks or multi-region setups, there is no native, push-button "Neutron Cells" equivalent to Nova Cells v2.</span>

This means that even if you perfectly partition your 3,000 hypervisors into beautifully isolated compute cells, the ovs-agent on every single one of those 3,000 machines is still reporting back to the exact same, monolithic, global Neutron RabbitMQ cluster and MySQL database. The compute plane is distributed, but the network control plane remains a massive single point of failure.

Because Cells cannot save the network, scaling an OpenStack cloud to hyperscale sizes inevitably forces a harsh realization: you cannot fix Neutron by partitioning the infrastructure around it. You have to rip out and replace the underlying SDN entirely.

### Looking Forward
It is worth noting that these cascading failures exist only in large-scale public installations of OpenStack. Originally, OpenStack was designed as a private cloud solution where you simply don't have this massive concentration of hypervisors and ports competing for the same message queue.

But VK Cloud is a hyperscaler and must protect its reputation and its clients. That is why a separate, proprietary SDN solution called SPRUT was created to combat the limitations of Neutron. While the open-source community eventually developed OVN to solve these same issues, OVN was not mature enough when SPRUT development began. Furthermore, we needed a system that maintained the exact same API to ensure a smooth, zero-downtime migration for our clients.

## V. The Architectural Shift: Moving to the Modern Models

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_sprut_comparsion.png)
Reference difference in neutron and sprut architectures (image 6) and how sprut solves sprut bottleneck.

Explain how sprut works in more details and it's architecture which exactly does 2 jobs we mentioned in the beginning! NFV + SDN (it's a really good abstraction, ref. image 7)


- Reference Column C ("The New Layer Model") in your diagram.

- Explain the shift from imperative RPC message queues to declarative REST APIs and state reconciliation (Intent-Based APIs).

- Introduce Sprut: Explain how migrating to a proprietary solution like VK Cloud's Sprut solved this. (You don't need code; focus on the architecture).

- Highlight the transition: Instead of a heavy central controller pushing massive state via RabbitMQ, modern systems separate the control plane into lightweight, distributed services that constantly poll their "Target State" vs. "Actual State" via HTTP REST APIs.

- ovn architectures

### Modern SDN design
(reference first diagram c)

### New Openstack standard: OVN

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/ovn_arch.png)

### Proprietary solution: Sprut

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/sprut_arch_detailed.png)

Developed by VK cloud
- reasons why separate soluion was used instead of OVN
- developed for few years before I joined vk I lead large scale migration to new SDN and had to figure out how it worked in details.

## VI. Conclusion and key takeaways

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


## Side notes

https://habr.com/ru/companies/vk/articles/763760/
- OpenFlow, FlowVisor, OpenvSwitch
- Active Networking
- SDN: Control and Data layer

Удешевление вычислительных ресурсов означало, что один простой сервер может хранить в себе и выполнять всю необходимую для большой сети логику по маршрутизации. Тут же обнаружились и дополнительные плюсы: бэкапы стало делать гораздо проще. 

4D: Data plane (процессинг пакетов на основе правил), Discovery plane (сбор топологии и измерение), Dissemination plane (размещение правил для обработки пакетов), Decision plane (логическое объединение контроллеров, реализующих обработку пакетов). Появились определяющие проекты: Ethane, SANE. Ethane стал предтечей OpenFlow - дизайн системы стал первой версией OpenFlow API. 


Openflow
Data plane с открытым интерфейсом, State management layer для поддержки консистентного понимания состояния сети и Control layer. 

Open virtual network in openstack

SDN стали основополагающим сервисом для облачных платформ. Одна из основных характеристик облака - это эластичность. Она, а также скорость и другие характеристики развёртываний подразумевают, что нижележащие сервисы должны быть максимально автоматизированы, отслеживая жизненный цикл объектов, на основе которых функционирует развёртываемый сервис. Например, для виртуальных машин это могут быть хранилище и сетевые настройки. Когда вы настраиваете виртуальную машину в любом облаке и переходите на вкладку сетей, чтобы настроить, в какой подсети будет виртуалка, вы участвуете в формировании запроса для SDN.

Лимиты архитектуры. Сам Mirantis говорит о том, что 500 узлов - это предел и Neutron не позиционируется как бесконечно масштабируемый продукт, который выдержит облачные масштабы. 


Переехать на другую SDN: Tungsten Fabric/OpenContrail, OpenDayLight, OVN. 
