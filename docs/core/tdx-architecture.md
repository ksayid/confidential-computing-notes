---
title: Intel TDX — Architecture & Threat Model
parent: Core Concepts
nav_order: 10
---

# Intel TDX: Architecture & Threat Model

## The Problem TDX Solves

Cloud computing has a fundamental trust problem. When you run a workload on someone else's hardware, the platform owner — the cloud provider, their administrators, their hypervisor (the software that manages virtual machines) — can, in principle, see everything: your data in memory, your processor state, your code. The entire software stack above the hardware — including the virtual machine manager, firmware, and operating system — sits inside the trust boundary, meaning a compromised hypervisor or a rogue administrator can read or tamper with tenant data. Intel TDX (Trust Domain Extensions) exists to flip that model: it carves out hardware-enforced isolated environments called Trust Domains (TDs), where the cloud provider's own software is locked out.

This builds on the trajectory Intel started with SGX (Software Guard Extensions) in 2015, which provided application-level isolation, but scales it up to full virtual machines — which is what cloud workloads actually look like.

## How It Works — The Core Architecture

TDX combines three things: extensions to Intel's existing virtualization instruction set (VMX), hardware-based memory encryption (TME-MK, or Total Memory Encryption — Multi-Key), and a new CPU-attested software module that acts as a trusted referee between the hypervisor and tenant VMs.

**The TDX Module and SEAM Mode.** The CPU gets a new operating mode called Secure Arbitration Mode (SEAM), which hosts an Intel-signed security module. This module lives in a protected memory range that nothing else on the platform can touch — not the hypervisor, not devices performing direct memory access, not system management mode. The module is the gatekeeper: it manages TD creation, deletion, scheduling, and enforces all security policies. The hypervisor still allocates resources (it remains the resource manager), but the TDX module ensures the hypervisor can't peek inside the TDs it's managing. Think of it as an intermediary that the hypervisor must go through — one that enforces boundaries the hypervisor can't override.

**Memory Encryption and Integrity.** Every TD gets a unique, temporary AES-128-XTS encryption key — a per-session key using a standard block cipher mode designed for data-at-rest protection — that the hardware generates and that no software, not even the TDX module, can extract. All private memory for that TD is encrypted with its key.

On top of encryption, TDX offers two integrity modes. The stronger one (cryptographic integrity) attaches a message authentication code (MAC) derived from SHA-3 (a cryptographic hash function) to every 64-byte block of memory, catching both software tampering — such as rowhammer attacks, where repeated access to one memory row causes electrical interference that flips bits in neighboring rows — and some physical attacks. The lighter one (logical integrity) uses a single-bit ownership tag to prevent software from accessing another TD's encrypted data, but doesn't defend against hardware-level tampering. Importantly, neither mode stops replay attacks via physical access — an acknowledged limitation where an attacker records old memory contents and substitutes them back later.

**Address Translation Integrity.** A TD's memory is split into private pages (encrypted with the TD's key) and shared pages (used to communicate with the outside world for input/output, networking, and storage). A single bit in the guest physical address distinguishes the two. The critical design choice is that TDX maintains a secure page table — specifically, an Extended Page Table (EPT), which maps a VM's memory addresses to physical hardware addresses — for private memory, managed through the TDX module so the hypervisor can't silently remap pages. A metadata structure called the Physical Address Metadata Table (PAMT) tracks every page, ensuring no page is double-mapped across TDs, every page is initialized before use, and address-translation caches are properly invalidated when pages are removed. The CPU also enforces that code can't execute from shared memory and page tables can't live in shared memory, closing off a class of code-injection attacks.

**CPU State Protection.** When a TD stops running (on a VM exit — the moment when control transfers from the virtual machine back to the hypervisor), its full CPU state — general-purpose registers, control registers, configuration registers, debug registers, performance counters, and extended state — is saved into a state-save area encrypted with the TD's private key. The TDX module then scrubs those registers before handing control back to the hypervisor. When the TD resumes, its state is restored. The hypervisor never sees the TD's register contents.

**Interrupt Delivery.** TDX uses existing hardware-level interrupt virtualization (APIC virtualization) so that hardware signals are delivered to TDs without transferring control to the hypervisor — and without giving the hypervisor a window to inspect TD state. The architecture blocks the hypervisor from injecting arbitrary error conditions into a TD. Non-maskable interrupts (NMIs) — high-priority hardware signals that can't be ignored — are delivered through controlled TDX module functions that preserve standard interrupt-handling behavior.

## Remote Attestation — Proving the Environment Is Real

Attestation is how a tenant (or any relying party) verifies that their workload is actually running inside a genuine TD on genuine Intel TDX hardware, at a known security level. The flow works like this: the TD requests a report from the TDX module; the CPU generates a hardware-bound report structure (signed with a message authentication code); and an SGX-based quoting enclave — a dedicated signing component — converts that report into a remotely verifiable quote, signed with a digital signature key (using ECDSA, an elliptic-curve signing algorithm). The quote includes TD measurements (a cryptographic hash chain of everything loaded into the TD at creation), runtime measurements added by TD software, the security version numbers of every component in the trusted computing base (TCB — the set of hardware and software components whose correct operation is critical to security), and any data the TD wants to bind to the attestation (like a public key for secure communication). Intel backs this with a certification infrastructure rooted in Intel-issued certificates, so anyone can verify the full signature chain.

If a vulnerability is patched, the attestation rotates — new keys are generated reflecting the updated trusted computing base, and relying parties can confirm the fix is in place.

## What TDX Doesn't Do

TDX is explicit about its boundaries. The hypervisor is still the resource manager, so denial-of-service attacks (the hypervisor refusing to schedule a TD or starving it of memory) are out of scope. Physical replay attacks on memory aren't detected. And TDX doesn't protect against attacks on the TD's own software — if the code running inside the TD is buggy or compromised, TDX can't help.

## The Threat Model in Plain Terms

TDX defends against two types of adversaries. A software adversary — a compromised hypervisor, a malicious administrator, rogue firmware — who can read and write system memory, manipulate page tables, and inject interrupts. TDX counters this through encryption, integrity MACs, secure page tables, and controlled interrupt delivery. A hardware adversary — someone with physical access to the memory bus — who can freeze and read memory or inject data. TDX counters this partially through encryption and cryptographic integrity, but replay attacks remain a gap.

## What's Coming Next

TDX 1.0 covers the core isolation story. Future updates add live migration (moving a running TD between platforms without breaking confidentiality), TD partitioning (running up to 4 total VMs inside a single TD — one primary VM acting as a local hypervisor plus up to 3 nested guest VMs — so unmodified legacy guests can run in a TD without operating system changes), and VM-preserving updates (patching the TDX module at runtime in milliseconds instead of rebooting the host, targeting 99.999% availability).
