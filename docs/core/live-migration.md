---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Live migration moves a running VM from one host to another while the workload keeps executing. The VM's state — memory, CPU registers, device queues — is copied to the destination; the network is redirected; and the guest OS (and anything talking to it) is none the wiser. Its IPs, metadata, disks, and network settings are all preserved across the move.

The payoff is that cloud providers can patch or upgrade host infrastructure — kernel upgrades, firmware updates, rack repairs — without draining workloads or forcing customer-visible reboots.

## Why migrate?

- **Controlled maintenance windows** instead of emergency ones: move work off a machine before taking it down.
- **Load balancing** for long-lived jobs: shift work off an overloaded server. (Short-lived jobs rarely justify the cost — they'll finish before migration pays off.)
- **Fault tolerance**: move jobs away from hardware that's getting flaky but hasn't failed yet.
- **Energy efficiency**: consolidate loads onto fewer machines to cut power and cooling.
- **Operator-driven placement**: move a VM to a different host or host group — from a shared host to a dedicated one, to rebalance a noisy-neighbor situation, etc. — without restarting the workload.

The data center is the natural home for all of this: many machines under unified control, with the right kind of shared infrastructure to make the moves clean.

## What makes clean migration possible

A running workload refers to a bunch of things — memory, files, network connections, devices. Each piece of that state is handled in one of two ways:

- **Local names** — state physically tied to the host. Memory, CPU registers, locally-attached disks, a USB device bolted to the box. To migrate, this state has to move with the VM.
- **Global names** — state the workload reaches by a name that works from anywhere. A file on network-attached storage is reachable from any host; an IP address can follow the VM via layer-2 networking or NAT.

The distinction matters because of **residual dependencies**: leftover bits of state on the old host that force it to stay up after migration (e.g., the old host still serving a local disk, or forwarding packets for the migrated VM). Every local-named resource has to either move physically or become a residual dependency. Global names eliminate that choice — the state is already reachable from anywhere.

This is why data centers make clean migration practical. They're built around global names (networked storage, managed IP addressing). Combine that with a hypervisor that treats the guest OS as an opaque blob of memory and devices — no need to understand what's running inside, no need for guest cooperation — and you can move a VM with no lingering dependencies on the source.

## Goals and trade-offs

Live migration optimizes several things at once, and they pull against each other:

- **Downtime** — the window where the service is actually unavailable. Should be short enough to be invisible to the workload.
- **Total migration time** — wall-clock from start to finish.
- **Total transferred bytes** — how much data goes over the network between source and destination.
- **Impact on other services** — the migration itself consumes bandwidth and CPU on both hosts. Too aggressive and it starves neighbors; too conservative and total migration time blows out. Throttling the migration (capping its network and CPU) is the standard lever.

No scheme minimizes all four. A short downtime needs a fast final copy, which eats bandwidth. A small total-transferred-bytes number (no re-copying) tends to mean longer downtime. Every live migration algorithm is a particular choice among these trade-offs.

## How it works

The canonical algorithm is **pre-copy**: the VM keeps running on the source while its memory is streamed to the destination, with a brief pause at the end to catch up.

1. **Allocate resources at the destination.** Confirm the target host can receive the VM (enough memory, CPU, etc.), spin up an empty receiving VM, and establish an authenticated channel between source and target.
2. **Iteratively copy memory** (*source brownout*). While the VM keeps running, copy its memory pages to the destination. Since the VM is still writing, any page that gets modified after it's copied has to be re-copied. Iterate: copy everything, then the pages dirtied during that copy, then the ones dirtied during that pass, and so on. Stop when dirty memory drops below a threshold — or when you stop making forward progress, because the VM is dirtying pages as fast as you can send them.
3. **Stop and copy the remainder** (*blackout*). Pause the VM on the source, send the last dirty pages, and wait for the destination to acknowledge. At that point the migration is committed — either side could restart the VM. This is the only window where the service is actually down; for most workloads it's invisible (one classic demo reported 60 ms for a running Quake 3 session — below the threshold a player notices). One subtle effect: wall-clock time keeps advancing while the VM is paused, so the guest clock appears to jump forward on resume. Long enough blackouts need a guest-side daemon to resync.
4. **Redirect the network.** Send a gratuitous ARP — an unsolicited announcement to the local network saying "this IP now lives at this MAC address" — so traffic starts arriving at the new host. A few in-flight packets may be lost, but that happens on normal networks and TCP recovers.
5. **Resume on the destination** (*target brownout*). The VM starts running again. The source may still provide temporary support — forwarding stray packets until the network fabric catches up with the new location, for example.
6. **Tear down the source.** With no residual dependencies, the old host is completely free.

The two **brownout** windows are periods of slightly degraded performance (disk, CPU, memory, and network utilization all dip as cycles and bandwidth go to the migration) while the VM is still running. The **blackout** is the actual pause. From the guest's perspective and from any external service talking to it, nothing changes — the VM's IPs, metadata, and configuration are all preserved.

## Memory-copy strategies

Pre-copy isn't the only way to move memory. The three main strategies differ in the order they transfer pages and when the VM switches hosts.

- **Pre-copy** — the algorithm above. Easy to implement, no fast inter-host network required, and present in most mainstream hypervisors (Xen, KVM, VMware ESX). Convergence depends on the copy rate outpacing the VM's memory write rate; if the VM dirties pages faster than they can be sent, the iterations never shrink.
- **Post-copy** — reverses the order. Suspend the source, copy a minimal slice of execution state (CPU registers, critical device state) to the destination, immediately transfer execution, then fetch pages on demand. Any access to an un-transferred page triggers a fault that's redirected back to the source over the network. Each page is sent at most once (total transferred bytes is minimized), but the destination is dependent on the source during the copy — page faults add latency, and a network hiccup can stall the VM.
- **Hybrid-copy** — run some pre-copy iterations first, then switch to post-copy. The warm-up reduces the chance of page faults after execution transfers, trading off total migration time against worst-case latency.

## Variants

Live migration schemes also differ in how much the guest OS cooperates:

- **Managed migration** — the VMM does everything from outside and the guest OS is oblivious.
- **Managed with paravirtualization** — same, but the guest pitches in on specific optimizations: *stunning* (briefly freezing) processes that are dirtying memory too fast to keep up, or ballooning unused pages out so they don't have to be copied. ("Paravirtualization" = guest OS is modified to cooperate with the VMM, rather than being kept unaware.)
- **Self migration** — the OS itself drives the migration. Harder: snapshotting a running OS using that same OS is like taking a photo of yourself taking the photo.

## What's hard to migrate live

The whole approach works because VM state — memory, registers, a few device queues — can be captured and reconstituted elsewhere. State that doesn't fit that model is where live migration breaks down:

- **Passthrough devices** — when a VM has direct access to a physical device (GPU, FPGA, SR-IOV NIC), the device's internal state lives in hardware the hypervisor doesn't control. There's no clean way to snapshot "what the GPU was in the middle of" and replay it on a different physical chip.
- **Hardware-bound identifiers** — anything tied to a specific physical machine (serial numbers, TPM state bound to the host's endorsement key, hardware-rooted secrets) has to be re-established on the destination or the migration can't preserve it.
- **Tight latency budgets** — workloads that can't absorb even a brownout, let alone the blackout, aren't good candidates. Terminate-and-restart on a planned window may be the cleaner answer.

Platforms handle these by refusing live migration for the affected VM class, giving advance notice so the workload can drain or checkpoint, or offering a paravirtualized equivalent whose state is capturable.

## Operational concerns

An individual migration is only half the story. At fleet scale there's a separate **scheduling layer** — cluster management software that decides when and which VMs migrate. It watches for trigger events (hardware-failure signals, planned maintenance, software/firmware rollouts, explicit operator requests), schedules migrations against policies (caps on concurrent migrations per customer, capacity limits on target hosts, fairness across tenants), and paces the fleet so a failing rack doesn't stampede the rest of the data center.

Not every VM runs in "always migrate" mode. Platforms typically expose a per-VM **availability policy**:

- **Maintenance behavior** — live-migrate (default) or terminate on an event. Termination suits workloads that need constant peak performance (no brownout acceptable) or apps that already handle instance failures.
- **Restart behavior** — whether a terminated VM comes back automatically or waits for manual restart.

The choice is whether the app prefers degraded-but-continuous (brownout) or clean-but-offline (terminate-and-restart).

Live migration is complex enough to demand the same rigor as any other critical piece of infrastructure. **Fault injection** at each interesting point in the algorithm (network drops mid-copy, destination crash during blackout, source crash after commit) verifies the system either completes the migration or cleanly aborts. A **simulated maintenance event** API lets workload owners trigger a migration on demand against their own VMs, so they can confirm their availability policies behave the way they expect.

## Hyper-V

_Notes to come._

## TDX Live Migration

_Notes to come._
