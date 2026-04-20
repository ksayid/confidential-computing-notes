---
title: Intel TDX
parent: Core Concepts
nav_order: 4
has_children: true
---

# Intel TDX

Intel Trust Domain Extensions (TDX) is Intel's VM-level confidential computing technology. It protects whole virtual machines — called **Trust Domains (TDs)** — from a potentially malicious host environment, in contrast to SGX, which protects individual enclaves within a process.

This page is the conceptual overview. For deeper material, see:

- [ABI reference](abi.md) — the binary interface the hypervisor and guest TDs use to talk to the TDX Module.
- [Trusted I/O (TDISP)](trusted-io.md) — bringing devices like GPUs and NICs into the TD's trust boundary.
- [Live migration](live-migration.md) — TDX-specific live migration via Migration TDs.

## The core idea

Cloud computing has a fundamental trust problem: when you run a workload on someone else's hardware, the platform owner — the cloud provider, their administrators, their hypervisor — can in principle see everything: your data in memory, your processor state, your code. The entire software stack above the hardware sits inside the trust boundary, meaning a compromised hypervisor or a rogue administrator can read or tamper with tenant data.

TDX flips that model. It creates a new kind of VM, the **Trust Domain**, whose contents are sealed off using hardware encryption so the host cannot read them. The host software can still manage the TD (schedule it, allocate resources), but it can no longer peek inside.

TDX achieves this by extending two existing Intel technologies — hardware virtualization (VMX) and full-memory encryption (TME-MK) — and introducing a privileged software component called the **TDX Module**. The module sits between the host and the guest TD, acting as a gatekeeper. It uses dedicated processor instructions and the chip's encryption engine to enforce isolation. Neither the host software nor any other software on the platform can bypass it.

This builds on the trajectory Intel started with SGX in 2015, which provided application-level isolation, but scales it up to full VMs — which is what cloud workloads actually look like. Importantly, TDX doesn't break the existing world: a TDX-aware host can run TDs alongside traditional VMs with no impact on legacy guests.

## Threat model

TDX assumes the attacker has privileged control of the hypervisor/VMM, the host OS, BIOS, System Management Mode, legacy VMs, other TDs, and any other non-TD software. It also defends against a hardware adversary with physical access to the memory bus, who can freeze and read memory or inject data.

Despite this, TDX guarantees:

- **Confidentiality** — no unauthorized entity can read TD data.
- **Integrity** — no unauthorized entity can tamper with TD data.

**Availability is explicitly excluded.** The host VMM can always deny a TD platform resources — refusing to schedule it, starving it of memory. Physical replay attacks on memory aren't fully detected either, and TDX doesn't protect against attacks on the TD's *own* software: if the code running inside the TD is buggy or compromised, TDX can't help.

## Architecture

TDX combines extensions to Intel's virtualization instruction set, hardware-based memory encryption, and a CPU-attested software module that acts as a trusted referee between the hypervisor and tenant VMs.

### SEAM and the TDX Module

The CPU gets a new operating mode called **SEAM (Secure Arbitration Mode)** that hosts an Intel-signed security module. SEAM lives in a protected memory range — defined by the **SEAM Range Register (SEAMRR)** — that nothing else on the platform can touch: not the hypervisor, not devices performing direct memory access, not system management mode.

The **Intel TDX Module** is the core runtime firmware running inside SEAM. It enforces all TD security guarantees — managing TD lifecycle, memory encryption, page-table integrity, and (in v1.5) migration operations. The hypervisor still allocates resources, but the module ensures the hypervisor can't peek inside the TDs it's managing.

New x86 instructions let the host VMM call into the TDX Module (the "Host Interface", `SEAMCALL`) and let guest TDs call in (the "Guest Interface", `TDCALL`).

### TME-MK (memory encryption)

**TME-MK (Total Memory Encryption, Multi-Key)** provides per-TD memory encryption. Each TD is assigned a **Host Key Identifier (HKID)**, and the TDX Module programs a unique private encryption key for that HKID into hardware. The key is AES-128-XTS — a per-session key using a standard block cipher mode designed for data-at-rest protection. The hardware generates the key and no software, not even the TDX Module, can extract it.

On top of encryption, TDX offers two integrity modes:

- **Cryptographic integrity** — attaches a SHA-3-derived MAC to every 64-byte block of memory, catching both software tampering (such as rowhammer) and some physical attacks.
- **Logical integrity** — uses a single-bit ownership tag to prevent software from accessing another TD's encrypted data, but doesn't defend against hardware-level tampering.

Neither mode stops replay attacks via physical access — an acknowledged limitation where an attacker records old memory contents and substitutes them back later.

### Address translation and the Secure EPT

A TD's memory is split into **private pages** (encrypted with the TD's key) and **shared pages** (used to communicate with the outside world for I/O, networking, and storage). A single bit in the guest physical address distinguishes the two — the **shared bit**, whose position depends on whether 4-level or 5-level page tables are in use (bit 47 or bit 51).

TDX maintains a **Secure Extended Page Table (Secure EPT)** for private memory, managed through the TDX Module so the hypervisor can't silently remap pages. A metadata structure called the **Physical Address Metadata Table (PAMT)** tracks every page, ensuring no page is double-mapped across TDs, every page is initialized before use, and address-translation caches are properly invalidated when pages are removed. The CPU also enforces that code can't execute from shared memory and page tables can't live in shared memory, closing off a class of code-injection attacks.

### CPU state protection

When a TD stops running (on a VM exit), its full CPU state — general-purpose registers, control registers, configuration registers, debug registers, performance counters, extended state — is saved into a state-save area encrypted with the TD's private key. The TDX Module then scrubs those registers before handing control back to the hypervisor. When the TD resumes, its state is restored. The hypervisor never sees the TD's register contents.

### Interrupt delivery

TDX uses existing hardware-level interrupt virtualization (APIC virtualization) so that hardware signals are delivered to TDs without transferring control to the hypervisor — and without giving the hypervisor a window to inspect TD state. The architecture blocks the hypervisor from injecting arbitrary error conditions into a TD. Non-maskable interrupts (NMIs) are delivered through controlled TDX Module functions that preserve standard interrupt-handling behavior.

For inter-processor interrupts within a TD, the TD writes to an emulated interrupt controller and the host delivers the signal using hardware support that injects it directly into the target processor. But this is an untrusted path — the TD's OS has to independently verify that interrupts were actually delivered rather than blindly trusting the host.

### MCHECK and the SEAM Loaders

**MCHECK** is a closed-source microcode component that runs at boot. It verifies platform configuration (correct SEAMRR setup, convertible memory ranges), stores configuration data securely inside SEAMRR, and validates that all cores and packages provide compatible CPU features.

The **SEAM Loaders** form a two-stage loading chain: the Non-Persistent SEAM Loader (NP-SEAMLDR) is an Authenticated Code Module that verifies and loads the Persistent SEAM Loader, which in turn verifies and loads or updates the TDX Module itself.

## Memory model

Memory in a TD comes in two fundamental flavors. **Private memory** is encrypted with the TD's unique key — the host cannot read or tamper with it. **Shared memory** is jointly accessible to the TD and the host, encrypted with host-managed keys, and is the mechanism through which the TD exchanges data with the outside world.

Beyond these two, the guest startup firmware also deals with **unaccepted memory** (pages the host has allocated but the guest hasn't yet acknowledged and taken ownership of) and **device memory regions** (hardware-mapped memory that the TD accesses indirectly through host calls; described in hardware configuration tables provided to the guest).

The TD's operating system manages its own private address space and can convert pages between private and shared by toggling the shared bit. Flipping a page to shared lets the host map I/O buffers into that space; flipping it back to private re-encrypts it under the TD's own key. This dynamic conversion means the TD doesn't need a large, permanently reserved shared region — it can expand and contract shared memory on demand.

There are rules around these conversions. If a page is assigned to a hardware device for direct data transfers, converting it from shared to private must fail — you can't pull a page out from under an active device. With a software-managed or cooperative device interface, the guest can signal that it's no longer using the page and the conversion can proceed as long as the page isn't locked.

### Guest-Host Communication Interface (GHCI)

Since the host can't reach into a TD, all communication flows through a controlled interface called the **Guest-Host Communication Interface (GHCI)**. When the TD tries to do something the hardware won't allow directly (like an I/O instruction), the TDX Module intercepts it and delivers a Virtualization Exception to the guest — a structured notification that the operation requires host involvement. The TD's exception handler processes the event, decides what it needs from the host, and issues a special instruction to make the request. The host receives the request, fulfills it, and hands control back to the TD.

Through this channel, the TD can ask the host to carry out specific operations it can't perform itself — querying processor capabilities, pausing execution, port-based I/O, reading and writing system configuration registers, accessing device memory. Cooperative device drivers — software drivers designed to work with the host directly rather than pretending to use physical hardware — place their data buffers in shared memory so the host can access them, and typically pre-allocate a shared buffer pool at startup to avoid constant page conversions.

## Trust model: assume the host is hostile

This is the defining principle of TDX's security posture. Every interaction with the host is treated as untrusted. Any data the TD receives from the host — responses, notifications, buffer contents — must be validated before use. The TD should never rely on host-provided event notifications for security-critical operations.

Specific safeguards are required throughout. When the TD receives response data containing lengths, offsets, or indices, it must apply defenses against speculative execution attacks before parsing those values. When sending data to the host (for example, requesting an identity proof), the TD should zero out any unused buffer space to avoid accidentally leaking information.

Even "success" responses need scrutiny. When the host relays a service command and reports success, that only means the relay itself worked — it says nothing about whether the underlying operation succeeded. The TD must always parse the actual response payload to determine the real outcome.

## Attestation

Attestation is how a TD proves its identity and integrity to a remote party — essentially saying, "I am running the software I claim to be running, on genuine TDX hardware."

The flow has two layers. First, the TD requests a **TDReport** from the TDX Module — a hardware-bound report structure containing platform configuration and software measurements, signed with a MAC. Then an **SGX-based quoting enclave** (a dedicated signing component, isolated in its own protected memory region) verifies the TDReport and signs it with an ECDSA digital signature key, producing a **TDQuote** that a remote verifier can check.

The quote includes:

- TD measurements — a cryptographic hash chain of everything loaded into the TD at creation,
- runtime measurements added by TD software,
- the security version numbers of every component in the trusted computing base,
- and any data the TD wants to bind to the attestation (like a public key for secure communication).

Intel backs this with a certification infrastructure rooted in Intel-issued certificates, so anyone can verify the full signature chain. Intel verifies that a TDQuote came from genuine hardware, but **customers are responsible for verifying report content** against their own security policy.

The quote request itself is asynchronous — a successful call only means the host received the request, not that the quote is ready. The TD has to wait for an event notification and then check a shared buffer for the result. Multiple concurrent quote requests are allowed.

Attestation also underpins storage encryption. Since a TD's storage is typically encrypted, the firmware uses attestation during boot to fetch volume encryption keys from a remote key server (see [Secure Key Release](../secure-key-release.md)), then passes them to the OS through a dedicated hardware configuration table.

For the on-the-wire structures (`TDREPORT_STRUCT`, `REPORTMACSTRUCT`, `TDINFO_STRUCT`, `TDSIGSTRUCT`, `TDKEYREQUEST`), see the [ABI reference](abi.md#attestation-proving-a-td-is-what-it-claims).

## SVNs and TCB recovery

Every TCB component (TDX Module, SEAM Loaders, CPU hardware) carries a **Security Version Number (SVN)**, embedded in attestation reports and checked at startup against minimum thresholds. When a vulnerability is fixed, the SVN is incremented. **TCB Recovery (TCB-R)** is the process that causes older, vulnerable versions to be flagged as insecure through attestation so that relying parties stop trusting them.

If a vulnerability is patched, the attestation rotates — new keys are generated reflecting the updated TCB, and relying parties can confirm the fix is in place.

## TD state machines

The TDX Module tracks each TD using two primary state machines, plus per-page state.

**Lifecycle state** lives in the **Trust Domain Root (TDR)** and tracks the high-level resource lifecycle: *HKID Assigned* (initial state after creation) → *Keys Configured* (encryption keys programmed — where the TD spends most of its life) → *Blocked* (resources being reclaimed) → *Teardown* (final teardown).

**Operation state** lives in the **TD Control Structure (TDCS)** and controls which operations are permitted at any given time. It is the primary gating mechanism for API access and is the more complex of the two machines. Key states include *Initialized*, *Runnable*, and several migration-specific states (see [Live Migration](live-migration.md)).

**Secure EPT entry state** is carried per-entry in the Secure EPT, ensuring that page-level operations (reads, writes, migrations) only happen when the entry is in an acceptable state for that operation.

## TD build sequence

Building a TD from scratch follows a structured sequence:

1. **Allocate and assign.** A root page is allocated for the TD and a unique HKID is assigned.
2. **Configure encryption.** The HKID and its private encryption key are programmed into TME-MK.
3. **Add control structures.** Multiple pages are added and initialized for the TD's internal control structures.
4. **Initialize TD state.** Global-scope state is initialized: CPUID configuration, virtualization controls, MSR bitmaps, measurement fields, and nested-VM settings. Data comes from both the TDX Module's internal structures and parameters supplied by the host VMM. After this step the TD enters the **Initialized** operation state.
5. **Create Virtual Processors.** One or more VPs are created and initialized with their own control-structure pages. This can happen in any order and from any logical processor.
6. **Populate memory.** The Secure EPT paging hierarchy is built and private memory pages are added with their contents. Each page's contents are incorporated into the TD's measurement.
7. **Finalize.** Measurements are locked. The TD enters the **Runnable** operation state and can begin executing.

## Multiprocessor startup

When a TD boots, the guest firmware sets up a mailbox-based wakeup structure so the primary processor can bring additional processors online. Each virtual processor gets its own mailbox for receiving operating-system-level messages during the wakeup sequence.

## Service TDs

A **Service TD** is a special Trust Domain that provides a dedicated utility — such as a security service or management function — to one or more target TDs. Connections between Service TDs and target TDs are many-to-many and can be established without the target TD's approval. Once connected, the Service TD can read and write specific metadata of the target TD (limited by the Service TD's designated role), and its presence is reflected in the target TD's attestation report, effectively extending the target's set of trusted components.

The current canonical Service TD is the **Migration TD (migTD)**, which brokers cross-platform live migration — see [Live Migration](live-migration.md).

## What TDX doesn't do

TDX is explicit about its boundaries:

- The hypervisor is still the resource manager, so **denial-of-service attacks are out of scope** (the hypervisor refusing to schedule a TD or starving it of memory).
- **Physical replay attacks** on memory aren't fully detected.
- TDX doesn't protect against attacks on the **TD's own software** — if the code running inside the TD is buggy or compromised, TDX can't help.

## What's coming next

TDX 1.0 covers the core isolation story. Subsequent updates add:

- **Live migration** — moving a running TD between platforms without breaking confidentiality (see [Live Migration](live-migration.md)).
- **TD partitioning** — running up to 4 total VMs inside a single TD (one primary VM acting as a local hypervisor plus up to 3 nested guest VMs), so unmodified legacy guests can run in a TD without OS changes.
- **VM-preserving updates** — patching the TDX Module at runtime in milliseconds instead of rebooting the host, targeting 99.999% availability.
- **Trusted I/O** — bringing devices into the TD's trust boundary via TDISP (see [Trusted I/O](trusted-io.md)).
