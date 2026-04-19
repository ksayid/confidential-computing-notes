---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Live migration lets a cloud provider patch or upgrade host infrastructure (e.g., host kernel upgrades) without disrupting the VMs running on it. Instead of draining and rebooting a host, VMs are relocated to other hosts while still running.

## Why migrate?

Migration — moving a running workload from one physical machine to another — is useful for several reasons:

- **Load balancing** for long-lived jobs: shift work off an overloaded server. (A natural question: why not short-lived jobs too? Probably because they'll finish before migration pays off.)
- **Controlled maintenance windows** instead of emergency ones: move work off a machine before taking it down.
- **Fault tolerance**: move jobs away from hardware that's getting flaky but hasn't failed yet.
- **Energy efficiency**: consolidate loads onto fewer machines to reduce cooling (A/C) needs.

The data center is the natural environment for all of this, since you have many machines under unified control.

## Two basic strategies for moving state

When you migrate a workload, you have to deal with all the things it refers to — memory, files, network connections, devices. How you handle each piece of state falls into one of two approaches, and which one is available shapes what kinds of migration are even possible.

**Local names**: physically move the state to the new machine. This covers things like local memory contents, CPU registers, and local disk. It's the only option when the resource is tied to the original machine, but it doesn't work for some physical devices — a tape drive bolted to the source host can't come with you.

**Global names**: the resource has a name that works from anywhere, so you just keep using it from the new location. Network-attached storage is the canonical example — the files live on a separate storage system, and both the old and new machine can reach them by the same name. For network identity, techniques like Network Address Translation (NAT) or layer 2 (Ethernet-level) addressing let an IP address effectively follow the workload to its new host.

This distinction matters because of **residual dependencies** — leftover bits of state on the old machine that force it to stay up after migration (e.g., the old host still serving a local disk or forwarding network traffic for the migrated workload). Every piece of state handled by local names has to either move physically or become a residual dependency. Global names eliminate the choice: the state is already reachable from anywhere, so nothing has to move and nothing has to stay behind. Data centers are designed around global names (networked storage, managed IP addressing), which is what makes clean migration possible there — and it's why the VMM approach below works well in data centers specifically.

## VMM migration

VMM migration (Virtual Machine Monitor migration — moving an entire virtual machine) moves the whole OS as a single unit. You don't need to understand what's running inside or what state belongs to what; it's all just memory and devices from the VMM's perspective. This means you can migrate applications whose source code you don't have, and ones whose owners don't trust you enough to cooperate with a migration. Combined with the global names a data center provides, VMM migration can avoid residual dependencies entirely — the memory moves with the VM, and everything else (storage, network identity) is already reachable from the new host.

Non-live VMM migration (where the VM is suspended, moved, and resumed) is also useful outside the data center. You can migrate your work environment from office to home and back by putting the suspended VM on a USB key or sending it over the network — the "Collective" project called this "Internet suspend and resume."

## Classifications

Live migration schemes can be classified along two axes.

**By what has to move (storage topology):**
- **Live memory migration** — the source and destination share storage, so only the VM's memory and device state need to move.
- **Live storage migration** — the VM's disk image also has to be transferred.
- **Live unified migration** — both memory and storage move together (required when there's no shared storage between the hosts).

**By network scope:**
- **Over LAN** — source and destination are in the same data center, with high bandwidth and low latency between them.
- **Over WAN** — migration across a wide-area link. Much more challenging because of limited bandwidth and high latency; the same algorithms that work cleanly on a LAN can stall or take much longer.

## Goals of live migration

Live migration — moving the VM while it keeps running — has several goals in tension with each other:

- **Minimize downtime** (the window where the service is unavailable).
- **Keep total migration time manageable.**
- **Minimize total transferred bytes** — the total amount of data pushed over the network from source to destination.
- **Limit the impact** on both the VM being moved and the local network during the move.

A good live migration scheme seeks a reasonable trade-off among these — they can't all be minimized at once.

### Impact on other services

Any live migration will put pressure on shared resources. In live unified migration over LAN, for example, memory and storage are both transferred in a succession of iterations, and that traffic can easily consume the bandwidth between source and destination — starving other active services. Some service degradation is unavoidable, but it can be alleviated by throttling the migration: cap the network bandwidth and CPU resources the migration process is allowed to consume.

## How live migration works

1. **Allocate resources at the destination.** Before starting, make sure the target host can actually receive the VM (enough memory, CPU, etc.).
2. **Iteratively copy memory pages** (*pre-migration brownout*). While the VM keeps running on the source, start copying its memory to the destination. The catch: the VM is still writing to memory, so any page that gets modified after you copy it has to be copied again ("dirtied" pages). You iterate — copy everything, then copy the pages that got dirtied during that copy, then the ones dirtied during that pass, and so on. You stop either when only a small amount of dirty memory remains or when you're not making forward progress (the VM is dirtying pages as fast as you can copy them). You can dedicate more bandwidth to later iterations to shrink the window in which new dirtying can happen. The decision to exit this phase is driven by an algorithm that balances remaining dirty bytes against the VM's current rate of change.
3. **Stop and copy the remainder** (*blackout*). Pause the VM on the source and copy the last bit of dirty state. This is the only window where the service is actually down. At the end of this step, the source and destination are identical — either could be restarted. Once the destination acknowledges the copy, the migration is committed (in the database transaction sense — it's officially done and can't be half-finished).
4. **Redirect the network.** Send a "gratuitous ARP" packet — essentially an unsolicited announcement to the local network saying "this IP address now lives at this new MAC address" — so traffic starts arriving at the new host. A few packets in flight may be lost, but that kind of packet loss happens on normal networks anyway, and TCP will retransmit and recover.
5. **Resume on the new host** (*post-migration brownout*). The VM starts running again on the destination. The source VM may still be hanging around providing supporting functionality — for example, forwarding packets to and from the target until the network fabric catches up with the VM's new location.
6. **Delete the source.** Tear down the original VM on the source host. No residual dependencies — the old host is completely free.

The three windows are often called **pre-migration brownout**, **blackout**, and **post-migration brownout**. The brownouts are periods of slightly degraded performance with the VM still running; the blackout is the brief pause where the service is actually down. For most workloads the blackout is invisible. The payoff: when a major vulnerability needs patching, hosts can be updated without customer-visible reboots — versus the alternative of forcing customers to schedule downtime.

### VM attributes are preserved

Migration doesn't change the VM's identity or configuration. Metadata, internal and external IP addresses, network settings, attached disks — all unchanged. From the guest's perspective (and from any external service talking to it) nothing happened.

### The scheduling layer

An individual migration is only half the story. Something upstream has to decide **when** a VM migrates and **which** VMs migrate **when**. That's a separate cluster-management layer that:

- Watches for trigger events — hardware-failure signals, planned maintenance windows, software-update rollouts.
- Schedules migrations against policies — caps on how many VMs for a single customer can migrate at once, capacity-utilization limits on target hosts, fairness across tenants, etc.
- Notifies the VM (or its operator) that a migration is about to happen, with a waiting period before the actual move.
- Picks a target host, spins up an empty receiving VM there, and establishes an authenticated channel between source and target before any state is copied.

This is where fleet-wide concerns live: you don't want every VM on a failing rack to migrate simultaneously and stampede the rest of the data center.

## Memory migration strategies

The "iteratively copy memory pages" step above describes one specific approach — **pre-copy**. There are three main strategies for moving memory state, differing in the order they transfer pages and when the VM switches hosts.

### Pre-copy

Pre-copy is easy to design and doesn't require a fast network between hosts, so it's implemented in most mainstream hypervisors (Xen, KVM, VMware ESX). It has two phases:

1. **Warm-up phase**: Copy the whole memory from source to destination while the VM keeps running on the source. Any pages dirtied during that copy are re-copied in a second iteration, and so on. This repeats until the remaining dirty memory drops below a pre-defined threshold. Convergence depends on the copy rate staying faster than the VM's memory write rate — if the VM dirties pages faster than you can send them, the iterations never shrink.
2. **Stop-and-copy phase**: Suspend the VM on the source, copy the last bit of dirty memory, then resume the VM on the destination.

### Post-copy

Post-copy reverses the order. It starts by suspending the source VM, copying a minimal subset of execution state (CPU registers, etc.) to the destination, and immediately transferring execution control — the VM resumes on the destination almost right away.

Any access to un-transferred memory triggers a page fault, which is trapped at the destination and redirected back to the source over the network to fetch that page. In parallel, a background thread copies over the remaining memory while the VM runs on the destination.

This minimizes total transferred bytes (each page is sent at most once) but makes the destination VM dependent on the source during the copy — page faults add latency, and if the network between them hiccups, the VM is stuck.

### Hybrid-copy

Hybrid-copy combines the two, trading off between minimizing downtime and minimizing total migration time. It runs some pre-copy warm-up iterations first, then switches to post-copy. The initial warm-up reduces the chance of page faults on the destination, because a large portion of memory is already there by the time execution transfers.

## Variants

- **Managed migration** moves the OS without its cooperation — the VMM does everything from outside, and the guest OS doesn't even know it's happening.
- **Managed migration with paravirtualization** is the same thing but with the guest OS pitching in on specific optimizations: for instance, *stunning* (briefly freezing) rogue processes that are dirtying memory too fast to keep up with, or pushing unused memory pages out of the VM so they don't need to be copied at all. ("Paravirtualization" means the guest OS is modified to cooperate with the VMM, rather than being unaware of it.)
- **Self migration** goes further: the OS itself drives the migration. This is harder because you're trying to snapshot a running OS using that same OS — it's like trying to take a photo of yourself taking the photo.

## Availability policies

Live migration isn't the only option for handling a maintenance event. A cloud platform typically lets the VM owner pick a per-VM **maintenance behavior**:

- **Live-migrate** — the default for most workloads. The VM keeps running through the event.
- **Terminate** — stop the VM when a maintenance event fires. Suitable for workloads that need constant, maximum performance (no brownout acceptable), or for applications that are already architected to handle instance failures.

A separate **restart behavior** controls whether a terminated VM automatically comes back up, or stays down until an operator restarts it. During a brownout, workloads may see a short dip in performance — terminate-and-restart avoids the dip at the cost of a hard interruption, so the choice depends on whether the app prefers degraded-but-continuous or clean-but-offline.

## What can't live-migrate

Not every workload can move while running. Two common cases:

- **GPU-attached VMs** — the GPU state is hard to snapshot and transfer live, so these VMs are typically set to terminate. The platform instead gives advance notice (on the order of tens of minutes) before the maintenance event so the workload can checkpoint or drain gracefully.
- **Preemptible / spot instances** — these trade availability for price and are always set to terminate on any disruption. Live migration is simply not offered.

Some locally-attached resources *do* migrate — e.g., local SSDs can be moved along with the VM to the new host — but anything with hardware state that can't be captured in memory is usually in the "must terminate" bucket.

## Testing live migration

Live migration is a critical piece of production infrastructure, and it's too complex to trust without continuous testing. Two practices are worth calling out:

- **Fault injection** — deliberately trigger failures at each interesting point in the migration algorithm (network drops mid-copy, destination crash during blackout, source crash after commit, etc.) and verify the system either completes the migration or cleanly aborts. Both active failures (something goes wrong) and passive ones (something just doesn't respond) should be covered.
- **Simulated maintenance events** — expose a knob that lets operators trigger a migration on-demand against their own VMs. This lets them validate their availability policies: does the app actually survive a live-migrate brownout? Do preemptible workloads shut down cleanly? Does the terminate-and-restart path come back to a healthy state?

## Results

The approach delivers on all three goals. Downtimes are very short — 60ms for a running Quake 3 game, which is below the threshold a player would notice. Impact on the running service and the local network stays limited and reasonable. Total migration time is on the order of minutes. And once migration finishes, the source host is completely free of the workload, with no lingering dependencies.

## Hyper-V

_Notes to come._

## TDX Live Migration

_Notes to come._
