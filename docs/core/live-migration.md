---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Migration — moving a running workload from one physical machine to another — is useful for several reasons. It enables **load balancing** for long-lived jobs, where you can shift work off an overloaded server (a natural question: why not short-lived jobs too? Probably because they'll finish before migration pays off). It gives you **controlled maintenance windows** instead of emergency ones, since you can move work off a machine before taking it down. It supports **fault tolerance** by letting you move jobs away from hardware that's getting flaky but hasn't failed yet. And it helps with **energy efficiency** — you can consolidate loads onto fewer machines to reduce cooling (A/C) needs. The data center is the natural environment for all of this, since you have many machines under unified control.

## Two basic strategies for moving state

When you migrate a workload, you have to deal with all the things it refers to — memory, files, network connections, devices. How you handle each piece of state falls into one of two approaches, and which one is available shapes what kinds of migration are even possible.

The first is **local names**: physically move the state to the new machine. This covers things like local memory contents, CPU registers, and local disk. It's the only option when the resource is tied to the original machine, but it doesn't work for some physical devices — a tape drive bolted to the source host can't come with you.

The second is **global names**: the resource has a name that works from anywhere, so you just keep using it from the new location. Network-attached storage is the canonical example — the files live on a separate storage system, and both the old and new machine can reach them by the same name. For network identity, techniques like Network Address Translation (NAT) or layer 2 (Ethernet-level) addressing let an IP address effectively follow the workload to its new host.

This distinction matters because of **residual dependencies** — leftover bits of state on the old machine that force it to stay up after migration (e.g., the old host still serving a local disk or forwarding network traffic for the migrated workload). Every piece of state handled by local names has to either move physically or become a residual dependency. Global names eliminate the choice: the state is already reachable from anywhere, so nothing has to move and nothing has to stay behind. Data centers are designed around global names (networked storage, managed IP addressing), which is what makes clean migration possible there — and it's why the VMM approach below works well in data centers specifically.

## VMM migration

VMM migration (Virtual Machine Monitor migration — moving an entire virtual machine) moves the whole OS as a single unit. You don't need to understand what's running inside or what state belongs to what; it's all just memory and devices from the VMM's perspective. This means you can migrate applications whose source code you don't have, and ones whose owners don't trust you enough to cooperate with a migration. Combined with the global names a data center provides, VMM migration can avoid residual dependencies entirely — the memory moves with the VM, and everything else (storage, network identity) is already reachable from the new host.

Non-live VMM migration (where the VM is suspended, moved, and resumed) is also useful outside the data center. You can migrate your work environment from office to home and back by putting the suspended VM on a USB key or sending it over the network — the "Collective" project called this "Internet suspend and resume."

## Goals of live migration

Live migration — moving the VM while it keeps running — has three goals in tension with each other: minimize downtime (the window where the service is unavailable), keep the total migration time manageable, and limit the impact on both the VM being moved and the local network during the move.

## How live migration works

1. **Allocate resources at the destination.** Before starting, make sure the target host can actually receive the VM (enough memory, CPU, etc.).
2. **Iteratively copy memory pages.** While the VM keeps running on the source, start copying its memory to the destination. The catch: the VM is still writing to memory, so any page that gets modified after you copy it has to be copied again ("dirtied" pages). You iterate — copy everything, then copy the pages that got dirtied during that copy, then the ones dirtied during that pass, and so on. You stop either when only a small amount of dirty memory remains or when you're not making forward progress (the VM is dirtying pages as fast as you can copy them). You can dedicate more bandwidth to later iterations to shrink the window in which new dirtying can happen.
3. **Stop and copy the remainder.** Pause the VM on the source and copy the last bit of dirty state. This is the only window where the service is actually down. At the end of this step, the source and destination are identical — either could be restarted. Once the destination acknowledges the copy, the migration is committed (in the database transaction sense — it's officially done and can't be half-finished).
4. **Redirect the network.** Send a "gratuitous ARP" packet — essentially an unsolicited announcement to the local network saying "this IP address now lives at this new MAC address" — so traffic starts arriving at the new host. A few packets in flight may be lost, but that kind of packet loss happens on normal networks anyway, and TCP will retransmit and recover.
5. **Resume on the new host.** The VM starts running again on the destination.
6. **Delete the source.** Tear down the original VM on the source host. No residual dependencies — the old host is completely free.

## Variants

There are a few flavors of live migration. **Managed migration** moves the OS without its cooperation — the VMM does everything from outside, and the guest OS doesn't even know it's happening. **Managed migration with paravirtualization** is the same thing but with the guest OS pitching in on specific optimizations: for instance, *stunning* (briefly freezing) rogue processes that are dirtying memory too fast to keep up with, or pushing unused memory pages out of the VM so they don't need to be copied at all. ("Paravirtualization" means the guest OS is modified to cooperate with the VMM, rather than being unaware of it.) **Self migration** goes further: the OS itself drives the migration. This is harder because you're trying to snapshot a running OS using that same OS — it's like trying to take a photo of yourself taking the photo.

## Hyper-V

_Notes to come._

## TDX Live Migration

_Notes to come._
