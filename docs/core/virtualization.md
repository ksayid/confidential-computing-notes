---
title: Virtualization and Hypervisors
parent: Core Concepts
nav_order: 5
---

# Virtualization and Hypervisors

Before confidential computing, before the cloud, the problem virtualization solves is much older: a "server" traditionally meant the whole package — hardware plus OS plus storage plus applications, referred to by function (the SQL server, the DNS server, the file server). When one of those servers got overloaded, the answer was to buy another physical machine. Clusters could give you round-the-clock availability and fault tolerance, but clusters have scalability limits, and not every application works clustered. Virtualization rewrites the assumption underneath: **one physical machine can run many virtual machines, each with its own (potentially heterogeneous) operating system, isolated from one another**. The same physical box that previously ran one OS becomes a virtual system hosting dozens — and the practical consequences are economy (fewer physical servers, less floor space, fewer licenses), dynamism (capacity on demand, new servers in minutes instead of weeks), and disaster recovery (servers brought up at an alternative site in minutes).

This note covers the cloud-agnostic substrate: what virtualization is, how a hypervisor implements it, the features hypervisors offer at the cluster level, and the security model that emerges when many tenants share one physical machine. Azure-specific Hyper-V details — partition model, VMBus, VSP/VSC, root partition — live in §8 of the [Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#8-the-hypervisor) note; this file establishes the generic concepts that note builds on.

## What virtualization is

**Virtualization** is a layer of abstraction between hardware and software that decouples the physical hardware from the operating system. The interaction between the two layers gets simpler, and IT resource utilization and flexibility both go up. Concretely: a customer-management environment might run on Operating System 1 while a testing environment runs on Operating System 2, both sitting on top of a **Virtual Machine Monitor (VMM)** which virtualizes all the resources — processors, memory, secondary storage, networks — and allocates them to the virtual machines.

The vocabulary is small and worth getting right:

| Term | Meaning |
|---|---|
| **Host machine** | The physical machine running the virtualization software. Owns the real CPU, memory, disk, and network resources that the VMs use. |
| **Virtual machine** | A machine run and maintained by the virtualization software. Each VM lives as a single file (or a small folder of files) on the host, and behaves to its guest software as if it were running on its own physical PC. |
| **Virtualization software** | Generic term for software that runs VMs on a host. |
| **VMM (Virtual Machine Monitor)** | The software that implements virtualization. It virtualizes hardware resources, presents emulated devices to VMs, and handles resource allocation. |

Three properties make virtualization the substrate cloud computing rests on. First, **partitioning**: many operating systems and applications coexist on a single physical system by separating the available resources. Second, **encapsulation**: a VM can be represented as a single file, identified by the service it provides — an encapsulated business service that can be picked up and moved as one thing. Encapsulation also keeps one application from interfering with another. Third, **isolation**: a VM is isolated from its host and from sibling VMs. If one virtual instance crashes, the others are unaffected; data is not shared between virtual containers. These three together — partitioning, encapsulation, isolation — are what make the cloud's economic model possible. The same agility, flexibility, and scalability that lets a provider pack many tenants onto one host comes from the fact that VMs are just files: cloned in seconds, migrated to another host while running, paused, snapshotted, rolled back.

### Hosted vs. bare-metal VMMs

The fundamental design choice is where the VMM sits relative to the hardware and any host OS.

In the **hosted approach**, the VMM runs *on* a conventional operating system, installed and started like any other application (originally on 8086-based systems, now ubiquitous). Because it leverages the host OS for device support and physical resource management, it supports the widest range of hardware configurations — but it pays for that with an extra layer of indirection.

In the **bare-metal approach** (also called the **hypervisor architecture**), the VMM is the first layer of software on a clean x86 system: no host OS underneath. It has direct access to hardware, which makes it more efficient, more scalable, more robust, and faster than the hosted approach.

| | Hosted | Bare-metal (hypervisor architecture) |
|---|---|---|
| Runs on | A standard host OS, as an application | Directly on hardware (first software layer on a clean x86 system) |
| Device / resource management | Relies on host OS | Direct hardware access |
| Hardware support | Widest range of configurations | Narrower; requires explicit support |
| Efficiency | Lower | Higher — better scalability, robustness, performance |

This taxonomy reappears in §The hypervisor as the Type 1 / Type 2 distinction.

## Why virtualization matters

The benefits fall into five buckets that recur in every cloud architecture discussion.

**Availability and reliability** — a software failure inside one VM does not affect any other VM on the same host. **Security** — environments with different security requirements run in different VMs, each with the guest OS and tools most suited to its workload; an attack on one VM, because of isolation, does not compromise the others. **Cost** — consolidating many smaller servers onto fewer powerful ones reduces hardware spend, operations headcount, floor space, and software licenses. **Adaptability to workload variations** — when load on one VM spikes, the hypervisor can reallocate resources and priorities across VMs; autonomic-style schemes can even move processors from one VM to another dynamically. **Load balancing** — because the VMM completely encapsulates a VM's software state, the VM can migrate easily to other platforms, opening the door to performance improvement through rebalancing.

From a business angle, the same properties translate into four readiness arguments: virtualization is **economical** (fewer physical servers, lower procurement and operational costs); it is **dynamic** (capacity on demand, services deployed in minutes); it makes **disaster recovery** efficient (servers brought up at an alternative site in minutes); and it is a **business-readiness assessment tool** (the shared-computing model lets an enterprise understand its infrastructure requirements in a virtualized environment without first paying to implement them physically).

## The hypervisor

The term **hypervisor** was introduced in **1956 by IBM**, naming software distributed with IBM RPQ for the IBM 360/65. A hypervisor is a hardware-virtualization technique that lets multiple guest OSes run at the same time on one host; it is directly responsible for hosting and managing VMs on the host server. It divides the hardware into multiple logical partitions and ensures isolation among them.

The hypervisor takes control **the moment the system is powered on**. It surveys memory, CPU, I/O, and other available resources, and it owns and controls every resource global to the system. For memory specifically, physical memory is broken into **PMBs (Physical Memory Blocks)** and logical memory into **LMBs (Logical Memory Blocks)**; PMBs are mapped to LMBs, and the hypervisor performs virtual memory management via a **global partition page table**, intercepting any partition that tries to step outside its allocation.

The vocabulary shifts slightly once the hypervisor enters:

- **Host** = the physical server + the hypervisor.
- **Guest VM** or **guest operating system** = a VM that runs on the host. Guest VMs can run on hardware from different vendors.
- The physical hardware is shared so that *each OS appears to have its own* memory, processor, and other resources.
- Each hypervisor ships with a layer of **hypervisor management software** offering the operational primitives — create VM, delete VM, move VM — that operators actually call.

### Type 1 and Type 2

Hypervisors come in two flavors, mapping onto the hosted-vs-bare-metal distinction from §What virtualization is. Common vendors include **VMware, Xen, and Microsoft Virtual Server**.

**Type 1 — bare-metal / native.** Sits between the hardware and the VMs, running directly on the host's hardware as hardware control. It interacts directly with the virtualized hardware, is independent of the OS, and boots before any OS. Type 1 is what every major desktop-virtualization vendor — **VMware, Microsoft, Citrix** — actually ships.

**Type 2 — embedded / hosted.** A software application running inside a conventional operating system — *above* the host OS, not below the OS and not below the VMs. It can run a different OS on top of another OS; because the hypervisor is a distinct layer, the guest OS sits at the **third level above the hardware**. The drawback is structural: a Type 2 hypervisor cannot boot until the host OS is running, so if the host OS crashes every end-user is affected, and a Type 2 hypervisor is only as secure as the OS underneath it.

| | Type 1 (bare-metal / native) | Type 2 (hosted / embedded) |
|---|---|---|
| Sits | Between hardware and VMs | Above the host OS |
| Boots | Before any OS; independent of host OS | Only after the host OS is running |
| Guest OS level | Second level above hardware | Third level above hardware |
| Risk | Smaller, tightly controlled attack surface | Host OS crash affects all users; security capped at host OS security |
| Production users | VMware, Microsoft, Citrix (and Azure Hyper-V — see [Azure §8]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#8-the-hypervisor)) | Desktop VM products (VirtualBox, VMware Workstation, …) |

### Three features every production hypervisor offers

Once a fleet of hypervisors runs across multiple physical servers connected to shared storage, three features become possible — and they progressively close the gap between "a VM is a thing that runs on one machine" and "a VM is a workload that floats over a pool."

**High Availability (HA).** HA minimizes the impact of downtime by continuously monitoring all VMs in the virtual resource pool (the set of physical servers running the VMs). When a physical server fails, its VMs are **automatically restarted on another server**. Concretely, with three servers A, B, C: if B fails, the VMs B1 and B2 restart on A and C. This works because **VM images live in the shared storage system** that every server attaches to, so any server can pick up a VM's disk and boot it. The limitation: a hardware failure can still cause data loss, because the restart loses whatever was in memory at the time. That gap is what Fault Tolerance closes.

**Fault Tolerance (FT).** FT runs an **identical real-time copy** of the VM on another server: every instruction the primary VM executes also executes on the secondary VM. In the same A/B/C scenario, if server B fails, B1 and B2 keep running on A and C with **no data loss and no downtime**, because the secondary was already executing in lockstep with the primary.

**Live Migration (LM).** LM **moves a running VM between physical hosts while it stays in operation**, typically because the server got too busy or needs maintenance. This is possible because a VM is hardware-(configuration-)independent — it is not tied to one physical server. The benefit is balanced capacity across the fleet, with each VM continuing to get the resources it needs on time. The deeper mechanics — pre-copy vs. post-copy, brownout and blackout, dirty tracking, why all of this is harder for confidential VMs — are covered in [Live Migration]({{ site.baseurl }}/docs/core/live-migration/).

| Feature | What it does | Limitation / note |
|---|---|---|
| HA | Restarts VMs on another server after a physical-server failure (images on shared storage) | Hardware failure can still lose in-memory state |
| FT | Real-time identical VM copy on another server; every instruction mirrored | No data loss, no downtime |
| LM | Moves running VMs between hosts | Used for load balancing and maintenance windows |

### Why hypervisor-based systems are secure (in principle)

A hypervisor that owns the hardware is, in principle, a strong security boundary. Four properties matter:

1. The hypervisor **controls the hardware**, providing a secure infrastructure and preventing unauthorized users from compromising the hardware infrastructure — it acts as a firewall.
2. The hypervisor is implemented **below the guest OS**, so an attack that gets past the guest OS's defenses still has to get past the hypervisor — and the hypervisor is positioned to detect it.
3. It acts as a **layer of abstraction** isolating the virtual environment from the hardware underneath.
4. It controls **all access** between guest OSes and the shared hardware, which simplifies transaction monitoring in a cloud environment.

These are the properties that *should* hold. The next section is about what happens when they don't.

## VMs in the cloud and hypervisor security

A virtual machine means sharing the resources of one physical computer among many "computers" running inside it. A software layer presents each guest with what looks like its own dedicated CPU, memory, disk, and network card, while the real hardware underneath is shared. The cloud runs many virtualized systems on the same hardware to **maximize resource utilization** — that is what makes the cloud economically efficient: one expensive server hosts dozens of tenants. The agility, flexibility, and scalability follow because VMs are just files — cloned in seconds, migrated while running, paused, snapshotted, rolled back.

The flip side is that the hypervisor, because it creates and controls every VM on a host, is the **single most valuable target** in the stack. Compromising one VM exposes one tenant's data; compromising the hypervisor potentially exposes *everyone's*.

### Architecture determines attack surface

The native ("bare-metal") architecture has the smallest surface. The hypervisor runs directly on hardware with no host OS beneath it; there are **no known attacks** on a native-architecture hypervisor, because it is embedded in the hardware — a much smaller, tightly controlled surface. But this requires specially configured hardware, which is why most real-world deployments use the **hosted architecture** instead: the hypervisor runs on top of a conventional host OS, on ordinary hardware. That host OS becomes the weak point, and it sets up the two main attack routes.

### Attack route 1 — through the host OS

The idea here is not to attack the hypervisor directly, but to attack the OS it sits on and then inherit control of everything above it:

1. Exploit vulnerabilities in the host OS the hypervisor runs on.
2. Native virtualization needs special hardware, so most deployments are hosted — a host OS almost always exists to be attacked.
3. Modern OSes carry vulnerabilities, so ordinary OS exploits give the attacker control of the host OS.
4. The hypervisor is just a layer on top of the host OS, so **owning the host OS effectively compromises the hypervisor** — the attacker is now underneath it, and nothing can defend against the layer it depends on.
5. With the hypervisor's administrative privileges, the attacker can do anything to any hosted VM: read memory, copy disks, inject code, snapshot data.

The propagation chain runs **Host OS → Hypervisor → VMs**: one breach at the bottom cascades to every tenant. A useful mental model is that the host OS is the foundation of a building — control the foundation and you control every apartment (VM) resting on it.

### Attack route 2 — through a guest OS (VM escape / jailbreak)

Here the attacker starts inside a VM they legitimately control and breaks out of it:

1. The attacker uses a guest OS to gain illegal access to other VMs or the hypervisor.
2. **VM escape** (also **jailbreak**) means escaping the confinement of your VM into layers otherwise unknown or invisible to it. A VM is supposed to believe it is a standalone computer; an escape shatters that illusion.
3. This is the **most feasible attack** on the hypervisor — because of access. Attackers can only reach a VM remotely, the host OS is invisible to them, but they can rent a VM legitimately. Escaping upward from their own VM is often the only door available.
4. The deeper vulnerability is **shared physical resources**. If the attacker works out how their VM's virtual resources map to physical resources, they can stop attacking the illusion and attack the real hardware directly.
5. Concretely: by modifying virtual memory in a way that exploits the virtual-to-physical mapping, the attacker can reach beyond their VM and affect **all VMs, the hypervisor, and other programs** on the machine.

The mental model: VM isolation is **logical, not physical** — an agreement enforced by software. VM escape proves the agreement is breakable: "your" memory and "your neighbor's" memory were always on the same physical chips.

| | Via host OS | Via guest OS (VM escape) |
|---|---|---|
| Entry point | Exploit the host OS under a hosted hypervisor | A VM the attacker controls (can be rented legitimately) |
| Mechanism | Own the foundation layer → inherit hypervisor privileges | Exploit virtual-to-physical resource mapping (e.g., virtual memory) to break VM confinement |
| Feasibility | Needs host-OS access, normally invisible remotely | **Most feasible** — the VM is the only door attackers can reach |
| Blast radius | Host OS → Hypervisor → all VMs | All VMs, hypervisor, other programs on the machine |
| Doesn't apply to | Native / bare-metal architecture (no host OS; no known attacks) | — |

### Other attacks on virtualization infrastructure

Beyond the two main routes, the infrastructure has several other exposed surfaces. **Virtual library checkout** attacks abuse how VM images are checked out of or stored in a library/repository. **Migration attacks** target the moment a live VM moves between hosts — a VM in transit is exposed and may cross less-trusted networks. **Encryption attacks** aim at the cryptographic protections of the virtualized environment.

The defensive principle that follows: guest OSes need to use the machine's physical peripherals, so the **guest ↔ hypervisor communication channel must be secure**, and the hypervisor's abstraction must be **enforced** — the guest must never reach past its allotted, virtualized view. This is exactly the threat model that confidential VMs (AMD SEV-SNP, Intel TDX) push further by encrypting and integrity-protecting VM memory in hardware, moving the hypervisor *out* of the trusted computing base of the guest. See [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/) for the CC framing, and [Intel TDX]({{ site.baseurl }}/docs/core/tdx/) for one concrete realization.

Once the hypervisor is pushed out of the trust boundary, however, the trusted hypervisor services the guest used to rely on (device emulation, vTPM, interrupt management) are no longer available from outside — and a different layer has to step in to provide them on the guest's behalf. That layer is the **paravisor**: it runs inside the confidential VM at a higher privilege level than the guest OS, providing those services so that unmodified Windows and Linux can run as confidential guests without becoming "fully enlightened" themselves. See [Paravisor]({{ site.baseurl }}/docs/core/paravisor/) for the architecture, OpenHCL specifics, and how it relates to (but does not fit into) the Type 1 / Type 2 taxonomy above.

## Key acronyms

VMM (Virtual Machine Monitor) • PMB / LMB (Physical / Logical Memory Block) • HA (High Availability) • FT (Fault Tolerance) • LM (Live Migration) • VM (Virtual Machine)
