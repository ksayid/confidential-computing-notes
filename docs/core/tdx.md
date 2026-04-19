---
title: Intel TDX
parent: Core Concepts
nav_order: 8
---

# Intel TDX

Intel Trust Domain Extensions (TDX) is Intel's VM-level confidential computing technology. It protects whole virtual machines — called **Trust Domains (TDs)** — from a potentially malicious host environment, in contrast to SGX, which protects individual enclaves within a process.

## Threat model

TDX assumes the attacker has privileged control of the hypervisor/VMM, the host OS, BIOS, System Management Mode, legacy VMs, other TDs, and any other non-TD software. Despite this, TDX guarantees **confidentiality** (no unauthorized entity can read TD data) and **integrity** (no unauthorized entity can tamper with TD data). **Availability is explicitly excluded** — the host VMM can always deny a TD platform resources.

## Architectural components

**SEAM (Secure Arbitration Mode)** is a CPU execution mode that hosts the TDX Module. It is isolated from the host VMM, other system software, and device DMA by a reserved memory region defined by the **SEAM Range Register (SEAMRR)**. New x86 instructions let the host VMM call into the TDX Module (the "Host Interface") and let guest TDs call in (the "Guest Interface").

The **Intel TDX Module** is the core runtime firmware running inside SEAM. It enforces all TD security guarantees — managing TD lifecycle, memory encryption, page-table integrity, and (in v1.5) migration operations.

**TME-MK (Total Memory Encryption, Multi-Key)** provides per-TD memory encryption and integrity. Each TD is assigned a **Host Key Identifier (HKID)**, and the TDX Module programs a unique private encryption key for that HKID into hardware. Only the TDX Module and the owning TD can access the encrypted memory.

**MCHECK** is a closed-source microcode component that runs at boot. It verifies platform configuration (correct SEAMRR setup, convertible memory ranges), stores configuration data securely inside SEAMRR, and validates that all cores and packages provide compatible CPU features.

The **SEAM Loaders** form a two-stage loading chain: the Non-Persistent SEAM Loader (NP-SEAMLDR) is an Authenticated Code Module that verifies and loads the Persistent SEAM Loader, which in turn verifies and loads or updates the TDX Module itself.

## Attestation

TDs can generate **local attestation reports (TDReports)** containing platform configuration and software measurements. An Intel SGX enclave (the TDQuoting Enclave) signs these to produce **remote attestation reports (TDQuotes)**. Attestation lets external parties verify a TD's execution environment; during Live Migration, it also establishes mutual trust between source and destination TDX Modules. Intel provides infrastructure to verify that a TDQuote came from genuine hardware, but **customers are responsible for verifying report content** against their own security policy.

## SVNs and TCB recovery

Every TCB component (TDX Module, SEAM Loaders, CPU hardware) carries a **Security Version Number (SVN)**, embedded in attestation reports and checked at startup against minimum thresholds. When a vulnerability is fixed, the SVN is incremented. **TCB Recovery (TCB-R)** is the process that causes older, vulnerable versions to be flagged as insecure through attestation so that relying parties stop trusting them.

## TD state machines

The TDX Module tracks each TD using two primary state machines, plus per-page state.

**Lifecycle state** lives in the **Trust Domain Root (TDR)** and tracks the high-level resource lifecycle: *HKID Assigned* (initial state after creation) → *Keys Configured* (encryption keys programmed — where the TD spends most of its life) → *Blocked* (resources being reclaimed) → *Teardown* (final teardown).

**Operation state** lives in the **TD Control Structure (TDCS)** and controls which operations are permitted at any given time. It is the primary gating mechanism for API access and is the more complex of the two machines. Key states include *Initialized*, *Runnable*, and several migration-specific states (see [Live Migration](live-migration.md#tdx-live-migration)).

**Secure EPT entry state** is carried per-entry in the Secure Extended Page Table (SEPT), ensuring that page-level operations (reads, writes, migrations) only happen when the entry is in an acceptable state for that operation.

## TD build sequence

Building a TD from scratch follows a structured sequence:

1. **Allocate and assign.** A root page is allocated for the TD and a unique HKID is assigned.
2. **Configure encryption.** The HKID and its private encryption key are programmed into TME-MK.
3. **Add control structures.** Multiple pages are added and initialized for the TD's internal control structures.
4. **Initialize TD state.** Global-scope state is initialized: CPUID configuration, virtualization controls, MSR bitmaps, measurement fields, and nested-VM settings. Data comes from both the TDX Module's internal structures and parameters supplied by the host VMM. After this step the TD enters the **Initialized** operation state.
5. **Create Virtual Processors.** One or more VPs are created and initialized with their own control-structure pages. This can happen in any order and from any logical processor.
6. **Populate memory.** The Secure EPT paging hierarchy is built and private memory pages are added with their contents. Each page's contents are incorporated into the TD's measurement.
7. **Finalize.** Measurements are locked. The TD enters the **Runnable** operation state and can begin executing.
