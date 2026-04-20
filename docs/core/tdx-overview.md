---
title: Intel TDX — A Cohesive Overview
parent: Core Concepts
nav_order: 9
---

# Intel TDX: A Cohesive Overview

## The Core Idea

Intel TDX solves a fundamental trust problem in cloud computing: when you run a virtual machine on someone else's hardware, the host software — the hypervisor and the entire management layer underneath you — can see everything. Your memory, your processor state, your data. TDX changes that. It creates a new kind of VM, called a **Trust Domain (TD)**, whose contents are sealed off using encryption so the host cannot read them. The host software can still manage the TD (schedule it, allocate resources), but it can no longer peek inside.

TDX achieves this by extending two existing Intel technologies — hardware virtualization and full-memory encryption — and introducing a privileged software component called the **TDX module**. This module sits between the host and the guest TD, acting as a gatekeeper. It uses dedicated processor instructions and the chip's encryption engine to enforce isolation. Neither the host software nor any other software on the platform can bypass it.

Importantly, TDX doesn't break the existing world. A TDX-aware host can run Trust Domains alongside traditional VMs with no impact on legacy guests. The host is only restricted in what it can do with TDs — everything else works as before.

## Memory: Private, Shared, and Everything Else

Memory in a TD comes in two fundamental flavors. **Private memory** is encrypted with a one-time key unique to that TD — the host cannot read or tamper with it. **Shared memory** is jointly accessible to the TD and the host, encrypted with host-managed keys, and is the mechanism through which the TD exchanges data with the outside world (input/output buffers, communication channels, etc.).

Beyond these two, the guest startup firmware also deals with **unaccepted memory** (pages the host has allocated but the guest hasn't yet acknowledged and taken ownership of) and **device memory regions** (hardware-mapped memory that the TD accesses indirectly through host calls rather than direct reads and writes; these are described in hardware configuration tables provided to the guest).

The TD's operating system manages its own private address space and can convert pages between private and shared by toggling a bit in the guest's internal address. Flipping a page to shared lets the host map input/output buffers into that space; flipping it back to private re-encrypts it under the TD's own key. This dynamic conversion means the TD doesn't need a large, permanently reserved shared region — it can expand and contract shared memory on demand, similar to how traditional VMs can dynamically return unused memory to the host and reclaim it later.

There are rules around these conversions. If a page is assigned to a hardware device for direct data transfers, converting it from shared to private must fail — you can't pull a page out from under an active device. With a software-managed or cooperative device interface, the guest can signal that it's no longer using the page for device transfers, and the conversion can proceed as long as the page isn't locked.

## How the TD Talks to the Host

Since the host can't reach into a TD, all communication flows through a controlled interface called the **Guest-Host Communication Interface (GHCI)**. The mechanics work like this: when the TD tries to do something the hardware won't allow directly (like an input/output instruction), the TDX module intercepts it and delivers a Virtualization Exception to the guest — a structured notification that the operation requires host involvement. The TD's exception handler processes the event, decides what it needs from the host, and issues a special instruction to make the request. The host receives the request, fulfills it, and hands control back to the TD.

Through this channel, the TD can ask the host to carry out specific operations it can't perform itself — things like querying processor capabilities, pausing execution, port-based input/output, reading and writing system configuration registers, and accessing device memory. Cooperative device drivers — software drivers designed to work with the host directly rather than pretending to use physical hardware — place their data buffers in shared memory so the host can access them. These drivers typically pre-allocate a shared buffer pool at startup to avoid constant page conversions.

For inter-processor interrupts (signals sent between virtual processors within the TD), the TD writes to an emulated interrupt controller and the host delivers the signal using hardware support that injects it directly into the target processor without software intervention. But this is an untrusted path — the TD's operating system has to independently verify that interrupts were actually delivered rather than blindly trusting the host.

## The Trust Model: Assume the Host is Hostile

This is the defining principle of TDX's security posture. Every interaction with the host is treated as untrusted. Any data the TD receives from the host — responses, notifications, buffer contents — must be validated before use. The TD should never rely on host-provided event notifications for security-critical operations.

Specific safeguards are required throughout. When the TD receives response data containing lengths, offsets, or indices, it must apply defenses against speculative execution attacks (where a processor's predictive behavior can be exploited to leak data) before parsing those values. When sending data to the host (for example, requesting an identity proof), the TD should zero out any unused buffer space to avoid accidentally leaking information.

Even "success" responses need scrutiny. When the host relays a service command and reports success, that only means the relay itself worked — it says nothing about whether the underlying operation succeeded. The TD must always parse the actual response payload to determine the real outcome.

## Attestation: Proving You Are What You Claim

Attestation is how a TD proves its identity and integrity to a remote party — essentially saying, "I am running the software I claim to be running, on genuine TDX hardware." The process has two layers. First, the guest firmware produces a tamper-evident log recording what was loaded and in what order, tied to a set of secure measurement registers that capture the TD's boot-time state. Then, when the TD needs to prove itself, it asks the TDX module for a locally authenticated report. A trusted signing component on the platform — isolated in its own protected memory region — verifies this report and signs it with a strong digital signature, producing a **TD Quote** that a remote verifier can check.

The quote request itself is asynchronous — a successful call only means the host received the request, not that the quote is ready. The TD has to wait for an event notification and then check a shared buffer for the result. Multiple concurrent quote requests are allowed, though the details are implementation-specific, and the TD should be prepared to handle retry responses.

Attestation also underpins storage encryption. Since a TD's storage is typically encrypted, the firmware uses attestation during boot to fetch volume encryption keys from a remote key server, then passes them to the operating system through a dedicated hardware configuration table.

## Multiprocessor Startup

When a TD boots, the guest firmware sets up a mailbox-based wakeup structure so the primary processor can bring additional processors online. Each virtual processor gets its own mailbox for receiving operating-system-level messages during the wakeup sequence.

## Service TDs

A **Service TD** is a special Trust Domain that provides a dedicated utility — such as a security service or management function — to one or more target TDs. Connections between Service TDs and target TDs are many-to-many and can be established without the target TD's approval. Once connected, the Service TD can read and write specific metadata of the target TD (limited by the Service TD's designated role), and its presence is reflected in the target TD's attestation report, effectively extending the target's set of trusted components.

## Live Migration

One of TDX's more complex features is the ability to migrate a running TD from one physical machine to another without breaking confidentiality. The process involves **Migration TDs** on both the source and destination machines, which establish a secure channel through mutual identity verification and negotiate a one-time encryption key for the transfer.

Migration happens in phases. Fixed TD configuration moves first. Then memory is iteratively pre-copied while the TD continues running. After a brief pause where the source is stopped, the final changing state (per-processor and per-TD) is exported. A commitment protocol using cryptographic tokens ensures that exactly one copy of the TD is runnable at any time — there's never a moment where both source and destination are live simultaneously. After the destination TD resumes, remaining pages can be transferred on demand as they're needed.

The internal page-mapping structures aren't migrated directly — they're rebuilt on the destination — but the TDX module cryptographically verifies that address mappings are recreated correctly, preventing remapping attacks that could expose private memory. Shared memory, which isn't confidential to begin with, migrates through the host's standard mechanisms.

Communication during migration is stream-oriented. The host must guarantee reliable, in-order delivery. The Migration TD keeps requests sequential per migration request ID but can issue concurrent requests across different IDs. Buffer sizing has minimums (64 KB for both command and response buffers), and edge cases are handled: a zero-size command buffer is valid for a destination Migration TD waiting for the first message, a zero-size response buffer is valid for sending a final handshake, but both being zero is an error.

## Specification Conventions

The TDX specification uses a standard convention for requirement language: terms like "must," "should," and "may" carry precise formal meanings — distinguishing absolute requirements from recommendations and truly optional features.
