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

The architecture of OpenStack Neutron has its distinct advantages, having been originally designed to provide network-as-a-service for private clouds and enterprise environments. In these environments-typically involving a moderate number of hypervisors-its design works well to abstract complex networking. However, in large-scale public cloud deployments, the architecture faces significant scaling challenges. A core structural issue at scale is its heavy reliance on a centralized control plane communicating over a single message queue system (RabbitMQ) to synchronize state across hundreds or thousands of distributed agents.

### Neutron architecture

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_arch.png)

The Neutron architecture is separated into distinct layers: 
1. **Controller Layer**: The Neutron API receives requests. The ML2 (Modular Layer 2) Plugin implements the core logic for L2 networking. It receives the network configuration, writes the persistent state to the Database (MySQL/MariaDB), and dictates the network topology.
2. **Transport Layer**: A message broker, typically RabbitMQ, sits between the controller and the compute nodes. It handles the heavy lifting of routing RPC (Remote Procedure Call) messages between the plugins and the distributed agents.
3. **Agents**: These are Python daemons running on compute or network nodes that translate logical configurations into actual dataplane rules.

    3.1. **OVS Agent (Layer 2)**: Configures switching rules on the hypervisor's Open vSwitch (OVS). It is also typically responsible for implementing Security Groups (firewall rules) locally on the compute node.
    
    3.2. **L3 Agent**: Handles Layer 3 protocols, routing, and Floating IPs (NAT).

    3.3. **DHCP Agent**: Manages IP address assignment and static routes for virtual machines.

    3.4. **Metadata Agent**: Proxies requests from instances to the Nova metadata service.

4. **Dataplane**: This is the underlying Linux/Network infrastructure. Note: NetNS (Network Namespaces) and Daemons (like dnsmasq or radvd) are not agents themselves. They are Linux kernel features and processes managed by the agents. For example, the L3 and DHCP agents use NetNS to isolate tenant traffic, and the DHCP agent spawns dnsmasq daemons to serve IP addresses.

### Neutron typical datacenter deployment

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_deployment.png)

### Neutron workflow example: port creation

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/port_creation_workflow.png)

### Neutron workflow example: network node full sync

![alt text](/assets/img/2026-04-17-surviving-200k-ports-why-openstack-neutron-breaks-at-hyperscale/neutron_fullsync.png)


## IV. The "Full Sync" Problem

In general full sync is a good idea that allows to make network consistent again relatively easy (especially for operations team). But it can horribly backfire and one incident can cost a company millions of dollars. The way Neutron works, when ML2 Plugin receives new network updates (create network, create port, update a port and etc.) it pushes the messages for responsible agents via rabbit MQ it also records the state into controllers database.

The queue helps to not overwhelm the agents with work and weaker hardware can used for these agents (that configure the dataplane e.g. ovs, l3, dhcp, meta).

The problem with the queue is that messages can be lost and it's not easy to figure out that something actually was missed. For that full sync was created, agent requests the info from control database with all state information about dataplane for which agent is responsible. So using this mechanism we can do something simillar to git pull in software development and make our ports "up to date". The problem that this mechanism still uses rabbitMQ to get this information about the dataplane state. Even getting info about few virtual ports and applying them can take few minutes. Now imagine (and this actually happened) if datacenter looses energy or just few network nodes (servers that hold network agents) loose power. Let's look at the case when whole datacenter looses power. Suppose we solved the problem with energy and got electricity back, but our backup power didnt work (this also happened to me) and servers turned off. Getting back vms even when there is 160k+ vms (including large ones 128 cpu 1024 gb ram) can take at most 20 minutes, without the network Compute is useless. User's infrastructure cannot serve outside requests, computational clusters cannot communicate with each other during map reduce operations and many more. The problem is that with 160k+ vms we have something like 200k+ virtual neutron ports (almost real numbers I but not exact because it's a private information). For network nodes and agents these ports are basically gone, we need a full sync. For simplicity I will mention only neutron network ports as a primitive that dataplane manages as it is the lowest level primitive necessary for vm to communicate, besides vms there are other things like networks, subnetworks, virtual routers, load balancers and etc.

Now remember that it takes a few minutes to setup just a few ports, now remember that all of these updates have to go through one single messaging queue (or more accurately distrubuted but not really autoscalable, we cannot suddenly make it larger just for cases of outage) this queue is not really infinitely scalabale and has it's limits, it is able to process aprox 9k ports per hour. You see a problem? We managed to quickly get the power back up and running, we have mechanisms to spin up vms back automatically relatively fast, our ceph cluster and storage is also reliable and did not loose any data, we still have a third piece network. Because of the network our 30 min outage turned in almost whole day outage 20+ hrs.



There is another terrible problem. Almost all of the hyperscalers and large cloud providers (vk cloud is not exception) have a few clients, VIP clients that bring 80%+ of revenue. Ofcourse any cloud provider would say to you that they care about all and everyone, but in a real business world ofcourse top paying customers are top priority, they are a driving force for cloud providers growth, with large customers provider cannot really grow (hardware count, employees, functionaility and many more). In Fullsync the order of ports that are being restored is not really controlable, you cant say to neutron restore vip clients first. Network nodes (that hold agents that configure dataplane) dont really now about tenants (tenant is a like a project or account in AWS basically isolated set of workload, quotas, billing) and just restore all missing ports of vms that were there. So you get a stream of updates requested by each agent responsible for set of hypervisors which hold set of vms. One client wont hold all of his vms on one hypervisors, it can potentially be done via labels or taints but it's also bad idea especially if it's some RAFT database cluster, better to have vms of a cluster on different hypervisors preferably in different datacenters even. So you have scattered vms of one expensive clients across different hypervisors and you cannot really speedup restoration for this particular client without some significant efforts or manual intervention by SREs. And manual intervention is risky too because it can break full sync process and everything will have to be started over again or some portion of progress will be lost e.g. due to error from engineer (also what happened in my case).



There other events or errors that can cause this cascading failure that can happen on several layers of the technical stack (server equipment failure, rabbitMQ outage and many more).



Worth noting that these problems exist only in large scale public installations of openstack, originally openstack as a private cloud solution where you dont have this large amount of hypervisors, vms, users. But VK cloud is a hyperscaler and needs to keep it's great reputation, that's why a separate solution called SPRUT was created to combat problems of neutron. There is also alternative created by openstack called OVN but by the time SPRUT started development OVN wasnt mature enough. Also same API was needed for the smooth migration for clients from neutron to new stable sprut sdn that I am going to talk about in the next article.

## V. The Architectural Shift: Moving to the Modern Models

Reference difference in neutron and sprut architectures (image 6) and how sprut solves sprut bottleneck.

Explain how sprut works in more details and it's architecture which exactly does 2 jobs we mentioned in the beggining! NFV + SDN (it's a really good abstraction, ref. image 7)


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
