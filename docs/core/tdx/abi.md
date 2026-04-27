---
title: ABI Reference
parent: Intel TDX
grand_parent: Core Concepts
nav_order: 1
---

# Intel TDX Module ABI: A Cohesive Overview

## What TDX Is and Why It Matters

Intel Trust Domain Extensions (TDX) lets a CPU create hardware-isolated virtual machines called **Trust Domains (TDs)**. The core guarantee: even a compromised hypervisor — the software that normally manages virtual machines — cannot read or tamper with a TD's memory or processor state. The **TDX Module** is trusted firmware that enforces this boundary. This document describes the binary interface (ABI) that the hypervisor and guest TD software use to communicate with that module.

The system rests on a layered trust model. The CPU and its microcode form the foundation. Above them, the TDX Module runs in a special processor mode called **SEAM** (Secure Arbitration Mode) that the hypervisor cannot enter. The hypervisor talks to the module through `SEAMCALL` instructions, and the guest TD talks to it through `TDCALL` instructions. Each call maps to a specific function (e.g., `TDH.MNG.INIT`, `TDG.MR.REPORT`), and the data types described here are the structured inputs and outputs that flow through those calls.

---

## The Module Lifecycle: From Boot to Running TDs

### 1. Platform Initialization and Enumeration

Before any TD can be created, the hypervisor must discover what the TDX Module supports. This happens in two stages.

**Hardware discovery.** At boot, the system firmware configures **Convertible Memory Ranges (CMRs)** — the physical memory regions that the CPU's built-in verification routine (MCHECK) has confirmed as safe for TDX use. Each CMR is described by a base address and size (both aligned to 4 KB boundaries), and the module returns them sorted and non-overlapping.

**Module enumeration.** The hypervisor calls `TDH.SYS.INFO` (or the newer `TDH.SYS.RDALL`) to retrieve a capability snapshot — a 1,024-byte structure called `TDSYSINFO_STRUCT`. This tells the hypervisor the module version (formatted as `MAJOR.MINOR.UPDATE.INTERNAL.BUILD`, e.g., `1.5.08.04.0234`), whether it's a production or debug build, and critical capacity limits like the maximum number of memory regions and the entry size needed for memory accounting.

The most important part of enumeration is the **TDX_FEATURES bitmap** — a 64-bit field where each bit advertises a capability beyond the baseline. These features range from foundational (bit 0: TD migration support, bit 2: Service TDs) to highly specific (bit 25: single-step instruction-count defense, bit 36: dynamic memory accounting). The hypervisor reads this bitmap to understand which functions are available, which TD settings are legal, and which configuration paths are open.

Enumeration also returns an array of **CPUID configuration entries**. Each entry describes one processor-identification leaf and indicates — via bitmasks — which bits the hypervisor may set when presenting virtualized processor features to a guest TD. This mechanism lets the hypervisor hide hardware features from a TD or present a custom processor topology.

### 2. Memory Configuration

The hypervisor must partition physical memory into **Trust Domain Memory Regions (TDMRs)** — contiguous, 1 GB-aligned regions that TDX will manage. Each TDMR has an associated **Physical Address Metadata Table (PAMT)** at three granularities: 1 GB, 2 MB, and 4 KB. PAMT entries track what each physical page is currently used for — its **page type**: unassigned, TD private memory, a control structure, a page-table entry, or a specialized role like device-assignment structures (TDX Connect).

Within each TDMR, the hypervisor may designate reserved areas (e.g., for PAMT storage itself). These reserved ranges must be sorted and non-overlapping, and all TDMRs and PAMTs must fall within CMRs. The hypervisor passes this layout to `TDH.SYS.CONFIG`, and the module validates and locks it.

### 3. TD Creation and Configuration

Creating a TD is a multi-step process centered on a 1,024-byte blueprint called **TD_PARAMS**. Its fields fall into several categories:

**Security-critical attributes (ATTRIBUTES).** A 64-bit bitmap divided into four trust groups:

- **TUD (TD Under Debug):** Includes the DEBUG flag — if set, the hypervisor can access the TD's memory and processor state, making the TD untrusted.
- **TUP (TD Under Profiling):** Controls whether performance-monitoring data is exposed.
- **SEC (Security):** Contains features that affect security positively (like single-step defense) or negatively (like enabling migration).
- **OTHER:** Covers attested but security-neutral features like performance-monitoring virtualization.

Certain combinations are forbidden: a migratable TD cannot be debuggable, and single-step defense cannot coexist with performance monitoring.

**Extended state (XFAM).** A 64-bit mask that determines which processor extensions (SSE, AVX, AMX, etc.) the TD may use. Must comply with both CPU capabilities and the module's constraints.

**Configuration flags (CONFIG_FLAGS).** Non-attested, TD-scoped controls. The most fundamental is **GPAW**, which sets where the **shared bit** sits in guest physical addresses — this bit distinguishes private (encrypted) memory from memory shared with the hypervisor. Position depends on whether 4-level or 5-level page tables are in use (bit 47 or bit 51, respectively). Other flags control device-assignment enablement, physical-address-width virtualization, and guest page-release capability.

**Measurement identities.** Three 384-bit hashes — `MRCONFIGID`, `MROWNER`, `MROWNERCONFIG` — are software-defined identifiers that become part of the TD's attestation report. They let a remote verifier distinguish who built the TD, who owns it, and what workload-specific configuration was applied.

**Processor-feature virtualization.** An array of entries, one per configurable processor-identification leaf. Only bits that enumeration reported as configurable may be set.

The hypervisor passes `TD_PARAMS` to `TDH.MNG.INIT`, which initializes the TD's internal control structures and locks the attested configuration.

---

## Memory Management: The Secure EPT

TD private memory is mapped through a **Secure Extended Page Table (Secure EPT)** that only the TDX Module can modify. The Secure EPT follows the standard Intel page-table hierarchy (up to 5 levels for 5-level paging) but adds TDX-specific state tracking.

### Page Lifecycle

Every Secure EPT leaf entry has a TDX state beyond the standard permission bits:

- **FREE → PENDING → MAPPED** is the normal lifecycle. The hypervisor adds a page with `TDH.MEM.PAGE.AUG` (state becomes PENDING), then the guest accepts it with `TDG.MEM.PAGE.ACCEPT` (state becomes MAPPED, granting full read/write/execute access).
- **BLOCKED** states are used for translation-cache management — before removing or relocating a page, the module blocks new translations, then tracks cache flushes across all processors that might hold the old mapping.
- **EXPORTED / BLOCKEDW** states support live migration — pages can be blocked for writes, exported, re-exported if dirtied, or cancelled.
- **MMIO_MAPPED / MMIO_PENDING** states support TDX Connect, allowing hardware devices to be securely assigned to TDs.

For nested VMs (TD partitioning), a parallel set of entry states mirrors this lifecycle with an `L2_` prefix.

### Page-Level Attributes

Beyond the Secure EPT's own permission bits, each page can carry fine-grained attributes per VM — including read, write, execute (for both supervisor and user modes), page-verification, and shadow-stack bits. The guest TD itself manages these attributes through dedicated calls, and they are included in migration data.

---

## Attestation: Proving a TD Is What It Claims

The attestation flow produces a **TD Report** (`TDREPORT_STRUCT`, 1,024 or 1,280 bytes depending on version) — a hardware-signed statement that a remote verifier can check. The report has three layers:

1. **MAC-protected envelope** (`REPORTMACSTRUCT`, 256 bytes): Contains the CPU's security version, SHA-384 hashes of the other two layers, 64 bytes of caller-supplied data, and a message authentication code (MAC). A type field identifies this as a TDX report and specifies the version.

2. **Module identity** (`TEE_TCB_INFO`, 239 bytes): The TDX Module's own measurement and security version numbers. Includes a secondary version field that reflects the current module version, which may differ from the original if a module update occurred while the TD was running.

3. **TD identity** (`TDINFO_STRUCT`, 448-768 bytes): The TD's own identity — its attributes, allowed extensions, initial-contents measurement, the three software-defined measurement hashes, four runtime-extendable measurement registers, and a hash of any bound service TDs. Version 2 reports add signer identity, product ID, and version fields tied to the TD signing infrastructure.

### TD Signing (TDSIGSTRUCT)

For TDs that use signing, a `TDSIGSTRUCT` (4 KB-aligned) binds a 3,072-bit RSA signature to reference measurements. A policy field specifies which measurements should be compared against the reference values. This enables a signer to vouch for a TD's identity at a specific point in its boot process, after runtime measurements have reached their expected steady-state values.

### Seal Keys (TDKEYREQUEST)

A TD can derive encryption keys through `TDG.MR.KEY.GET` using a request structure that specifies a key policy: which measurements should be mixed into the key derivation. This lets a TD bind encrypted data to its own identity with the desired granularity — for example, tying data to a specific owner, workload, or code version.

---

## Migration: Moving a TD Between Platforms

TD migration is orchestrated through a **Migration TD** (a special service TD) and structured around epochs and streams.

### The Migration Bundle

All migrated state travels in migration bundles, each with a metadata header (**MBMD**, up to 128 bytes). The header identifies the bundle type: immutable TD-scope state, mutable TD-scope state, per-virtual-processor state, private memory pages, or control tokens (epoch/abort). Every header carries a per-stream counter, a migration epoch number, and an initialization vector counter for AES-256-GCM encryption — the authentication tag at the end covers both the metadata and any associated page data.

### Memory Migration

Private memory pages are migrated using **GPA lists** — 4 KB pages containing up to 512 entries, each specifying a guest physical address, its page-table entry state, and an operation code (MIGRATE, CANCEL, REMIGRATE, or NOP). The migration flow goes: block pages for writing, encrypt and export page content, then decrypt and install on the destination. The list format is designed so that the output of one step feeds directly into the next.

Each exported page gets a per-page authentication tag stored in a MAC list (up to 256 entries of 128-bit values). For partitioned TDs, an attributes list travels alongside, carrying nested-VM attributes for each address.

The migration buffers themselves are shared-memory pages referenced by compact structures that pack a physical address and entry range into a single 64-bit register value, making them efficient to pass through the register-based `SEAMCALL`/`TDCALL` interfaces.

### Epoch Tokens and Dirty Tracking

Migration proceeds in **epochs**. An epoch token records the total number of bundles exported so far. Between epochs, dirtied pages (tracked via a dedicated page-table state) can be re-exported. This enables iterative convergence toward a final cutover with minimal downtime — each round transfers only the pages that changed since the last round.

---

## Supporting Subsystems

### Service TDs

Service TDs are special TDs that provide services (currently: migration) to other TDs. The binding between a service TD and its client is tracked in a binding table within the client's control structures. Each binding entry records a state, a type, a UUID, and an identity hash — a SHA-384 hash of the service TD's identity, with configurable masking that lets the client choose which of the service TD's measurements matter for the binding.

### TD Partitioning (Nested VMs)

A TD can host up to three nested VMs (L2 VMs), each with its own Secure EPT. Guest-side functions transfer control between the primary VM (L1) and nested VMs (L2), using a 154-byte transition structure containing register state, flags, instruction pointer, and interrupt status. Nested-VM page-table management follows the same state machine as the primary level, with L2-prefixed states.

### Fatal Error Diagnostics

If the TDX Module detects an internal fatal error, it logs a 64-byte diagnostic structure to a hypervisor-specified memory location. The basic format includes the processor ID of the faulting core and an error code specific to the module release. Extended formats provide context-specific details: TD handle for TD-specific errors, guest address and page-table level for address-translation failures, physical address for memory errors, metadata-table contents for accounting errors, exception type for unexpected exceptions, or exit reason for unexpected VM exits.

### Metadata Access

All control-structure fields — global, TD-scope, and per-virtual-processor — are accessed through a uniform 64-bit field identifier. This identifier packs a field code, element size, scope (platform/TD/virtual processor), class code, and optional sequencing information for bulk reads. This uniform encoding enables functions like `TDH.SYS.RDALL` to enumerate all module metadata in a single sweep, and it provides the serialization format for migration state transfer.

---

## How It All Fits Together

The TDX ABI is designed around a principle of **progressive trust establishment**:

1. **Boot:** Hardware verifies memory (CMRs). The module enumerates its capabilities.
2. **Configure:** The hypervisor partitions memory (TDMRs + PAMTs), staying within verified boundaries.
3. **Create:** The hypervisor specifies a TD's identity (`TD_PARAMS`), which the module validates and locks.
4. **Build:** Pages are added and accepted, building the TD's address space through the Secure EPT.
5. **Run:** The TD executes with hardware-enforced isolation. TD entry/exit carries detailed exit information for precise error diagnosis.
6. **Attest:** The TD generates a hardware-signed report that a remote verifier can check against expected measurements.
7. **Migrate (optional):** State is exported, transferred, and imported using authenticated encryption, with epoch-based dirty tracking for live migration.
8. **Tear down:** Pages are blocked, translation caches are tracked, and memory is reclaimed — with cache-writeback requirements depending on platform capabilities.

Every data type in the ABI serves one or more of these stages, and the feature bitmap (`TDX_FEATURES`) acts as the master switch that determines which stages and data types are available on a given platform.
