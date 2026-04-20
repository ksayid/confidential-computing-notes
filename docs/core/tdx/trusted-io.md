---
title: Trusted I/O (TDISP)
parent: Intel TDX
grand_parent: Core Concepts
nav_order: 3
---

# Intel TDX Trusted I/O (TDISP)

## The Core Problem

When you run a virtual machine inside a trusted execution environment (a hardware-isolated zone often called a "TVM"), the whole point is that you don't trust the host operating system or the hypervisor — they could be compromised. But I/O devices like GPUs, network cards, and accelerators sit outside that protected zone. So today, if a protected VM wants to communicate with a device, it must copy data into a shared memory buffer and encrypt everything in software. That's slow, especially for demanding workloads like AI training or large-scale networking.

## What TDISP Solves

The PCI-SIG (the industry group that governs the PCIe interconnect standard) defined a protocol called **TDISP — TEE Device Interface Security Protocol** — that lets a protected VM bring a device into its trust boundary. Once inside, the device can directly read and write the VM's private memory with no copying and no software encryption overhead.

TDISP doesn't work alone; it relies on two building blocks:

- **SPDM (Security Protocol and Data Model)**, a cross-industry standard from the DMTF, handles device authentication — the device cryptographically proves its identity and that its firmware hasn't been tampered with.
- **IDE (Integrity and Data Encryption)** encrypts the physical PCIe link between the host and the device in hardware, so that even someone physically tapping the bus cannot read or tamper with the data.

## How Intel TDX Implements It

Intel TDX extends its existing hardware security module to act as the **TEE Security Manager** — the trusted referee that TDISP requires on the host side. Because the hypervisor can't be trusted, every sensitive operation flows through this module instead. There are three key areas it controls.

### Secure device-register access

Normally the hypervisor builds the memory mappings that let a VM access device registers (a technique called memory-mapped I/O). For a trust domain, the TDX module manages those mappings in a protected page table so the hypervisor can't tamper with them or eavesdrop on device interactions.

### Secure direct memory access

Similarly, the TDX module takes over configuration of the IOMMU — the hardware unit that controls which memory regions a device can read from or write to directly. The module ensures those regions belong to the correct trust domain and handles clearing stale mappings so they can't be exploited.

### Secure messaging

Even though the hypervisor physically sends TDISP and IDE key-management messages to the device (it has to — it owns the bus), it can't forge or tamper with them. The TDX module prepares the encrypted payloads, and when a trust domain initiates a message, the module verifies that the hypervisor faithfully forwarded exactly what the trust domain intended.

## The Lifecycle, Simplified

The whole flow has four phases:

1. **Host boot.** The host boots and initializes the TDX module with I/O security support, setting up the trusted IOMMU and launching a helper component called the **TPA (TDX Provision Agent)**.
2. **Device authentication.** The TPA authenticates the physical device using SPDM — it collects the device's certificates and firmware measurements and establishes a secure session.
3. **Link encryption setup.** The hypervisor and TDX module cooperate to set up IDE encryption on the PCIe link, provisioning hardware-level encryption keys through the module (never exposed to the hypervisor).
4. **Interface assignment.** A specific device interface gets assigned to a trust domain: the trust domain verifies the device's identity against its own policy, accepts the memory mappings, and moves the interface into a **RUN** state where direct, encrypted, trusted I/O can flow.

## The Software Layering

Much of this design isn't Intel-specific. SPDM and IDE are general operating-system-level features that any platform could implement. TDISP support is a common virtualization-layer concern. Only the bottom layer — the actual calls into the TDX module, the protected page-table management, the trusted IOMMU control — is Intel-specific. This layering means that as other chip architectures (such as AMD's SEV-TIO) adopt TDISP, the upper layers of code should be reusable across vendors.

## The Bottom Line

First-generation TDX kept devices outside the trust boundary and paid for it with expensive software encryption and memory copying. This extension lets trusted devices operate as though they're inside the protected VM, with hardware-enforced isolation and link encryption replacing the software workarounds — and it does so through a standards-based architecture that isn't entirely proprietary to Intel.
