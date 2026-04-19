---
title: Live Migration
parent: Core Concepts
nav_order: 7
---

# Live Migration

Live migration lets a cloud provider patch or upgrade host infrastructure (e.g., host kernel upgrades) without disrupting the VMs running on it. Instead of draining and rebooting a host, VMs are relocated to other hosts while still running.

## How it works

1. A VM is running on a host that needs to go offline.
2. A target VM is created on a different host; state begins copying over (RAM, VM spec, machine type, attached disks, network connections).
3. Once most state is copied, the source VM is briefly paused.
4. The remaining state copies over.
5. The VM is unpaused on the target host.

The pause creates a brief blackout, but for most workloads it's invisible. The payoff: when a major vulnerability needs patching, hosts can be updated without customer-visible reboots — versus the alternative of forcing customers to schedule downtime.

## Hyper-V

_Notes to come._

## TDX Live Migration

_Notes to come._
