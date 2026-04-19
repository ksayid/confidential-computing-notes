---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Migration — moving a running workload from one physical machine to another — is useful for several reasons. It enables **load balancing** for long-lived jobs, where you can shift work off an overloaded server (a natural question: why not short-lived jobs too? Probably because they'll finish before migration pays off). It gives you **controlled maintenance windows** instead of emergency ones, since you can move work off a machine before taking it down. It supports **fault tolerance** by letting you move jobs away from hardware that's getting flaky but hasn't failed yet. It helps with **energy efficiency** — you can consolidate loads onto fewer machines to reduce cooling (A/C) needs. And it supports **operator-driven placement** — moving a VM to a different host or host group (shared to dedicated, rebalancing a noisy-neighbor situation) without restarting the workload. The data center is the natural environment for all of this, since you have many machines under unified control.

## The spectrum of migration approaches

Live migration sits at one end of a spectrum of techniques for moving a workload to new physical resources, ordered from most disruptive to least:

1. **Hard power down.** Cut power to the VM, move the virtual disk to a new host, cold boot there. Any in-memory state is lost; applications that weren't built to survive a sudden power loss lose whatever they were working on.
2. **Soft power down.** Send a shutdown request to the guest OS, which asks applications to save state and exit cleanly. Depends on the guest OS supporting clean shutdown and on each application cooperating.
3. **Hibernation.** The guest OS freezes all processes, saves device and memory state to disk, and powers down. The disk can then be moved and resumed elsewhere. Preserves more state than a power down, but depends on the guest OS supporting hibernation and on the guest being stable enough to do it.
4. **Quick migration (VM save and restore).** The host pauses the VM, gathers its full state (memory and device state) from outside the guest, transfers it to a new host, and resumes there. Because the host drives this rather than the guest, it doesn't depend on guest cooperation — more reliable than hibernation. The user sees noticeable downtime, though, since a multi-GB memory image has to transfer while the VM is paused.
5. **Live migration.** Same idea as quick migration, but most of the state transfers while the VM is still running. Only the small remainder transfers during a pause, minimizing downtime.

Each step reduces how much the customer notices that their workload changed machines. Taken together, this progression moves virtualization toward the ideal of **assignment independence** — the physical machine becomes irrelevant to the workload.

## Two basic strategies for moving state

When you migrate a workload, you have to deal with all the things it refers to — memory, files, network connections, devices. How you handle each piece of state falls into one of two approaches, and which one is available shapes what kinds of migration are even possible.

The first is **local names**: physically move the state to the new machine. This covers things like local memory contents, CPU registers, and local disk. It's the only option when the resource is tied to the original machine, but it doesn't work for some physical devices — a tape drive bolted to the source host can't come with you.

The second is **global names**: the resource has a name that works from anywhere, so you just keep using it from the new location. Network-attached storage is the canonical example — the files live on a separate storage system, and both the old and new machine can reach them by the same name. For network identity, techniques like Network Address Translation (NAT) or layer 2 (Ethernet-level) addressing let an IP address effectively follow the workload to its new host.

This distinction matters because of **residual dependencies** — leftover bits of state on the old machine that force it to stay up after migration (e.g., the old host still serving a local disk or forwarding network traffic for the migrated workload). Every piece of state handled by local names has to either move physically or become a residual dependency. Global names eliminate the choice: the state is already reachable from anywhere, so nothing has to move and nothing has to stay behind. Data centers are designed around global names (networked storage, managed IP addressing), which is what makes clean migration possible there — and it's why the VMM approach below works well in data centers specifically.

## VMM migration

VMM migration (Virtual Machine Monitor migration — moving an entire virtual machine) moves the whole OS as a single unit. You don't need to understand what's running inside or what state belongs to what; it's all just memory and devices from the VMM's perspective. This means you can migrate applications whose source code you don't have, and ones whose owners don't trust you enough to cooperate with a migration. Combined with the global names a data center provides, VMM migration can avoid residual dependencies entirely — the memory moves with the VM, and everything else (storage, network identity) is already reachable from the new host.

Non-live VMM migration (where the VM is suspended, moved, and resumed) is also useful outside the data center. You can migrate your work environment from office to home and back by putting the suspended VM on a USB key or sending it over the network — the "Collective" project called this "Internet suspend and resume."

## Goals of live migration

Live migration optimizes four things at once, and they pull against each other:

- **Downtime** — the window where the service is unavailable. Should be short enough to be invisible to the workload.
- **Total migration time** — wall-clock from start to finish.
- **Total transferred bytes** — how much data goes over the network between source and destination.
- **Impact on other services** — the migration consumes bandwidth and CPU on both hosts. Too aggressive and it starves neighbors; too conservative and total migration time blows out. **Throttling** (capping the migration's network and CPU usage) is the standard lever.

No scheme minimizes all four. A short downtime needs a fast final copy, which eats bandwidth. Minimizing total transferred bytes (no re-copying) tends to mean a longer downtime. Every live migration algorithm is a particular choice among these trade-offs.

The downtime target is more concrete than "users don't notice." Network protocols like TCP have timeouts on connections to external endpoints, and if the VM is unreachable longer than those timeouts, live connections break. A common target is keeping total pause time under **750 ms**, which stays below most common stack timeouts. This is why sub-second downtime matters — it's about preserving existing network connections, not just perceived responsiveness. (The 60 ms figure reported for a running Quake 3 game on early live migration work is well under this bar.)

## How live migration works

Live migration proceeds in two named phases. **Brownout** is the period where the VM keeps running while memory is iteratively copied — the workload experiences some performance impact but stays live. **Blackout** is the brief paused window at the end where the final dirty state is copied and the VM resumes on the target. The goal is to push as much of the work as possible into brownout so blackout stays short.

1. **Allocate resources at the destination.** Before starting, make sure the target host can actually receive the VM (enough memory, CPU, etc.).
2. **Validate compatibility.** Confirm the target is actually capable of resuming this VM — driver versions, firmware versions, and other ambient system state all need to be compatible. You want to catch incompatibility here, before committing, not after the VM has gone dark.
3. **Iteratively copy memory pages (brownout).** While the VM keeps running on the source, start copying its memory to the destination. The catch: the VM is still writing to memory, so any page that gets modified after you copy it has to be copied again ("dirtied" pages). This requires **dirty bit tracking** — hardware or software machinery that records which pages have been written, so each iteration only has to copy the pages that changed since the last pass. You iterate: copy everything, then copy pages dirtied during that copy, then pages dirtied during that pass, and so on. You stop either when only a small amount of dirty memory remains or when you're not making forward progress (the VM is dirtying pages as fast as you can copy them). Convergence isn't guaranteed — it depends entirely on the workload. Some VMs dirty memory slowly enough that iterations shrink to almost nothing; others write fast enough that you just have to cut over without convergence. You can dedicate more bandwidth to later iterations to shrink the window in which new dirtying can happen — though on the source side this is a balancing act, since migration traffic competes with the VM's own work. On the target side the tradeoff is easier: the VM isn't running there yet, so the full bandwidth budget is available for receiving.

    User-facing traffic keeps flowing during brownout, but **control-plane operations on the VM are typically blocked** — configuration changes, resize operations, and other admin actions are paused so the VM's definition doesn't shift underneath the migration.

    There's a design decision around **when** to enable dirty tracking. If tracking has a meaningful performance cost, you only turn it on at the start of brownout — accepting that the first iteration has to copy the entire memory image since nothing was tracked before. If tracking is cheap enough to run continuously, you enable it at VM startup; then the first brownout iteration already knows which pages are dirty and can skip the clean ones. For workloads that dirty memory slowly (e.g., CPU-heavy VMs that barely touch most of RAM), that's a big win.
4. **Stop and copy the remainder (blackout).** Pause the VM on the source and copy the last bit of dirty state. This is the only window where the service is actually down. At the end of this step, the source and destination are identical — either could be restarted. Once the destination acknowledges the copy, the migration is committed (in the database transaction sense — it's officially done and can't be half-finished). One subtle effect: wall-clock time keeps advancing while the VM is paused, so the guest clock appears to jump forward on resume. Long enough blackouts need a guest-side daemon to resync.
5. **Redirect the network.** Send a "gratuitous ARP" packet — essentially an unsolicited announcement to the local network saying "this IP address now lives at this new MAC address" — so traffic starts arriving at the new host. A few packets in flight may be lost, but that kind of packet loss happens on normal networks anyway, and TCP will retransmit and recover.
6. **Resume on the new host (target brownout).** The VM starts running again on the destination. The source may still provide temporary support — forwarding stray packets until the network fabric catches up with the new location.
7. **Delete the source.** Tear down the original VM on the source host. No residual dependencies — the old host is completely free.

The two numbers to watch when a migration runs long are **blackout duration** and **dirty-page count**. High dirty-page counts point to workload pressure — the VM is writing memory fast enough that brownout iterations can't shrink. Long blackouts with modest dirty-page counts point to network pressure — the final copy is slow because bandwidth between source and target is the bottleneck, not the amount of data.

## Memory-copy strategies

The algorithm above is called **pre-copy** — transfer memory while the VM keeps running, then pause briefly to catch up. It isn't the only option. There are three main strategies, each making a different choice among the trade-offs:

- **Pre-copy.** The canonical algorithm: iteratively copy memory from source to destination while the VM runs on the source, then pause for a final catch-up. Easy to implement, no fast inter-host network required, present in most mainstream hypervisors (Xen, KVM, VMware ESX). Convergence depends on the copy rate outpacing the VM's memory write rate; if the VM dirties pages faster than they can be sent, the iterations never shrink and you have to cut over without convergence.
- **Post-copy.** Reverses the order. Suspend the source, copy a minimal slice of execution state (CPU registers, critical device state) to the destination, immediately transfer execution, then fetch pages on demand. Any access to an un-transferred page triggers a fault that's redirected back to the source over the network. Each page is sent at most once, so **total transferred bytes is minimized** — but the destination depends on the source throughout the copy, page faults add latency, and a network hiccup can stall the VM. (TDX live migration uses this model for its final memory phase — see below.)
- **Hybrid.** Run some pre-copy iterations first, then cut over to post-copy. The pre-copy warm-up reduces the chance of page faults after execution transfers, trading total migration time against worst-case latency.

## Variants

There are a few flavors of live migration, orthogonal to the memory-copy strategy above. **Managed migration** moves the OS without its cooperation — the VMM does everything from outside, and the guest OS doesn't even know it's happening. **Managed migration with paravirtualization** is the same thing but with the guest OS pitching in on specific optimizations: for instance, *stunning* (briefly freezing) rogue processes that are dirtying memory too fast to keep up with, or pushing unused memory pages out of the VM so they don't need to be copied at all. ("Paravirtualization" means the guest OS is modified to cooperate with the VMM, rather than being unaware of it.) **Self migration** goes further: the OS itself drives the migration. This is harder because you're trying to snapshot a running OS using that same OS — it's like trying to take a photo of yourself taking the photo.

## What's hard to migrate live

The approach works because VM state — memory, registers, a few device queues — can be captured and reconstituted elsewhere. State that doesn't fit that model is where live migration breaks down:

- **Passthrough devices.** When a VM has direct access to a physical device (GPU, FPGA, SR-IOV NIC), the device's internal state lives in hardware the hypervisor doesn't control. There's no clean way to snapshot "what the GPU was in the middle of" and replay it on a different physical chip.
- **Hardware-bound identifiers.** Anything tied to a specific physical machine — serial numbers, TPM state bound to the host's endorsement key, hardware-rooted secrets — has to be re-established on the destination or the migration can't preserve it.
- **Tight latency budgets.** Workloads that can't absorb even a brownout, let alone the blackout, aren't good candidates. Terminate-and-restart on a planned window may be the cleaner answer.

Platforms handle these by refusing live migration for the affected VM class, giving advance notice so the workload can drain or checkpoint, or offering a paravirtualized equivalent whose state is capturable.

## Operational concerns

An individual migration is only half the story. At fleet scale there's a separate scheduling layer — cluster management software that decides when and which VMs migrate. It watches for trigger events (hardware-failure signals, planned maintenance, firmware rollouts, explicit operator requests), schedules migrations against policies (caps on concurrent migrations per customer, capacity limits on target hosts, fairness across tenants), and paces the fleet so a failing rack doesn't stampede the rest of the data center.

Not every VM runs in "always migrate" mode. Platforms typically expose a per-VM **availability policy**:

- **Maintenance behavior** — live-migrate (default) or terminate on an event. Termination suits workloads that need constant peak performance (no brownout acceptable) or apps that already handle instance failures.
- **Restart behavior** — whether a terminated VM comes back automatically or waits for manual restart.

The choice is whether the app prefers degraded-but-continuous (brownout) or clean-but-offline (terminate-and-restart).

Live migration demands the same rigor as any other critical infrastructure. **Fault injection** at each interesting point in the algorithm (network drops mid-copy, destination crash during blackout, source crash after commit) verifies the system either completes the migration or cleanly aborts. A **simulated maintenance event API** lets workload owners trigger a migration on demand against their own VMs, so they can confirm their availability policies behave the way they expect.

## Hyper-V

_Notes to come._

## TDX Live Migration

Intel TDX v1.5 adds live migration on top of the TDX v1.0 confidential-VM machinery described in [Intel TDX](tdx.md). The same brownout/blackout shape from the generic case still applies — what changes is who can see the state in transit.

### Why it's harder under TDX

In legacy VMM migration, the host VMM is trusted and has full access to VM memory and state, so source and destination VMMs handle verification, authentication, and transport directly. Under TDX, the host VMM is **untrusted** and cannot read TD memory or state at all. Every byte of migration data has to flow through the TDX Module, encrypted and integrity-protected. This forces two additional components with no legacy equivalent: a **Migration TD (migTD)** that brokers trust between platforms, and a set of **TDX Module migration APIs** that package and unpackage state.

### Roles

| Role | Description |
|---|---|
| **Target TD (tgtTD)** | The TD being migrated. It is *unaware* migration is happening and performs no special actions. |
| **Source TD (srcTD)** | The tgtTD while running on the source platform. |
| **Destination TD (dstTD)** | A template TD constructed on the destination platform to receive migration data. |
| **Migration TD (migTD)** | A special TD created on *both* platforms to verify attestation and exchange cryptographic keys. Bound to the tgtTD before its measurements are finalized. |
| **Source/Destination VMM** | The host VMMs on each platform. They coordinate the transfer but are untrusted and cannot see TD data. |

Although migration is designed for cross-platform relocation, the operations can technically take place on a single platform.

### Bundles, session keys, and attestation

All TD state transferred during migration is packaged into **migration bundles** — units of private memory or non-memory state that are both encrypted and integrity-protected under AES-GCM-256 using a **Migration Session Key (MSK)**. The TDX Module generates the MSK; the migTDs on each platform exchange MSKs with each other, and the destination migTD installs the key into the dstTD.

Before migration proceeds, the migTDs examine each other's TDX Module capabilities and attestation evidence and check them against a **migration policy** that specifies acceptable TD attributes, allowed SVNs, and supported migration protocol versions. This is what extends TDX's confidentiality and integrity guarantees across platforms: the dstTD only accepts state that comes from a TDX Module the migTD has vouched for.

### Migration Stream Contexts

Some migration steps — decrypting bundles, importing metadata — are time-consuming. The TDX Module checks for pending hardware interrupts during these operations and, if one arrives, saves progress to a **Migration Stream Context (MigSC)**, returns control to the VMM, and later resumes from where it left off — verifying that no conflicting operations occurred in the interim. This keeps the untrusted VMM responsive without giving it a window to tamper with in-flight state.

### Phases

The migration walks the dstTD through a sequence of operation states, paralleling the normal build sequence but sourced from bundles rather than from the VMM directly.

1. **Setup.** The destination VMM creates a template dstTD (allocate → configure encryption → add control structures, the same common setup as a normal build). A migTD is created on both platforms and bound to the target TD. Migration Stream Contexts are created, and the migTDs exchange Migration Session Keys.
2. **Immutable state import.** The dstTD's immutable state — the equivalent of what the normal build's initialization step sets up — is imported from encrypted bundles. Interruption and resumption via MigSCs is supported. On completion, the TD enters the **Memory Import** operation state.
3. **In-order memory import (brownout).** The srcTD *continues running*. Memory pages are exported from the source, encrypted into bundles, and imported into the dstTD where they are decrypted and verified (attributes and location must match between source and destination). Order is critical: if the same page is exported multiple times because the srcTD modified it, imports must happen oldest-first, newest-last so the final state is correct.
4. **Blackout begins.** The Intel TDX-imposed blackout starts: the srcTD is *stopped*. Mutable TD-scope state (registers, control structures, other runtime state that changes while the TD runs) can only be exported during this blackout. That state is imported into the dstTD, which enters the **State Import** operation state.
5. **VP state and dirty-page migration.** With the srcTD stopped, per-VP state is imported and any pages dirtied during brownout are re-migrated. Memory import continues in parallel. **Epoch tracking** manages the transition out of this phase: the source creates migration epoch tokens and the destination consumes them; when the final (start) token arrives, the TD enters **Post-Import**.
6. **Out-of-order memory import (post-copy).** The srcTD is no longer runnable and page ordering no longer matters, so memory can be imported in any order. Once the import is committed, **the blackout ends and the dstTD can start running — even before all memory pages have arrived.** If the dstTD accesses a page that hasn't transferred yet, an EPT violation fires and the destination VMM can prioritize that page. The rest stream in in the background. When all memory is transferred and the migration is finalized, the TD returns to the **Runnable** state.

Shared (non-private) memory is handled separately through traditional VMM workflows, outside the TDX Module.

### Operation-state flow

```
              ┌─────────────────────────────────────────┐
              │          COMMON SETUP                   │
              │  Allocate TD → Configure Encryption     │
              │  → Add Control Structures               │
              └──────────────┬──────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
         BUILD PATH                    IMPORT PATH
              │                             │
     Initialize TD state          Bind migTD, create
              │                   streams, exchange keys
              ▼                             │
        INITIALIZED               Import immutable state
              │                             │
     Create VPs, populate                   ▼
     memory, measure                 MEMORY IMPORT
              │                   (srcTD still running,
     Finalize measurements         in-order page import)
              │                             │
              │                   Stop srcTD (blackout),
              │                   import mutable TD state
              │                             │
              │                             ▼
              │                      STATE IMPORT
              │                   (VP state, dirty pages,
              │                    epoch tracking)
              │                             │
              │                             ▼
              │                      POST-IMPORT
              │                   (out-of-order pages,
              │                    no ordering needed)
              │                             │
              │                   Commit → blackout ends
              │                   → dstTD can run
              │                             │
              │                             ▼
              │                      LIVE IMPORT
              │                   (post-copy, dstTD running,
              │                    remaining pages transfer)
              │                             │
              │                   Finalize migration
              │                             │
              ▼                             ▼
                        RUNNABLE
```
