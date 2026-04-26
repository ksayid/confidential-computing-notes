---
title: Microsoft Azure Platform Trust Chain
parent: Reference
nav_order: 6
---

Eight documents — Microsoft's secure-boot, firmware, code-integrity, hypervisor, and compute pages, the Linux kernel's dm-verity guide, and Wikipedia on UEFI and the SHA family — describe a single chain. From the moment power reaches an Azure server, each layer is verified by the layer below it before being allowed to run: silicon root of trust → firmware → bootloader → operating-system kernel → drivers and binaries → hypervisor → tenant workloads. Cryptographic hash functions are the primitive every link in that chain depends on. This note follows the chain top-down, reads the documents as descriptions of layers in it, and connects them to the rest of this repo.

## 1. UEFI: The Firmware-to-OS Interface

**UEFI** (Unified Extensible Firmware Interface) is the open specification that defines how platform firmware presents itself to the operating system. It replaces the legacy BIOS that originated with the IBM PC. Where BIOS was 16-bit, proprietary, capped at 2 TB disks and four primary partitions, and had no built-in cryptographic verification, UEFI is C-based and modular, supports modern GPT-partitioned disks (the GUID Partition Table format) of any practical size, and ships a standardized signature-verification mechanism (Secure Boot) directly in firmware. The current specification is **UEFI 2.11**, published December 17, 2024 by the UEFI Forum. Backward compatibility with BIOS is optional via the Compatibility Support Module (CSM) and is being phased out.

### The boot phases

UEFI firmware progresses through six named phases. The first four matter most for security:

- **SEC (Security Phase)** — earliest stage; minimal assembly that establishes the platform's root of trust before main memory is even initialized. Runs out of CPU cache used as RAM, since DRAM has not been brought up yet. Everything downstream is rooted here.
- **PEI (Pre-EFI Initialization)** — brings up DRAM, CPU, and chipset, and hands a structured set of "Hand-Off Blocks" to the next phase.
- **DXE (Driver Execution Environment)** — C-based driver dispatch; where most UEFI drivers run and where their signatures get checked.
- **BDS (Boot Device Select)** — the boot manager. Reads non-volatile RAM (NV-RAM) boot variables, executes UEFI drivers, selects and verifies the OS bootloader. **Secure Boot lives here.**
- **TSL (Transient System Load)** — intermediate stage where a UEFI shell or OS bootloader runs before the OS takes over.
- **RT (Runtime)** — after the OS calls `ExitBootServices()`, firmware drops to a small set of runtime services callable from the OS (e.g., variable read/write, capsule update).

### The EFI System Partition

The **EFI System Partition (ESP)** is a FAT-formatted disk partition (a UEFI-specific variant of FAT12/16/32) that holds bootloaders, UEFI applications, drivers, and vendor utilities. Files live under `\EFI\BOOT\BOOT<arch>.EFI`. Recommended minimum size is around 512 MB.

### Implementations

Several UEFI implementations ship in production, and they all matter for Azure:

- **TianoCore EDK II** — Intel's open-source reference implementation; the de facto upstream.
- **Project Mu** — Microsoft's December 2018 fork of EDK II, used in Surface and Hyper-V. Designed around "Firmware as a Service" so that quality and security patches can roll across many product lines quickly. Microsoft is adding Rust support for memory-safe UEFI modules.
- **Commercial vendors** — AMI (Aptio), Phoenix (SecureCore), Insyde (InsydeH2O) — all downstream of EDK II, shipping on most OEM hardware.

## 2. Secure Boot: The First Cryptographic Gate

UEFI's boot-phase machinery is the substrate; **Secure Boot** is the first cryptographic check sitting on it. It runs inside the BDS phase and verifies that every UEFI driver, EFI application, and OS bootloader is signed by a trusted authority before it is allowed to execute.

### The four-tier key hierarchy

Secure Boot is governed by four authenticated UEFI variables stored in NV-RAM:

| Variable | Role | Cardinality |
|---|---|---|
| **PK (Platform Key)** | Master asymmetric key — owns the platform; authorizes changes to KEK. | One |
| **KEK (Key Exchange Key)** | Signed by PK; authorizes updates to db and dbx. Does not itself verify boot binaries. | One or more |
| **db (signature database)** | Hashes/certificates of trusted bootloaders, OS loaders, drivers permitted to load. | Many entries |
| **dbx (forbidden signatures)** | Revoked hashes/certificates. **dbx wins** if a hash appears in both. | Many entries |

The chain is: firmware verifies the OS bootloader's signature against `db` (and that no entry in `dbx` matches), then the bootloader hands off to a verified kernel.

### Authenticated variables

Secure Boot's keys are themselves protected by the firmware's *authenticated-variable* mechanism — a write protocol that requires the writer to prove possession of a trusted key before the variable can be modified. A variable flagged with `EFI_VARIABLE_AUTHENTICATED_WRITE_ACCESS` can be modified only if the writer presents a payload signed by a key already trusted by the platform. Mechanically, the caller hashes the data plus a monotonic counter (a value that only ever increases, used here for replay protection) with SHA-256 and signs the digest with an RSA key; firmware verifies before accepting the write.

### Capsule updates

UEFI's signed firmware-update format. The OS hands a "capsule" (signed firmware payload) to firmware via a runtime service; firmware validates the signature and applies the update on the next boot. This is the mechanism Linux's `fwupd` and the Linux Vendor Firmware Service use, and it is how Microsoft pushes signed BIOS updates to its fleet.

### Where Secure Boot has been bypassed

Secure Boot is sound in principle but has had real-world bypasses worth naming:

- **BootHole — CVE-2020-10713 (2020)** — buffer overflow in GRUB2's config-file (`grub.cfg`) parser, exploitable to run arbitrary code inside GRUB before Secure Boot's verification of the kernel completes. Required coordinated updates across GRUB2, `shim` (the small Microsoft-signed first-stage EFI loader that itself loads GRUB on Linux), the kernel, `fwupd`, and `dbx`.
- **BlackLotus (2022–2023)** — first in-the-wild UEFI *bootkit* (malware that installs itself in the early boot path) to bypass Secure Boot on fully patched Windows 11, exploiting CVE-2022-21894 ("Baton Drop") to roll back Windows boot managers to vulnerable signed versions whose hashes were not yet in `dbx`.
- **LogoFAIL (2023, including CVE-2023-40238)** — class of vulnerabilities in firmware image parsers (BMP/PNG/JPEG decoders that draw the boot logo during DXE). A malicious boot logo triggers code execution before signature checks would matter.

These are reminders that Secure Boot's guarantee is "the bootloader's signature is valid," not "the firmware that checks the signature is itself sound." Closing that gap is what the next layer addresses.

## 3. Firmware Integrity: Project Cerberus and the Silicon Root

Secure Boot can only be trusted if the firmware running it is itself trustworthy — and the bypasses just listed all hinged on either bugs in firmware code or weaknesses in how revocations propagate. Microsoft's answer is a hardware root of trust that sits *below* UEFI and gates access to firmware flash itself.

### Hardware root of trust

A **hardware root of trust (RoT)** is a trust anchor implemented in immutable silicon — its identity cannot be cloned, and it is implicitly trusted to verify everything above it. **NIST SP 800-193** ("Platform Firmware Resiliency Guidelines") sets the compliance bar: protection, detection, and recovery for platform firmware.

### Project Cerberus

**Project Cerberus** is Microsoft's open hardware RoT, contributed to the Open Compute Project in 2017 and now widespread on Azure motherboards. It is a cryptographic microcontroller running secure code that intercepts the SPI bus (the Serial Peripheral Interface bus connecting the main CPU to the flash chip where firmware is stored). Every flash read or write is measured and attested. Tampering is detected continuously, not just at boot, so firmware modification at runtime is also caught. Cerberus is CPU- and I/O-architecture-agnostic, supports a hierarchical RoT extending to peripherals (NIC, SSD, GPU, FPGA), and is NIST 800-193 compliant.

The key design move is **separating the firmware-integrity gate from the firmware itself**: the chip that decides whether firmware is allowed to run is not the chip running the firmware, so a compromised UEFI image cannot subvert its own verification.

### Microsoft Pluton

**Microsoft Pluton** is a related but different security processor, originally derived from Xbox and Azure Sphere. Where Cerberus is a discrete chip on the motherboard, Pluton is integrated directly into the System-on-Chip (SoC) — on AMD Ryzen 6000+, Intel Core Ultra, and Qualcomm Snapdragon X — and acts as both a hardware RoT and a TPM. It is default-on for Copilot+ PCs. **Flag:** Microsoft announced in 2020 that Pluton would also ship in Azure servers, but as of early 2026 there is no public evidence of large-scale Pluton-equipped Azure server SKUs. Treat that as not-yet-deployed.

### Anti-rollback

Modern silicon enforces **anti-rollback** through one-time-programmable fuses (Intel calls them Field Programmable Fuses, burned in the chipset's Platform Controller Hub at manufacturing) or monotonic counters. Each firmware image carries a Security Version Number (SVN); the platform refuses any image whose SVN is below the persistent floor. This blocks the "downgrade to a signed-but-vulnerable older version" attack — exactly the BlackLotus pattern, where signed-but-broken Windows boot managers existed for over a year before being revoked.

### Measured boot and the TPM

**Measured boot** runs in parallel to Secure Boot. Each component computes a hash of the next component before executing it and *extends* the value into a **Platform Configuration Register (PCR)** inside a **TPM (Trusted Platform Module)**. A PCR is an append-only register: its value is updated as `PCR_new = hash(PCR_old || measurement)`, never overwritten without reboot. The result is a tamper-evident log of exactly what software ran from power-on onward.

**TPM 2.0** is the standard on Azure hardware. It supports configurable hash algorithms (SHA-256 and beyond, in per-algorithm PCR banks), where TPM 1.2 was hardcoded to SHA-1. It also splits authorization into four hierarchies: **Platform** (used by early-boot firmware), **Storage** (general user-key operations), **Endorsement** (privacy-sensitive identity keys), and **Null** (volatile, cleared on every reboot). Measurements feed two consumers: **remote attestation** (a TPM-signed quote of PCR values that a relying party verifies) and **sealing** (binding secrets so they unlock only when PCRs match expected values).

The contrast is worth holding onto: **Secure Boot says "this stage's signature is valid," measured boot says "and here is exactly what ran."** A relying party that only trusts a signature has a weaker statement than one that can audit the whole chain.

## 4. Code Integrity and dm-verity: Extending the Chain into Runtime

Secure Boot and measured boot together cover power-on through kernel hand-off. After that, neither prevents a running kernel from later loading an unsigned driver, building an executable payload in writable memory, or reading a tampered binary off disk. **Code integrity** (on Windows hosts) and **dm-verity** (on Linux hosts) are how Azure extends the chain from boot into runtime.

### Code integrity on the Windows / Azure Host side

Microsoft's "code integrity" service (Windows Server 2016 and later) is a kernel-level gate that checks drivers, DLLs, executables, and scripts against a policy of allowed signing certificates and SHA-256 file hashes before allowing them to load. Three current technologies layer on top of it:

- **App Control for Business (ACfB)**, formerly **Windows Defender Application Control (WDAC)** — the modern allow-listing engine; renamed with Windows 11 24H2. Works in both user mode and kernel mode and so can block unsigned kernel drivers. The older **AppLocker** runs only in user mode and so cannot. Microsoft's current guidance: prefer App Control; keep AppLocker only for per-user fine-tuning.
- **Driver Signature Enforcement (DSE)** — 64-bit Windows refuses to load unsigned kernel-mode drivers. Tightened over time so that on a default modern install, only Microsoft-attested drivers signed with an Extended Validation (EV) certificate load. EV certs are issued only after stricter identity vetting than ordinary code-signing certs, which gives the platform a higher-trust signer pool for kernel code.
- **Hypervisor-protected Code Integrity (HVCI)**, marketed to consumers as **Memory Integrity** — moves the code-integrity check out of the normal kernel and into the hypervisor's secure world. HVCI relies on **Virtualization-Based Security (VBS)**: a Hyper-V feature that carves a normal Windows install into Virtual Trust Levels, where the normal kernel runs at VTL0 and an isolated "secure kernel" plus small *trustlets* (tiny VTL1 processes that hold secrets the normal kernel must not see — e.g., the cached login credentials that Credential Guard moves out of the Local Security Authority process) run at VTL1. The hypervisor then enforces two invariants on guest physical pages: a kernel page becomes executable only after the integrity check validates its signature, and a page is never simultaneously writable and executable (W^X, enforced via Second-Level Address Translation — the hardware page-table extension Intel calls EPT and AMD calls NPT). Even a fully compromised NT kernel cannot turn this off, because the rules are enforced from a layer the kernel cannot reach.

The asymmetry between user mode and kernel mode is what makes these measures load-bearing: a misbehaving signed *user-mode* binary crashes one process, while a misbehaving signed *kernel-mode* driver owns the whole machine. That is why kernel-mode signing requirements (Microsoft attestation, EV certs, Kernel-Mode Code Signing) are dramatically stricter than user-mode signing.

### Azure's build- and deploy-time gates

Microsoft's Azure code-integrity page describes the host-side production pipeline: every binary is signed with an Azure build certificate after Security Development Lifecycle validation; a final pre-deployment check called **Code Signature Validation (CSV)** verifies that every shipping binary matches the production code-integrity policy; deployment rolls out in stages (internal → first-party Microsoft → external customers); and any code-integrity block in production pages an on-call engineer. The same trust anchor that gates the kernel at boot also gates what can ever reach a host.

### dm-verity on Linux

The Linux equivalent is **dm-verity**, a kernel `device-mapper` target (a stackable block-layer plug-in that intercepts I/O between the filesystem and the underlying disk) providing transparent integrity verification of read-only block devices. Conceptually it is the data-at-rest analog to HVCI: HVCI guarantees that only authorized code pages execute; dm-verity guarantees that the files those pages are loaded from have not been altered.

Mechanics:

- The disk is divided into fixed-size data blocks (typically 4 KiB). Every block is hashed with SHA-256.
- Those hashes are arranged into a **Merkle tree** — a tree where every leaf is the hash of a data block and every interior node is the hash of its children's hashes. A single 32-byte **root hash** transitively covers the entire device. Any change to any block changes the root.
- On read, the kernel verifies the requested block against its leaf hash and walks up the tree. Verification failure at any level fails the I/O. Once a block is in the kernel's *page cache* (the in-RAM cache of recently-read disk pages) it is not re-verified.
- Where the root hash comes from is what makes dm-verity trustworthy: it is a parameter at device-mapper construction time, and the optional `root_hash_sig_key_desc` makes the kernel verify a signature on the root hash against keys in its trusted keyring before mounting the volume. The chain becomes **firmware → bootloader → kernel → trusted keyring → root hash → every block of `/`**.
- **Forward Error Correction (FEC)** is optional. It interleaves Reed–Solomon codes (an error-correcting code that can reconstruct a limited number of corrupted blocks from extra parity blocks) across the device so that bit-rot, bad sectors, or limited corruption can be reconstructed transparently. Reconstructed blocks are still validated against their hash before being returned, so FEC does not weaken the integrity guarantee — it only prevents valid data from being lost to media decay.
- Failure modes are configurable: log-only, fail the I/O with `EIO` (the standard "input/output error" return code; this is the default), reboot, or kernel panic.

dm-verity ships in production on **Chrome OS** (where it originated), **Android Verified Boot** (a public key on the boot partition validates the dm-verity table signature), and **Azure Linux** (formerly CBL-Mariner), where the image-config keys `ReadOnlyVerityRoot` and `RootHashSignatureEnable` build a verity-protected root filesystem (the verity hash data ships in the `initramfs`, the in-RAM filesystem the kernel uses before mounting the real root) and a `TmpfsOverlays` option provides RAM-backed writable paths over the read-only root. Azure Linux 3.0 (April 2025) extends this with **OS Guard**, an immutable container-host variant where `/usr` is a signed dm-verity volume.

The conceptual symmetry is the point: HVCI and dm-verity are the runtime continuation of the same chain Secure Boot started. The trust anchor at the firmware extends to every executable instruction the host runs and every block of `/` it reads, not just the boot transition.

## 5. Hashes: The Primitive Underneath Everything

PCR extends, dm-verity Merkle trees, capsule signatures, authenticated variables — every layer above has reduced to hashing at some point. A **cryptographic hash function** maps arbitrary-length input to a fixed-length digest, and three security properties are required for it to be useful in a trust chain:

1. **Preimage resistance** — given a digest, infeasible to find any input that hashes to it.
2. **Second-preimage resistance** — given an input, infeasible to find a different input with the same hash.
3. **Collision resistance** — infeasible to find any two distinct inputs with the same hash.

If collision resistance breaks, an attacker can craft a benign-looking artifact and a malicious twin with the same hash; a signature on one transfers to the other, and signed-but-not-what-you-think becomes possible. Hash strength bounds the strength of every signature, attestation, and Merkle proof above it.

### The SHA family

NIST publishes the SHA standards as FIPS (Federal Information Processing Standards) documents.

- **SHA-1 (1995)** — 160-bit output. **Practically broken in February 2017** by the **SHAttered** attack (CWI Amsterdam and Google), which produced two PDFs with the same SHA-1 digest at a cost of roughly 2^63.1 operations. Subsequent **chosen-prefix collisions** (Leurent and Peyrin, 2020) made SHA-1 even more dangerous, since they let an attacker pick the document content and forge a colliding twin. Deprecated for TLS and code signing on a phased schedule starting in 2016; now disallowed for digital-signature generation by NIST. Still appears in legacy PKI and (transitionally) Git object IDs.
- **SHA-2 family (2001)** — variants by output size: SHA-224, SHA-256, SHA-384, SHA-512, plus SHA-512/224 and SHA-512/256. **The current default**. NIST has no plans to deprecate SHA-2; SHA-256 is the workhorse.
- **SHA-3 (2015, FIPS 202)** — based on Keccak; selected via a 2007–2012 NIST competition. Uses a different internal construction (a "sponge," in which input is absorbed into a fixed state and output is squeezed back out, rather than the Merkle–Damgård chain that SHA-1 and SHA-2 use), so an attack on SHA-2 would not automatically carry over. NIST's position is that SHA-3 is a parallel option, not a forced migration. Adoption is still limited.

### Where hashes appear in this chain

Every link in the trust chain reduces to a hash:

- **TLS certificates** — the CA signs the SHA-256 digest of the certificate's contents.
- **UEFI Secure Boot signatures** — signature over a hash of the binary; SHA-256 is the default.
- **TPM PCR extension** — a PCR's update rule is `PCR_new = hash(PCR_old || measurement)`.
- **dm-verity** — Merkle-root hash over the entire root filesystem.
- **Capsule update signatures** — hash of the firmware payload.
- **Attestation quotes** (TPM, AMD SEV-SNP, Intel TDX) — signature over a hash of platform state plus a nonce (a one-time freshness value chosen by the verifier).
- **Authenticated UEFI variables** — hash of payload plus monotonic counter.

Collision resistance on the underlying hash function is the load-bearing assumption of every signature, every measurement, every revocation list.

## 6. The Hypervisor: What Sits on Top of the Trust Chain

By the time a hypervisor launches on an Azure node, the chain below it has already done substantial work: silicon RoT verified firmware, UEFI Secure Boot verified the bootloader, the host kernel was loaded under code-integrity policy with HVCI, and every binary on disk is covered either by signed allow-lists or by dm-verity's Merkle tree. The hypervisor sits **on top of** this trust foundation and extends it sideways into tenant isolation.

### What the Azure hypervisor is

The Azure hypervisor is **based on Hyper-V** and is **type-1** — it runs directly on hardware, not inside a host OS like a type-2 hypervisor (VMware Workstation, VirtualBox). It is **microkernel-style**: the hypervisor itself is small and does only CPU/memory virtualization and inter-partition messaging, while device drivers and most of the I/O stack live in a privileged partition outside it. This is in contrast to monolithic designs like classic Xen Dom0 or ESXi.

Hyper-V's partition model:

- **Root partition** — runs the host OS (a hardened Windows Server variant), has direct access to physical devices, and hosts the virtualization management stack.
- **Guest partitions** — host tenant VMs, have no direct hardware access, see only virtual devices.
- **VMBus** — a logical, channel-based inter-partition bus; the only sanctioned path for I/O from a guest to physical hardware.
- **VSP (Virtualization Service Provider)** in the root partition brokers real device I/O to **VSC (Virtualization Service Client)** instances in each guest.

### How the hypervisor enforces tenant isolation

Microsoft's hypervisor-security page lists four enforced boundaries: guest vs. host, guest vs. guest, hypervisor vs. host, and hypervisor vs. all guests. The mechanisms:

- **Memory partitioning** — separate guest physical address spaces enforced by Second-Level Address Translation (the same EPT/NPT hardware HVCI uses). The hypervisor controls the page tables; no guest can map another guest's memory.
- **vCPU scheduling** — the hypervisor multiplexes physical cores; virtual CPUs run under guest-mode constraints provided by Intel VMX or AMD SVM (the CPU instruction-set extensions for hardware virtualization).
- **I/O brokering** — all device I/O goes guest VSC → VMBus → root VSP → hardware (except SR-IOV pass-through, where Single-Root I/O Virtualization lets a single PCIe device expose multiple "virtual functions" directly to guests, configured explicitly).
- **Network isolation** — a per-host virtual switch with port ACLs and filters segments tenant traffic before it leaves the host.
- **Defense-in-depth code defenses** — ASLR (address-space layout randomization), DEP (data execution prevention, marking data pages non-executable), control-flow integrity, stack-variable initialization, kernel-heap zeroing, isolation of host-side processes that handle cross-VM components.

**Flag:** the page mentions "side-channel information leaks" generically but does not name Spectre, Meltdown, L1TF, MDS, Retbleed, or Downfall. Microsoft's mitigation guidance for these CVEs lives separately (KB4072698, advisories ADV180018 / ADV190013 / ADV220002), and many are mitigated at the firmware/microcode layer rather than in the hypervisor itself. Treat any blanket "Azure mitigates all transient-execution attacks" claim as overreach — mitigation completeness varies by CPU generation and microcode level.

### Where the hypervisor's trust ends — confidential computing

In the standard model, a tenant VM trusts both the hypervisor and the root partition. **Confidential computing** removes that trust:

- **AMD SEV-SNP** (Secure Encrypted Virtualization with Secure Nested Paging) — used by **DCasv5 / ECasv5** SKUs (5th-gen "v5" series, "C" for confidential, "as" for AMD SEV-SNP).
- **Intel TDX** (Trust Domain Extensions) — used by **DCesv5 / ECesv5** SKUs ("es" for Intel). The newer **DCesv6 / ECesv6** family on 5th Gen Intel Xeon entered general availability in early 2026.

Both encrypt VM memory in hardware and provide integrity protection so the hypervisor cannot read or tamper with VM state. Both move the hypervisor and host OS *out* of the confidential VM's trusted computing base. See [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/), [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), and [TDX Architecture]({{ site.baseurl }}/docs/core/tdx-architecture/) for the deeper treatment.

## 7. The Compute Taxonomy as Trust Boundaries

The hypervisor sets the lower edge of what tenants can avoid trusting; the compute service they pick sets the upper edge. Microsoft's compute decision tree reads as a managed-vs-unmanaged spectrum, but it can also be read as a **monotonically growing trust boundary**: the more managed the service, the more layers a customer is implicitly trusting Azure to run correctly.

### The current taxonomy

The decision tree (last refreshed 18 February 2026, well after Cloud Services classic was retired on 31 August 2024) lays out:

- **IaaS** — Azure Virtual Machines, Azure VMware Solution, Azure Dedicated Hosts. Customer manages the OS upward.
- **Managed containers / PaaS** — Azure Kubernetes Service (managed control plane), Azure Red Hat OpenShift, Azure Container Apps (Kubernetes hidden), Azure Container Instances (single container, no orchestration), App Service.
- **FaaS / serverless** — Azure Functions on Consumption, Flex Consumption, Premium, or App Service plans.
- **Specialty** — Azure Batch (HPC, parallel jobs), Service Fabric.

The flowchart's split is greenfield vs. brownfield (lift-and-shift vs. cloud-native), then container-exclusive vs. container-compatible at the bottom. **Confidential VM SKUs do not appear** in the current decision tree, which is itself worth noting — the trust-boundary question is not yet first-class in Microsoft's compute-selection framing.

### Trust grows monotonically

| Compute mode | Customer's trusted computing base includes |
|---|---|
| **Confidential VM** (SEV-SNP / TDX) | CPU + firmware + guest OS only. Hypervisor and host OS are *removed* from the TCB. |
| **Standard VM (IaaS)** | Hardware + firmware + Azure hypervisor + host OS + guest OS. |
| **AKS / Container Apps / Container Instances** | All of the above, plus the host OS and Kubernetes (or equivalent) control plane and container runtime. |
| **App Service** | All of the above, plus the App Service runtime, IIS, language workers. |
| **Azure Functions (Consumption / Flex)** | All of the above, plus the Functions host, trigger/binding system, and shared multi-tenant workers. |

The same hypervisor-rooted chain underlies every row. Confidential computing is the only mode that breaks the dependency on the hypervisor. Everything else is a question of how many layers of platform code the customer is willing to trust.

## 8. The Through-Line

Reading these eight documents together, the cross-document relationships become explicit:

- **UEFI defines the interface; Secure Boot defines the gate.** PK/KEK/db/dbx are the keys; authenticated variables are how those keys are themselves protected.
- **Project Cerberus protects the firmware that runs Secure Boot.** Without a hardware RoT below UEFI, a sufficiently privileged attacker can subvert the very logic that checks signatures.
- **Measured boot complements Secure Boot.** Secure Boot enforces, measured boot audits — and attestation is what makes the audit verifiable to a remote relying party.
- **HVCI and dm-verity extend the chain into runtime.** Once the kernel is up, code-integrity policy gates every code load and dm-verity gates every disk read, anchored back to the same firmware-rooted keys.
- **Hashes are the single primitive everything depends on.** Every signature, every PCR extend, every Merkle root, every revocation list — all are hash operations. SHA-256 is what makes the chain hold; SHA-1's break is why you cannot still rely on signatures over a SHA-1 digest.
- **The hypervisor inherits the chain.** It is launched only after the host below it has been verified, then extends isolation laterally between tenants.
- **Confidential computing breaks the chain on purpose.** It is the only Azure compute mode in which the customer does not have to trust Azure's hypervisor or host OS.
- **The shared-responsibility model is layered onto this trust chain.** Microsoft owns everything from the silicon RoT through the hypervisor (and, for PaaS/SaaS, further). The customer owns guest OS configuration, application code, identity, and data classification. See the [security model]({{ site.baseurl }}/docs/misc/microsoft-azure-security-model/) note for the full split.

## 9. Practical Takeaways

**Defaults Azure provides on every fleet host:**
- Secure Boot enabled at factory and tooled to stay enabled across the buildout pipeline.
- Project Cerberus on Azure motherboards as the hardware RoT.
- TPM 2.0 measured-boot logs feeding host attestation.
- Code-integrity policy gating Windows host kernel loads, validated at build time by Code Signature Validation, staged at deploy, paged on at runtime.

**Defaults customers should know about:**
- **Trusted Launch** for Generation-2 Azure VMs adds Secure Boot, a virtual TPM (vTPM), and boot-integrity monitoring inside the guest. Per-SKU and per-image — verify support before relying on it.
- **HVCI / Memory Integrity** is default-on (alongside VBS and Credential Guard) for clean installs of Windows 11 22H2 and later, and Windows Server 2025, on hardware that meets the requirements; older Windows Server SKUs (2019, 2022) require explicit enablement.
- **dm-verity** in Azure Linux is opt-in via `ReadOnlyVerityRoot`; combine with `RootHashSignatureEnable` so the root hash is bound to the kernel's trusted keyring. Azure Linux's OS Guard variant ships this on by default for the container host.

**When to break the chain on purpose:**
- Use **Confidential VMs** (DCasv5 / ECasv5 on AMD SEV-SNP, DCesv5 / ECesv5 or DCesv6 / ECesv6 on Intel TDX) when the threat model includes the hypervisor or Azure operators.
- Use **Confidential GPUs** (the `NCCadsH100v5` series, e.g., `Standard_NCC40ads_H100_v5`, NVIDIA H100 NVL on AMD SEV-SNP) when the same threat model applies to GPU workloads. See [GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/).

**On verifying the chain:**
- Attestation, not signature checking, is what proves the chain end-to-end. PCR quotes from TPM 2.0 (or attestation reports from SEV-SNP / TDX) give a relying party an auditable record of what ran. See the [hardware trust and key management]({{ site.baseurl }}/docs/misc/microsoft-azure-hardware-trust-and-keys/) note for the host-attestation, THIM, and Secure Key Release flow built on top of these PCRs, and [Artifact Map]({{ site.baseurl }}/docs/misc/microsoft-azure-hardware-trust-and-keys/#8-artifact-map) for a concise producer/consumer/verifier view of quote, collateral, token, and policy handoffs.

**On hash hygiene:**
- Treat any signature over a SHA-1 digest as untrusted. Use SHA-256 or stronger across TLS, code signing, and PCR banks.

## Sources

- [Secure boot — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/secure-boot)
- [Firmware security — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/firmware)
- [Platform code integrity — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/code-integrity)
- [dm-verity — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html)
- [Hypervisor security on the Azure fleet](https://learn.microsoft.com/en-us/azure/security/fundamentals/hypervisor)
- [Choose an Azure compute service](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/compute-decision-tree)
- [UEFI — Wikipedia](https://en.wikipedia.org/wiki/UEFI)
- [Secure Hash Algorithms — Wikipedia](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms)
- [UEFI Specification 2.11 (UEFI Forum, December 2024)](https://uefi.org/specs/UEFI/2.11/)
- [Project Cerberus — Open Compute Project](https://www.opencompute.org/projects/security)
- [Project Mu — Microsoft](https://microsoft.github.io/mu/)
- [SHAttered — first SHA-1 collision (CWI Amsterdam / Google, 2017)](https://shattered.io/)
- [BootHole (CVE-2020-10713) — Eclypsium write-up](https://eclypsium.com/blog/theres-a-hole-in-the-boot/)
- [App Control for Business and AppLocker overview](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview)
- [Trusted Launch for Azure VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/trusted-launch)
- [Azure Linux — read-only verity roots](https://github.com/microsoft/azurelinux/blob/3.0/toolkit/docs/security/read-only-roots.md)
- [Azure Linux with OS Guard (announcement, May 2025)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/azure-linux-with-os-guard-immutable-container-host-with-code-integrity-and-open-/4437473)
- [Microsoft guidance on the BlackLotus campaign (CVE-2022-21894)](https://www.microsoft.com/en-us/security/blog/2023/04/11/guidance-for-investigating-attacks-using-cve-2022-21894-the-blacklotus-campaign/)
- [LogoFAIL — Binarly research on UEFI image-parser flaws](https://www.binarly.io/blog/finding-logofail-the-dangers-of-image-parsing-during-system-boot)
- [Credential Guard / VBS default-on, Windows 11 22H2+](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/)
- [NCCadsH100v5 series — confidential GPU VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/gpu-accelerated/nccadsh100v5-series)
- [DC family confidential VM sizes (DCasv5, DCesv5, DCesv6)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dc-family)
