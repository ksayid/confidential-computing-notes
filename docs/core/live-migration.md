---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Migration — moving a running workload from one physical machine to another — is useful for several reasons. It enables **load balancing** for long-lived jobs, where you can shift work off an overloaded server (a natural question: why not short-lived jobs too? Probably because they'll finish before migration pays off). It gives you **controlled maintenance windows** instead of emergency ones, since you can move work off a machine before taking it down. It supports **fault tolerance** by letting you move jobs away from hardware that's getting flaky but hasn't failed yet. And it helps with **energy efficiency** — you can consolidate loads onto fewer machines to reduce cooling (A/C) needs. The data center is the natural environment for all of this, since you have many machines under unified control.

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

Live migration has three goals in tension with each other: minimize downtime (the window where the service is unavailable), keep total migration time manageable, and limit the impact on both the VM being moved and the local network during the move.

The downtime target is more concrete than "users don't notice." Network protocols like TCP have timeouts on connections to external endpoints, and if the VM is unreachable longer than those timeouts, live connections break. A common target is keeping total pause time under **750 ms**, which stays below most common stack timeouts. This is why sub-second downtime matters — it's about preserving existing network connections, not just perceived responsiveness. (The 60 ms figure reported for a running Quake 3 game on early live migration work is well under this bar.)

## How live migration works

Live migration proceeds in two named phases. **Brownout** is the period where the VM keeps running while memory is iteratively copied — the workload experiences some performance impact but stays live. **Blackout** is the brief paused window at the end where the final dirty state is copied and the VM resumes on the target. The goal is to push as much of the work as possible into brownout so blackout stays short.

1. **Allocate resources at the destination.** Before starting, make sure the target host can actually receive the VM (enough memory, CPU, etc.).
2. **Validate compatibility.** Confirm the target is actually capable of resuming this VM — driver versions, firmware versions, and other ambient system state all need to be compatible. You want to catch incompatibility here, before committing, not after the VM has gone dark.
3. **Iteratively copy memory pages (brownout).** While the VM keeps running on the source, start copying its memory to the destination. The catch: the VM is still writing to memory, so any page that gets modified after you copy it has to be copied again ("dirtied" pages). This requires **dirty bit tracking** — hardware or software machinery that records which pages have been written, so each iteration only has to copy the pages that changed since the last pass. You iterate: copy everything, then copy pages dirtied during that copy, then pages dirtied during that pass, and so on. You stop either when only a small amount of dirty memory remains or when you're not making forward progress (the VM is dirtying pages as fast as you can copy them). Convergence isn't guaranteed — it depends entirely on the workload. Some VMs dirty memory slowly enough that iterations shrink to almost nothing; others write fast enough that you just have to cut over without convergence. You can dedicate more bandwidth to later iterations to shrink the window in which new dirtying can happen — though on the source side this is a balancing act, since migration traffic competes with the VM's own work. On the target side the tradeoff is easier: the VM isn't running there yet, so the full bandwidth budget is available for receiving.

    User-facing traffic keeps flowing during brownout, but **control-plane operations on the VM are typically blocked** — configuration changes, resize operations, and other admin actions are paused so the VM's definition doesn't shift underneath the migration.

    There's a design decision around **when** to enable dirty tracking. If tracking has a meaningful performance cost, you only turn it on at the start of brownout — accepting that the first iteration has to copy the entire memory image since nothing was tracked before. If tracking is cheap enough to run continuously, you enable it at VM startup; then the first brownout iteration already knows which pages are dirty and can skip the clean ones. For workloads that dirty memory slowly (e.g., CPU-heavy VMs that barely touch most of RAM), that's a big win.
4. **Stop and copy the remainder (blackout).** Pause the VM on the source and copy the last bit of dirty state. This is the only window where the service is actually down. At the end of this step, the source and destination are identical — either could be restarted. Once the destination acknowledges the copy, the migration is committed (in the database transaction sense — it's officially done and can't be half-finished).
5. **Redirect the network.** Send a "gratuitous ARP" packet — essentially an unsolicited announcement to the local network saying "this IP address now lives at this new MAC address" — so traffic starts arriving at the new host. A few packets in flight may be lost, but that kind of packet loss happens on normal networks anyway, and TCP will retransmit and recover.
6. **Resume on the new host.** The VM starts running again on the destination.
7. **Delete the source.** Tear down the original VM on the source host. No residual dependencies — the old host is completely free.

The two numbers to watch when a migration runs long are **blackout duration** and **dirty-page count**. High dirty-page counts point to workload pressure — the VM is writing memory fast enough that brownout iterations can't shrink. Long blackouts with modest dirty-page counts point to network pressure — the final copy is slow because bandwidth between source and target is the bottleneck, not the amount of data.

## Variants

There are a few flavors of live migration. **Managed migration** moves the OS without its cooperation — the VMM does everything from outside, and the guest OS doesn't even know it's happening. **Managed migration with paravirtualization** is the same thing but with the guest OS pitching in on specific optimizations: for instance, *stunning* (briefly freezing) rogue processes that are dirtying memory too fast to keep up with, or pushing unused memory pages out of the VM so they don't need to be copied at all. ("Paravirtualization" means the guest OS is modified to cooperate with the VMM, rather than being unaware of it.) **Self migration** goes further: the OS itself drives the migration. This is harder because you're trying to snapshot a running OS using that same OS — it's like trying to take a photo of yourself taking the photo.

## Hyper-V

_Notes to come._

## TDX Live Migration

_Notes to come._
