---
title: Microsoft Azure Security Architecture
parent: Reference
nav_order: 2
---

This note traces Azure's security story end-to-end as a single connected narrative. The four Parts are chapters of one argument:

1. **The platform earns trust from the bottom up** — silicon → firmware → boot → kernel → drivers → hypervisor. Each layer is verified by the layer below before being allowed to run.
2. **On top of that trust, Azure organizes physical infrastructure and gives customers tenant-isolation choices** — datacenters, networks, the Fabric Controller orchestrator, operator access; and the customer-facing isolation knobs from multi-tenant VMs through Confidential VMs.
3. **Once a host has cleared attestation, encryption and key management protect data** — host attestation, MAA, THIM, encryption at rest and in transit, double encryption, the five-way HSM decision, and Secure Key Release that gates key handover on a fresh attestation token.
4. **Finally, the shared-responsibility contract defines who owns what**, including the limits Microsoft itself accepts on its ability to reach into customer data (Customer Lockbox, NIST 800-88 hardware disposition, residency vs. sovereignty).

The five Parts are listed below; Section numbering runs continuously from 1 to 22 across them.

---

## Part I — The Platform Trust Chain

The platform earns trust from the bottom up. From the moment power reaches an Azure server, each layer is verified by the layer below before being allowed to run: silicon root of trust → firmware → bootloader → operating-system kernel → drivers and binaries → hypervisor → tenant workloads. Cryptographic hash functions are the primitive every link in that chain depends on.

## 1. The Silicon Layer

A **System-on-a-Chip (SoC)** integrates most or all components of a computing system — CPU cores, memory controller, I/O interfaces, accelerators, power management, and security blocks — onto a single die. This contrasts with the older multi-chip model where a CPU, memory controller, and southbridge (the I/O hub on PC motherboards) sat in separate packages connected by board-level buses. Modern server chips (AMD EPYC, Intel Xeon, Microsoft's Azure Cobalt 100) are SoCs in this sense.

The implication for trust: when the security processor — typically a Trusted Platform Module (TPM, the standardized chip that holds keys and measures boot state) or its equivalent — is co-integrated with the CPU on the same die, the **hardware root of trust (RoT)** is silicon physically next to the cores it protects, not a discrete chip on the motherboard talking over a bus that can be sniffed or replaced. This is what makes Project Cerberus on Azure motherboards (described in §2 below) and the on-die security blocks on AMD and Intel server SoCs cryptographically meaningful as anchors.

### Server SoCs in the Azure fleet

- **AMD EPYC** — integrates the Platform Security Processor (PSP), an ARM Cortex-A5 secure core. AMD's confidential-VM technology, **SEV-SNP** (Secure Encrypted Virtualization with Secure Nested Paging — encrypts and integrity-protects guest memory from the host), runs on Azure's **DCasv5 / ECasv5** SKUs (general-purpose / memory-optimized).
- **Intel Xeon Scalable** — integrates the Converged Security and Management Engine (CSME). It supports two confidential-computing technologies: **SGX** (Software Guard Extensions, which carves small encrypted "enclaves" inside a process) on DCsv2 / DCsv3, and **TDX** (Trust Domain Extensions, which encrypts entire VMs the way SEV-SNP does on AMD). The TDX general-availability family is **DCesv6 / ECesv6** (announced GA on 5th-Gen Xeon in West US and West US 3 and rolling outward); **DCesv5 / ECesv5** reached preview only.
- **Microsoft Cobalt 100** — Azure's first in-house ARM-based server SoC, 128 cores on Arm Neoverse N2. VMs went GA in October 2024 and are deployed across roughly 32 Azure regions and growing. A successor, Cobalt 200 (132 Neoverse-V3 cores on TSMC 3 nm), was announced in late 2025 with broader rollout in 2026.

The CVM SKU families above are picked up again in §9, where the trust chain reaches confidential computing.

### Caliptra: the open-source silicon RoT IP block

**Caliptra** is an open-source silicon root-of-trust IP block (not a chip; a reusable hardware design dropped into an SoC), founded in 2022 by AMD, Google, Microsoft, and NVIDIA, now a CHIPS Alliance project. Caliptra 2.0 added post-quantum primitives (NIST FIPS 204 ML-DSA signatures and ML-KEM key exchange via the Adams Bridge accelerator); Caliptra 2.1 is the latest RTL release. **Flag:** Microsoft has called Caliptra a "committed intercept" for first-party cloud silicon and AMD server silicon, but the first chips integrating it are only beginning to appear in 2026, so treat Caliptra as in-adoption rather than ubiquitous on Azure production hosts.

## 2. Hardware Roots of Trust

Secure Boot (covered in §4 below) can only be trusted if the firmware running it is itself trustworthy. Microsoft's answer is a hardware root of trust that sits *below* UEFI and gates access to firmware flash itself.

### The hardware-RoT concept

A **hardware root of trust (RoT)** is a trust anchor implemented in immutable silicon — its identity cannot be cloned, and it is implicitly trusted to verify everything above it. **NIST SP 800-193** ("Platform Firmware Resiliency Guidelines") sets the compliance bar: protection, detection, and recovery for platform firmware.

### Project Cerberus

**Project Cerberus** is Microsoft's open hardware RoT, contributed to the Open Compute Project in 2017 and now widespread on Azure motherboards. It is a cryptographic microcontroller running secure code that intercepts the SPI bus (the Serial Peripheral Interface bus connecting the main CPU to the flash chip where firmware is stored). Every flash read or write is measured and attested. Tampering is detected continuously, not just at boot, so firmware modification at runtime is also caught. Cerberus is CPU- and I/O-architecture-agnostic, supports a hierarchical RoT extending to peripherals (NIC, SSD, GPU, FPGA), and is NIST 800-193 compliant.

The key design move is **separating the firmware-integrity gate from the firmware itself**: the chip that decides whether firmware is allowed to run is not the chip running the firmware, so a compromised UEFI image cannot subvert its own verification.

### Microsoft Pluton

**Microsoft Pluton** is a related but different security processor, originally derived from Xbox and Azure Sphere. Where Cerberus is a discrete chip on the motherboard, Pluton is integrated directly into the System-on-Chip (SoC) — on AMD Ryzen 6000+, Intel Core Ultra, and Qualcomm Snapdragon X — and acts as both a hardware RoT and a TPM. It is default-on for Copilot+ PCs. **Flag:** Microsoft announced in 2020 that Pluton would also ship in Azure servers, but as of early 2026 there is no public evidence of large-scale Pluton-equipped Azure server SKUs. Treat that as not-yet-deployed.

### Anti-rollback

Modern silicon enforces **anti-rollback** through one-time-programmable fuses (Intel calls them Field Programmable Fuses, burned in the chipset's Platform Controller Hub at manufacturing) or monotonic counters. Each firmware image carries a Security Version Number (SVN); the platform refuses any image whose SVN is below the persistent floor. This blocks the "downgrade to a signed-but-vulnerable older version" attack — exactly the BlackLotus pattern (see §4.4), where signed-but-broken Windows boot managers existed for over a year before being revoked.

## 3. UEFI: The Firmware-to-OS Interface

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

## 4. Secure Boot: The First Cryptographic Gate

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

These are reminders that Secure Boot's guarantee is "the bootloader's signature is valid," not "the firmware that checks the signature is itself sound." Closing that gap is what §2 (hardware RoT) addresses.

## 5. Measured Boot and the TPM

**Measured boot** runs in parallel to Secure Boot. Each component computes a hash of the next component before executing it and *extends* the value into a **Platform Configuration Register (PCR)** inside a **TPM (Trusted Platform Module)**. A PCR is an append-only register: its value is updated as `PCR_new = hash(PCR_old || measurement)`, never overwritten without reboot. The result is a tamper-evident log of exactly what software ran from power-on onward.

**TPM 2.0** is the standard on Azure hardware. It supports configurable hash algorithms (SHA-256 and beyond, in per-algorithm PCR banks), where TPM 1.2 was hardcoded to SHA-1. It also splits authorization into four hierarchies: **Platform** (used by early-boot firmware), **Storage** (general user-key operations), **Endorsement** (privacy-sensitive identity keys), and **Null** (volatile, cleared on every reboot). Measurements feed two consumers: **remote attestation** (a TPM-signed quote of PCR values that a relying party verifies — the foundation of §12 below) and **sealing** (binding secrets so they unlock only when PCRs match expected values).

The contrast is worth holding onto: **Secure Boot says "this stage's signature is valid," measured boot says "and here is exactly what ran."** A relying party that only trusts a signature has a weaker statement than one that can audit the whole chain.

## 6. Code Integrity and dm-verity: The Chain into Runtime

Secure Boot and measured boot together cover power-on through kernel hand-off. After that, neither prevents a running kernel from later loading an unsigned driver, building an executable payload in writable memory, or reading a tampered binary off disk. **Code integrity** (on Windows hosts) and **dm-verity** (on Linux hosts) are how Azure extends the chain from boot into runtime.

### Code integrity on the Windows / Azure host

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

## 7. Hashes: The Primitive Underneath Everything

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

## 8. The Hypervisor

By the time a hypervisor launches on an Azure node, the chain below it has already done substantial work: silicon RoT verified firmware, UEFI Secure Boot verified the bootloader, the host kernel was loaded under code-integrity policy with HVCI, and every binary on disk is covered either by signed allow-lists or by dm-verity's Merkle tree. The hypervisor sits **on top of** this trust foundation and extends it sideways into tenant isolation.

### What the Azure hypervisor is

The Azure hypervisor is **based on Hyper-V** and is **type-1** — it runs directly on hardware, not inside a host OS like a type-2 hypervisor (VMware Workstation, VirtualBox). It is **microkernel-style**: the hypervisor itself is small and does only CPU/memory virtualization and inter-partition messaging, while device drivers and most of the I/O stack live in a privileged partition outside it. This is in contrast to monolithic designs like classic Xen Dom0 or ESXi.

### Three OS images per node

Each node hosts three kinds of operating-system image above the hypervisor:

- **Host OS** — a hardened Windows Server variant in the privileged root partition. It mediates all disk and network access on behalf of guest VMs and is not publicly accessible.
- **Native OS** — runs directly on storage tenants (e.g., Azure Storage), without a hypervisor.
- **Guest OS** — runs inside the customer's VM.

Microsoft's hypervisor-security doc states explicitly that *"machine boundaries are enforced by the hypervisor, which doesn't depend on the operating system security."* This is the load-bearing claim of Azure tenant isolation: if you trust the hypervisor and the host OS, you trust that one customer's VM cannot reach another's. The hypervisor itself only earns that trust because it is launched on top of the verified boot chain (§§2–7).

### Hyper-V partition model

- **Root partition** — runs the host OS, has direct access to physical devices, and hosts the virtualization management stack.
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

## 9. Where the Trust Chain Ends — Confidential VMs and GPUs

In the standard model, a tenant VM trusts both the hypervisor and the root partition. **Confidential computing** removes that trust through **AMD SEV-SNP** or **Intel TDX**: both encrypt VM memory in hardware and provide integrity protection so the hypervisor cannot read or tamper with VM state, moving the hypervisor and host OS *out* of the confidential VM's trusted computing base. See [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/), [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), and [Intel TDX]({{ site.baseurl }}/docs/core/tdx/) for the deeper treatment.

### Confidential VM SKU families

This note is the canonical home for the Azure confidential-VM SKU catalog:

- **DCasv5 / ECasv5** — AMD SEV-SNP, general-purpose / memory-optimized.
- **DCesv5 / ECesv5** — Intel TDX, preview only.
- **DCesv6 / ECesv6** — Intel TDX, the GA family on 5th-Gen Xeon (West US and West US 3 first, rolling outward).

### Confidential GPU SKU

The current Azure offering for confidential GPU workloads is the **`NCCadsH100v5`** series (e.g., `Standard_NCC40ads_H100_v5`) — NVIDIA H100 NVL paired with AMD SEV-SNP host VMs. See [GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/) for the GPU-side architecture.

### SKU naming convention

The `DC` / `EC` prefix marks the size family, `v5` / `v6` is the generation, the trailing letter `C` indicates confidential, `as` marks AMD SEV-SNP, `es` marks Intel TDX, and `NCC` marks confidential GPU. Memorizing the suffix codes is the fastest way to read a VM size string.

The compute trust taxonomy across all of Azure's compute services — IaaS, AKS, App Service, Functions — is in §11.3 below, where the "monotonically growing trusted computing base" framing is laid out as a table.

---

## Part II — Infrastructure and Tenant Isolation

What Azure builds on top of the trust chain: the operational layout, the people, and the choices customers have for sharing infrastructure.

## 10. The Infrastructure Stack

### Two networks, one company

Azure splits its operating environment into two physically and administratively separate networks:

- **Azure production network** — runs customer workloads. Operated by service teams (Azure Compute, Storage, SQL, Microsoft Entra ID, etc.).
- **Microsoft corpnet** — Microsoft's internal corporate IT network, used by employees for email, document collaboration, and the like. Operated by Microsoft IT.

The split is the outermost trust boundary: a compromise of Microsoft's corporate IT should not reach the production network where customer workloads live.

### Physical-to-logical hierarchy

Microsoft's infrastructure-components page describes the lower-level pieces but does not lay out the geographic hierarchy. From related Azure reliability docs:

- **Geography** — a data-residency boundary (e.g., "United States"). Contains one or more regions.
- **Region** — one or more datacenters connected by a high-capacity, low-latency network. Always inside one geography.
- **Availability Zone (AZ)** — a physically separated group of datacenters within a region, with independent power, cooling, and networking. Regions that support zones have at least three.
- **Datacenter** — a physical building managed by **MCIO** (Microsoft Cloud Infrastructure & Operations), the team responsible for all of Microsoft's physical datacenter facilities.
- **Cluster / scale unit / "stamp"** — within a datacenter, hardware grouped into a fault-isolation unit managed by a single Fabric Controller. Older Microsoft material describes this as roughly 1,000 servers per cluster. **Flag:** Microsoft's current public docs no longer publish a fixed cluster size; the "~1,000" figure should be read as illustrative, not contractual.
- **Node** — a single physical server (a "blade") in a cluster.

### Fabric Controller (FC): the per-cluster control plane

The FC is Azure's per-cluster orchestrator — a proprietary control plane analogous in spirit to a Kubernetes control plane, but lower-level and managing hardware in addition to VMs. Each FC owns one cluster and is itself replicated across an FC cluster of its own for fault tolerance. Its responsibilities include:

- VM lifecycle: provisioning, and recovery onto healthy nodes after a failure.
- Hardware health monitoring.
- Application deployment, upgrade, and scale-out.
- Maintaining a `datacenter.xml` inventory of every hardware and network device.

Beyond orchestration, the FC is also a cryptographic actor inside the platform itself:

- Inter-component traffic uses **TLS** (Transport Layer Security, the standard channel-encryption protocol used by HTTPS) with X.509 certificates, mostly self-signed. FC certificates and the externally facing certificates come from a Microsoft certificate authority chained to a trusted root, which enables key rollover.
- **Application images** submitted by developers are encrypted with the FC's public key in transit and at rest, protecting any embedded secrets.
- **Hardware credentials** (passwords and keys for network gear the FC controls) are encrypted with the **FC master identity public key** and decrypted only on demand. The design intent is that Microsoft developers, administrators, and backup personnel cannot read them.

That last point is worth pausing on: even Microsoft's own infrastructure uses key separation and encryption-at-rest internally. The same patterns recommended to customers (§§14–17 below) are applied to the platform itself.

### Operational access: just-in-time, not persistent

Layered on top of the technical boundaries are organizational ones. Azure separates several operational roles, each with bounded access:

- **MCIO** owns physical buildings, perimeter networking, and bare-metal setup. Customers never interact with them.
- **Service teams** own the engineering of each service and are on-call 24/7. **No physical hardware access by default.**
- **Datacenter engineers** handle physical security and hardware break-fix. **No customer data access.**
- **Incident triage, deployment, and live-site engineers** may touch customer data, but only via **just-in-time access** — the privilege is granted for a bounded window and audited; persistent access is limited to non-customer systems.

Operators connect from **Secure Admin Workstations (SAWs)**: dedicated, hardened machines with administrative accounts kept separate from regular user accounts. (Customers can mirror this pattern with **Privileged Access Workstations**, or PAWs. In Microsoft's own usage SAW and PAW are largely interchangeable, with PAW sometimes reserved for the highest-privilege tier.)

The technical isolation above keeps customers apart from each other; just-in-time access keeps Microsoft personnel from persistently sitting inside customer systems. **Customer Lockbox** (§20.3 below) is the customer-facing pre-approval workflow that gates this just-in-time access on customer consent.

## 11. Tenant Isolation Choices

The infrastructure above describes *what* the platform looks like. A separate question is how it keeps two customers from interfering with each other on the same shared hardware, and what choices the customer has for sharing or not sharing.

### The customer's grouping hierarchy

The customer's view of Azure has four levels of grouping:

- **Tenant** — a Microsoft Entra ID directory instance. The identity boundary. Two tenants cannot see each other's directories.
- **Subscription** — a billing and resource container associated with exactly one tenant. The primary blast-radius and RBAC scope for resources. One tenant can hold many subscriptions.
- **Management group** — optional grouping above subscriptions for applying policy and RBAC across many subscriptions at once. The hierarchy supports up to six levels of depth below the tenant root group, though Microsoft's own guidance is to keep it to three or four for manageability.
- **Resource group** — a namespace inside a subscription for grouping related resources.

Tenant is not the same as subscription. Tenant is *who can log in*; subscription is *where stuff lives and who pays*.

### The compute isolation spectrum

Five rungs at the IaaS level, weakest to strongest tenancy:

| Option | What it gives you | Threat addressed |
|---|---|---|
| Standard multi-tenant VM | Hyper-V hypervisor isolation; per-host packet filters | Untrusted neighbor tenant; baseline side-channel mitigations |
| **Isolated VM SKU** | A SKU sized to fill an entire physical host — guaranteed sole tenant on that machine | Compliance/regulatory single-tenancy without managing hosts |
| **Azure Dedicated Host** | You rent the whole physical server, place your own VMs on it, and control maintenance windows | Single-tenancy plus placement and patch-window control |
| **Confidential VM** (AMD SEV-SNP, Intel TDX) | Hardware-encrypted VM memory; the host and hypervisor cannot read VM memory; remote attestation | Malicious cloud operator; compromised hypervisor |
| **Confidential VM on Dedicated Host** | All of the above stacked | Both neighbor-tenant and operator threats |

A key conceptual split: **isolated and dedicated options protect against neighbor tenants. Confidential VMs protect against Azure itself.** They address orthogonal threats and can be combined. The specific SKU families are catalogued in §9 above.

### Trust grows monotonically across compute services

Microsoft's compute decision tree (last refreshed 18 February 2026, well after Cloud Services classic was retired on 31 August 2024) reads as a managed-vs-unmanaged spectrum, but it can also be read as a **monotonically growing trust boundary**: the more managed the service, the more layers a customer is implicitly trusting Azure to run correctly.

The current taxonomy, for reference:

- **IaaS** — Azure Virtual Machines, Azure VMware Solution, Azure Dedicated Hosts. Customer manages the OS upward.
- **Managed containers / PaaS** — Azure Kubernetes Service (managed control plane), Azure Red Hat OpenShift, Azure Container Apps (Kubernetes hidden), Azure Container Instances (single container, no orchestration), App Service.
- **FaaS / serverless** — Azure Functions on Consumption, Flex Consumption, Premium, or App Service plans.
- **Specialty** — Azure Batch (HPC, parallel jobs), Service Fabric.

The flowchart's split is greenfield vs. brownfield (lift-and-shift vs. cloud-native), then container-exclusive vs. container-compatible at the bottom. **Confidential VM SKUs do not appear** in the current decision tree, which is itself worth noting — the trust-boundary question is not yet first-class in Microsoft's compute-selection framing.

| Compute mode | Customer's trusted computing base includes |
|---|---|
| **Confidential VM** (SEV-SNP / TDX) | CPU + firmware + guest OS only. Hypervisor and host OS are *removed* from the TCB. |
| **Standard VM (IaaS)** | Hardware + firmware + Azure hypervisor + host OS + guest OS. |
| **AKS / Container Apps / Container Instances** | All of the above, plus the host OS and Kubernetes (or equivalent) control plane and container runtime. |
| **App Service** | All of the above, plus the App Service runtime, IIS, language workers. |
| **Azure Functions (Consumption / Flex)** | All of the above, plus the Functions host, trigger/binding system, and shared multi-tenant workers. |

The same hypervisor-rooted chain underlies every row. Confidential computing is the only mode that breaks the dependency on the hypervisor. Everything else is a question of how many layers of platform code the customer is willing to trust.

### Network isolation

- **Virtual Network (VNet)** — the traffic-isolation boundary. VMs in different VNets cannot talk directly, even within one customer or subscription. Subnets subdivide a VNet.
- **NSG (Network Security Group)** — stateful 5-tuple allow/deny rules attached to subnets or NICs. Does not inspect application-layer content.
- **Service Endpoint vs. Private Endpoint** — both let you reach PaaS services from your VNet, but they differ. A *service endpoint* extends your VNet identity to a multi-tenant PaaS service over Azure's backbone; the service still has a public IP. A *private endpoint* injects a private network interface for the service into your VNet; the service gets an IP from your address space and traffic never leaves the VNet. Private endpoints are stronger; service endpoints are simpler.

### Per-service isolation: Storage, SQL, App Service, AKS

- **Storage** runs on physically separate hardware from compute, with virtual disks that are sparse — physical blocks are allocated only on first write, preventing data remanence from a prior tenant.
- **Azure SQL Database** is a multi-tenant PaaS with logical isolation. A "logical SQL server" is a metadata grouping, not a VM; databases under one logical server can land on different physical instances. A stateless gateway tier proxies connections (over the SQL Server Tabular Data Stream, or TDS, application-layer protocol) to a backend tier of replicated database instances, which validates logins, applies firewall rules, and routes the request to the physical machine that holds the database.
- **App Service** runs each app in a Windows sandbox per app, rooted at an IIS worker process running under a low-privilege, randomly-generated app-pool identity. Apps share VMs with other tenants but cannot read each other's memory. **App Service Environment (ASE) v3** is a single-tenant deployment of the entire App Service stack into your VNet; use it for regulated workloads or when default network egress controls are insufficient.
- **AKS (Azure Kubernetes Service)** layers isolation through Namespaces (RBAC and quota scope), Network Policies (default-deny pod-to-pod traffic), Pod Security, and separate Node Pools per tenant. For hard isolation, use separate clusters or confidential node pools.

### Decision criteria

- **Subscription-level isolation is enough** when your concern is RBAC blast radius, billing separation, or policy scoping — not infrastructure neighbors.
- **Choose Dedicated Host** when you need single-tenant hardware *plus* control over VM placement and patch windows.
- **Choose an Isolated VM SKU** when you need single-tenancy on one large VM without managing hosts — but accept that the SKU has a hardware-deprecation lifecycle.
- **Choose Confidential VM** when the threat model includes Microsoft itself or a compromised hypervisor.
- **Choose ASE over multi-tenant App Service** when you need VNet-native deployment or hardware-level isolation for web workloads.
- **Choose Private Endpoint over Service Endpoint** when traffic must traverse on-premises links or must never appear on a public IP.

---

## Part III — Encryption and Key Management

How data is protected once a host has cleared attestation. Each encryption guarantee Microsoft makes ultimately rests on the trust chain in Part I — the trust chain is what makes the guarantee credible.

## 12. Host Attestation: Fleet-Internal Verification

**Host attestation** is the fleet-internal check that a physical Azure host is in a known-good state *before* it is allowed to receive customer workloads, accept fleet credentials, or talk to the rest of the control plane. It is distinct from **guest (or workload) attestation**, where a workload running inside a Confidential VM proves the state of *its* **trusted execution environment** (TEE — a CPU-enforced isolated runtime, e.g. SGX, SEV-SNP, or TDX) to a relying party. Host attestation is invisible to customers; guest attestation is what they consume.

### What gets measured and what happens on failure

The host attestation service runs inside each Azure cluster in a "specialized locked-down environment" and validates a compliance statement from each host against a Microsoft-defined attestation policy. The Microsoft Learn page enumerates the specific items checked:

- **Secure Boot state** — that UEFI Secure Boot is enabled, plus the contents of the four UEFI key databases: the signature database (db, allowed signers), the revoked-signatures database (dbx), the Key Exchange Key database (KEK, in this UEFI sense — *not* the encryption-key wrapper introduced in §14 below), and the Platform Key (PK, the top-level UEFI signing key).
- **Debug controls** — production hosts must boot with kernel debuggers disabled.
- **Code-integrity policy** — which kernel-mode drivers and binaries are signed and trusted.
- Implicit in the PCR set: firmware, bootloader, hypervisor, and kernel measurements (per §5).

If any item fails (TCG event-log mismatch, Secure Boot off, debugger enabled, code-integrity violation), the cluster blocks all communication to and from the host and triggers an incident workflow. The host stays isolated until a post-mortem clears it.

### The TPM key hierarchy used in attestation

Three TPM keys do the work in any attestation flow, host or guest:

- **Endorsement Key (EK)** — an asymmetric key burned into the TPM at manufacture, certified by an EK certificate from the TPM vendor. It uniquely identifies one specific TPM. The EK is a key-exchange / decryption key, not a signing key, and is privacy-sensitive (using it directly correlates a machine across requests).
- **Attestation Key (AK)** — formerly called the Attestation Identity Key (AIK). A restricted signing key derived inside the TPM and bound to the EK via a credential-activation protocol. The AK is what actually signs PCR **quotes** (defined next). Multiple AKs can exist as privacy-preserving aliases of one EK.
- **Quote** — a TPM-signed structure containing the requested PCR digests and a caller-supplied **nonce** (a one-time random number used as a freshness challenge that defeats replay attacks), signed by the AK. Verifying a quote tells the relying party that (a) the measurements came from a real TPM, (b) they are bound to that TPM's EK chain, and (c) they are fresh.

### End-to-end flow

1. The host boots; PCRs accumulate measurements through firmware, bootloader, kernel, drivers, and hypervisor.
2. A **host attestation agent** in the Hyper-V root partition (§8 — Microsoft-controlled, not customer-visible) collects the TCG event log and asks the TPM for an AK-signed quote over the relevant PCRs, with a nonce supplied by the cluster's attestation service.
3. The cluster's host attestation service verifies the quote signature against the host's enrolled AK and EK chain, replays the event log against the PCRs, and compares the result to the cluster's policy.
4. On success, the cluster's PKI issues credentials sealed to the host's PCRs — only that host can unseal them, so a clone or man-in-the-middle cannot reuse them.
5. The host is now allowed to take customer workloads.

### Microsoft Azure Attestation (MAA): the customer-facing service

Customers see a different surface called **Microsoft Azure Attestation (MAA)**. MAA is generally available, with a regional shared provider in every Azure region where confidential computing is offered. It verifies TEE evidence from SGX enclaves, AMD SEV-SNP confidential VMs, Intel TDX confidential VMs, VBS enclaves (Windows Virtualization-Based Security in-process enclaves; see §6.1), and TPM-based platforms, and issues a signed **JSON Web Token (JWT — the JSON-based signed-token format from RFC 7519)**. The token's signing keys are published at an **OpenID Connect (OIDC)** / **JSON Web Key Set (JWKS)** metadata endpoint — the same standards OAuth identity providers use; relying parties fetch the certificate by its key ID (`kid`), verify the signature, and then evaluate the claims. Standard claims include `x-ms-attestation-type`, `x-ms-policy-hash`, platform-specific claims (`x-ms-sevsnpvm-*`, `x-ms-isolation-tee.*`), and `rp_data` — a relying-party-supplied nonce for freshness.

A **relying party** is whichever component consumes the MAA token to make a trust decision: typically Azure Key Vault (deciding whether to release a key, see §17), an application gating a secret, Azure Policy evaluating compliance, or Microsoft Defender for Cloud verifying posture.

**Flag:** older docs reference services like the "TVM Service" or "Host Verification Service." Treat these as superseded by the modern MAA + host-attestation split. Likewise, EPID-based SGX attestation (Intel's older Enhanced Privacy ID scheme) on legacy DCsv2 SKUs is being phased out industry-wide in favor of ECDSA-based attestation through Intel's **Data Center Attestation Primitives (DCAP)** library and THIM (next section).

## 13. Trusted Hardware Identity Management (THIM)

MAA can only verify a quote if it has the right **vendor-issued attestation collateral** — certificates, revocation lists, and **Trusted Computing Base (TCB)** information (the inventory of CPU microcode, firmware, and TEE-module versions a platform must be at to be considered current) published by Intel and AMD. **Trusted Hardware Identity Management (THIM)** is the Azure-internal service that caches and serves that collateral. THIM is not the attester; it is the upstream collateral provider that attesters and verifiers call into.

### What collateral actually means

For Intel SGX and TDX, THIM serves what Intel calls Quote Verification Collateral (QVC):

- **PCK certificate (Provisioning Certification Key)** — a per-platform X.509 certificate. On 4th-Gen Intel Xeon Scalable and later, Azure performs "indirect registration" with Intel using a Platform Manifest (a structured description of the platform's keys), and the resulting PCK certificate lives only inside THIM. Customer VMs cannot reach Intel's Provisioning Certification Service directly because they lack the Platform Manifest — they have to go through THIM.
- **TCB Info** — JSON describing the current TCB level the platform should be at: CPU microcode **SVN** (Security Version Number — a monotonic counter Intel and AMD bump on each security fix), platform engine SVNs, and TDX module versions where applicable.
- **QE Identity** — the identity of the Quoting Enclave (an Intel architectural enclave that signs quotes); required to verify that a quote came from a legitimate enclave.
- **CRLs** (Certificate Revocation Lists) from the SGX Platform CA and Processor CA.

For AMD SEV-SNP, THIM caches:

- **VCEK certificate (Versioned Chip Endorsement Key)** — a per-chip, per-TCB X.509 certificate that binds the attestation key to a specific physical CPU and a TCB version.
- **Certificate chain** — the AMD SEV Key (ASK) and AMD Root Key (ARK, AMD's self-signed root). VCEK is signed by ASK, ASK by ARK.

### Why THIM exists

Three reasons, only the first of which Microsoft states explicitly:

1. **Reduce external dependency on Intel and AMD endpoints.** Each Azure region serves cached collateral from `global.acccache.azure.net` and a per-host **Instance Metadata Service (IMDS)** path — `http://169.254.169.254/metadata/THIM/...`, the link-local endpoint every Azure VM uses to query its own metadata — so a transient outage at Intel's Provisioning Certification Service (PCS) or AMD's Key Distribution Service (KDS) doesn't cascade into Azure attestation failures.
2. **Indirect registration on modern Xeons.** Starting with 4th-Gen Intel Xeon Scalable, Intel does not store root keys for the platform (`CachedKeys=false`); ongoing communication requires the Platform Manifest, which Azure does not pass to guest VMs. So PCK certificates are *only* obtainable from THIM on those CPUs.
3. **Stability over freshness — a deliberate design choice.** The Microsoft page states it directly: "Trusted Hardware Identity Management takes a slower approach to updating the TCB baseline." THIM intentionally serves an older `tcbinfo` than Intel's current one so that customers who haven't updated SDKs or microcode don't see surprise attestation failures.

### TCB recovery: the dynamic piece

Attestation verdicts are not static. When Intel issues a **TCB recovery (TCB-R)** — its term for raising the minimum acceptable TCB after a vulnerability disclosure — Intel publishes new PCK certificates and new TCB Info, and verifiers using fresh collateral start returning `OUT_OF_DATE` for unpatched platforms. AMD does the same for SEV-SNP via VCEK reissuance bound to a new `tcbm` (TCB measurement) value. The customer-visible consequence: a quote that passes against THIM's baseline today may fail against Intel's stricter baseline (or against THIM after Azure rolls its baseline forward later). Customers running confidential workloads with attestation-gated key release should monitor TCB-R guidance and have a plan to re-attest.

### Who consumes THIM

- **Microsoft Azure Attestation (MAA)** — the canonical consumer; MAA's verdict is "does this quote meet THIM's baseline."
- **Intel's Quote Provider Library (QPL)** — Intel's reference library for fetching collateral when verifying SGX/TDX quotes; works on Azure if reconfigured to point `pccs_url` and `collateral_service` at the `global.acccache.azure.net` endpoint. ("PCCS" — Provisioning Certificate Caching Service — is Intel's name for any caching layer in this role; THIM is Azure's.)
- **Azure DCAP client (`az-dcap-client`)** — Microsoft's drop-in replacement for QPL, pre-wired to THIM. Required for legacy DCsv3 SGX scenarios on supported Ubuntu and Windows.
- **AKS Confidential Containers (the Azure Kubernetes Service variant that runs pods inside TEEs) and confidential-VM nodes** — reach THIM via the per-host IMDS path.
- **Azure Key Vault Premium and Managed HSM Secure Key Release (SKR)** — transitive consumer; SKR validates an MAA token, and MAA in turn relied on THIM to validate the underlying quote. See §17.

**Flag:** the documented endpoint is global (`global.acccache.azure.net`); per-region URLs are not published, and the docs do not describe regional failover behavior beyond the in-host IMDS cache. Treat THIM as a global anycast surface backed by a per-host cache.

## 14. Encryption Layered on the Trusted Foundation

Once a host has cleared host attestation and is running with valid platform identity, the next layer up is the encryption applied to data on that host. Microsoft's three encryption pages (`encryption-overview`, `encryption-atrest`, `storage-service-encryption`) describe a uniform model: every Azure **resource provider** (the API surface that implements a given service — Storage, SQL, Cosmos DB, etc.) applies encryption at rest the same way, and the customer's choice is mostly *who controls the key*.

### Three states of data

Encryption protects data in three states:

| State | Meaning | Typical defenses |
|---|---|---|
| **At rest** | Static data on physical media | AES-256 at the storage cluster |
| **In transit** | Data moving over a network | TLS, IPsec VPN, MACsec at the link layer |
| **In use** | Data being processed in memory | Confidential VMs (AMD SEV-SNP, Intel TDX, Intel SGX) — addressed in §9 above and in [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/) |

The encryption docs cover at-rest and in-transit thoroughly, but treat in-use only conceptually — pointing at hardware-managed keys without naming the technologies. The rest of this repo (see [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/), [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), [Intel TDX]({{ site.baseurl }}/docs/core/tdx/), [SGX]({{ site.baseurl }}/docs/core/sgx/)) goes deeper on data in use.

### Envelope encryption: the universal pattern

Azure uses **envelope encryption** almost everywhere — a two-tier scheme where one key encrypts data and a second key encrypts the first key. The two key types:

- **Data Encryption Key (DEK)** — a symmetric AES-256 key that encrypts a partition or block of data. A single resource has many DEKs; per-block separation limits the blast radius of a compromised key.
- **Key Encryption Key (KEK)** — wraps (encrypts) the DEKs. Lives in Azure Key Vault or Managed HSM and ideally never leaves it. (In the encryption literature this is sometimes also called the **Customer Master Key (CMK)** when the customer owns it.)

Envelope encryption is used for performance — bulk crypto needs DEKs local to the service, but a Key Vault round-trip per I/O operation would be prohibitive. The KEK gives a single revocation point: disabling the KEK cryptographically erases all dependent DEKs, since a wrapped DEK cannot be unwrapped. Wrapped DEKs are persisted as metadata alongside the encrypted data.

A subtle but important detail: when a service caches a DEK in memory for active operations, those cached keys are protected by host-level platform isolation. **Key Vault is the root of trust for cold keys, but warm DEKs live in host memory** — and the integrity of those warm DEKs reduces back to the SoC root of trust (§§1–2), measured boot (§5), host attestation (§12), and (for confidential workloads) TEE-enforced isolation (§9). This is the explicit handoff point between the trust chain and the encryption stack.

### What's always on at rest, no customer action

| Service | Default behavior | Algorithm |
|---|---|---|
| Azure Storage (Blob, File, Queue, Table) — Storage Service Encryption (SSE) | On, cannot be disabled | AES-256 in **GCM** (Galois/Counter Mode — an authenticated mode that produces both ciphertext and an integrity tag) |
| Managed disks, snapshots, images | Server-side encryption with platform-managed keys | AES-256 (mode unspecified in current Microsoft docs — flag) |
| Azure SQL Database (created since June 2017) | **Transparent Data Encryption (TDE — encrypts the database files; transparent to applications)** on by default | AES-256 |
| Azure Cosmos DB (non-volatile storage) | Service-managed encryption, default | AES-256 |
| Temp / ephemeral OS disks on Vv5+ VMs | Auto-encrypted via encryption at host | AES-256 |

Worth flagging: **temp disks on pre-v5 VM SKUs are not encrypted unless encryption at host is on**. This is a real gap many customers miss. **Azure Disk Encryption (ADE)** — the older guest-side option that uses dm-crypt on Linux and BitLocker on Windows — is **scheduled for retirement on September 15, 2028**; after that date, ADE-encrypted VMs will fail to unlock on reboot, so ADE workloads need to migrate to encryption at host before then.

### Disks: three mechanisms often confused

- *Server-side encryption (SSE for disks)* — always on at the storage cluster.
- *Encryption at host* — the VM's host server encrypts data before it leaves to storage, covering temp disks and VM caches that storage-side SSE alone misses. Microsoft positions this as the recommended option for new VMs. Note that "encryption at host" crosses the **hypervisor boundary** (§8) — encryption happens on the host before data leaves for storage.
- *Azure Disk Encryption (ADE)* — older, runs inside the guest using BitLocker (Windows) or dm-crypt (Linux). Retiring 2028-09-15.

### Customer-configurable knobs

- **Customer-managed keys (CMK).** The KEK lives in Azure Key Vault Premium or Managed HSM. The customer holds it; the service generates and keeps the DEK. Most managed-disk and SQL CMK uses RSA 2048, 3072, or 4096 bits; Synapse and SQL TDE want RSA 3072 specifically; Data Lake Store is pinned to RSA 2048.
- **Customer-provided keys (CPK).** Specific to Azure Blob Storage. The customer sends the encryption key on each REST request; Azure never persists it. A SHA-256 hash of the key is persisted alongside the blob so subsequent requests can verify the same key is supplied. **Side effect:** Microsoft Defender for Storage cannot scan CPK-encrypted blobs, since it has no key.
- **Encryption scopes** (Azure Storage, generally available). A scope is a key boundary smaller than a storage account: a default scope per container, with per-blob override. Each scope independently chooses Microsoft-managed or customer-managed keys, so a single storage account hosting multiple tenants can keep one tenant's blobs unreadable to another by giving each tenant its own scope and KEK.
- **Confidential disk on a Confidential VM.** Disk encryption keys are bound to the VM's **virtual TPM (vTPM — a TPM exposed to the guest by the hypervisor)**, and keys are released to the VM only after platform attestation succeeds. Confidential OS-disk encryption is supported only for OS disks below 128 GB (a real constraint for lift-and-shift migrations of larger system disks). This is the path that closes the loop with §17 (SKR).
- **Encryption at host.** Encrypts temp disks and OS / data disk caches *on the VM host* before data flows to Storage, using no guest CPU. Replaces ADE and is Microsoft's recommended option for new VMs.

### Database — Transparent Data Encryption (TDE)

Default for Azure SQL Database, Managed Instance, and Synapse. Encrypts page-level database files using a database encryption key (DEK), which is itself wrapped by a TDE protector. With CMK, the protector is a 2,048- or 3,072-bit RSA key in Key Vault or Managed HSM (4,096-bit is not supported for TDE).

### In transit

- **HTTPS** for all portal traffic and Storage REST API calls.
- **Site-to-site VPN** — connects an on-prem network to an Azure virtual network using IPsec.
- **Point-to-site VPN** — connects a single workstation to a virtual network.
- **ExpressRoute** — a dedicated private circuit between an on-prem datacenter and Azure. Notably, ExpressRoute traffic **is not encrypted by default**; MACsec is available on ExpressRoute Direct as an opt-in customer-configured feature, and the encryption-best-practices doc recommends layering TLS or IPsec on top.

The doc recommends TLS but does not pin a minimum version. Azure Storage will require TLS 1.2 or higher starting February 3, 2026 (TLS 1.0 and 1.1 are being retired); TLS 1.3 is also supported, though it cannot yet be enforced as the minimum for Storage accounts.

### Algorithm and FIPS status

- **Azure Storage server-side encryption** uses AES-256-GCM (explicitly stated by Microsoft).
- **Storage client-side encryption v1** used AES in **CBC (Cipher Block Chaining)** mode and is deprecated due to an implementation vulnerability in that mode; v2 uses AES-GCM.
- **Managed disk SSE** is documented only as "256-bit AES, FIPS 140-2 compliant"; the mode is not stated by Microsoft. Don't assume **XTS** (the tweakable AES mode standard for full-disk encryption, used by BitLocker) just because BitLocker uses it.
- **RSA-OAEP-256 is the recommended wrapping algorithm.** Plain RSA-OAEP defaults to SHA-1 for its hash and mask-generation functions and is treated as legacy because of known weaknesses in SHA-1 (§7.2).
- **FIPS 140-2 (the older NIST cryptographic-module standard) moves to NIST's "historical" list on September 21, 2026.** Modules already validated to 140-2 may keep operating, but new validations after that date must be FIPS 140-3 — the successor standard. (HSM-specific FIPS levels per Azure offering are tabulated in §16.)

### Key rotation: a subtlety

Rotating a **key encryption key (KEK)** **does not re-encrypt the underlying data**. It re-wraps the DEKs with the new KEK version. Both the old and new KEK versions must remain enabled until the re-wrap completes.

For suspected key compromise, **order matters**:

1. Rotate to a new key.
2. Reconfigure dependent services to use the new key.
3. *Then* disable or delete the old key.

Disabling first breaks the dependent services and leaves DEKs still wrapped by the compromised key — so an attacker holding the old key material can still decrypt.

### BYOK vs CMK vs PMK

Two related but distinct concepts:

- **BYOK (Bring Your Own Key)** — the customer generates the key in their own on-prem HSM and imports it into Azure. Describes *how the key got there*.
- **Platform-managed key (PMK) vs Customer-managed key (CMK)** — describes *who controls the key in production*. PMKs are created, rotated, and used transparently by Microsoft. CMKs live in the customer's Key Vault and **wrap** (encrypt) the service's data encryption key (DEK); revoking a CMK effectively destroys access to the data because the DEK can no longer be unwrapped.

## 15. Double Encryption: When One Layer Is Not Enough

Standard at-rest and in-transit encryption defends against the obvious threats: a stolen disk, a wiretapped fiber, a misplaced credential. **Double encryption** adds a second independent layer, defending against three sharper failure modes:

- **Configuration error** in one encryption layer.
- **Implementation bug** in a single algorithm.
- **Compromise** of a single key.

Independence between the two layers is the load-bearing requirement: **separate keys, separate key hierarchies, separate operators, ideally different algorithms**. If both layers shared a key or algorithm, a single break would unwind both, and the protection would be cosmetic.

### At rest: two layers

| Layer | What it is | Key holder |
|---|---|---|
| **Service-level** | Encryption at rest using a CMK in Key Vault (customer's choice of BYOK or generated). | Customer |
| **Infrastructure-level** | A second, lower layer using a PMK with a *different* algorithm and a *separate* Microsoft-managed key. Always on by default for services that support it. | Microsoft |

For Azure Storage specifically, the infrastructure-encryption toggle **must be enabled at storage account creation** — it cannot be turned on later. This is a forward-looking decision, not a recoverable one. The infrastructure layer is always Microsoft-managed (a customer cannot CMK the bottom layer), and Microsoft asserts the two layers use different algorithms, but the second layer's specifics are not published.

### In transit: TLS plus MACsec

| Layer | OSI level | Mechanism | Scope |
|---|---|---|---|
| **Application/transport** | L4–L7 | TLS 1.2 (default) | Between Azure services and customers; also between Azure services |
| **Link** | L2 | MACsec (IEEE 802.1AE) | Between Azure datacenters, point-to-point on physical network hardware |

**MACsec** — IEEE 802.1AE — is link-layer encryption many readers will not have encountered:

- Encrypts Ethernet frame payloads using AES-GCM (128- or 256-bit keys).
- Adds an integrity check value (ICV) and replay protection.
- **Hop-by-hop, not end-to-end**: each network hop decrypts on ingress and re-encrypts on egress, so frames are briefly in cleartext inside a network device.
- Implemented in network hardware (typically the switch or router silicon), so Microsoft claims line-rate encryption with no measurable latency increase.
- Defends specifically against physical wiretap or man-in-the-middle attacks on cables between datacenters.
- Always on for Microsoft's own inter-datacenter backbone — no customer action required.

Why two layers: TLS protects an application stream end-to-end across the public path but terminates at endpoints. MACsec protects the physical wire even where TLS sessions don't exist (e.g., infrastructure-to-infrastructure traffic) and acts as a backstop if TLS is misconfigured or downgraded.

### When double encryption makes sense

- Regulated workloads — finance, healthcare, government — where defense-in-depth against cryptographic failure is mandated.
- Ultra-sensitive data where the cost of a single-algorithm break is catastrophic.
- Threat models that include sophisticated adversaries with possible 0-day exploits in a cipher implementation.

For most workloads it is overkill. Single-layer AES-256 is not the weak link in real breaches — credentials, misconfiguration, and access control are. The cost of double encryption is mostly operational: with CMK you take on Key Vault rotation, access-policy management, HSM-backed keys, and the risk that losing the key destroys the data.

## 16. Choosing a Key Manager: The Five-Way Decision

Once the CMK pattern is in place, the next question is *where the customer-held KEK actually lives*. Microsoft's current decision page lists five options. The previously prominent Azure Dedicated HSM is no longer in the comparison table — the IaaS-style HSM role is now filled by **Azure Cloud HSM** (general availability in 2025, also on Marvell LiquidSecurity), and Microsoft routes on-prem-HSM migrations there.

| Option | Tenancy | FIPS level | Primary role |
|---|---|---|---|
| **Key Vault Standard** | Multi-tenant PaaS | FIPS 140-2 Level 1 (software) | Cheap software vault for keys, secrets, certificates |
| **Key Vault Premium** | Shared HSM, PaaS | FIPS 140-3 Level 3 (Platform 2 keys); FIPS 140-2 Level 2 (legacy Platform 1) | HSM-backed keys without a dedicated HSM cost |
| **Azure Key Vault Managed HSM** | Single-tenant HSM pool, PaaS | FIPS 140-3 Level 3, PCI DSS, PCI 3DS | Customer-controlled HSM with key sovereignty; **keys only**, no secrets or certificates |
| **Azure Cloud HSM** | Single-tenant, IaaS-style | FIPS 140-3 Level 3 | Full **PKCS#11** / **JCA** / **CNG** support (the standard cryptographic APIs used by C, Java, and Windows clients respectively); on-prem HSM migration, Oracle TDE, code signing |
| **Azure Payment HSM** | Single-tenant, IaaS-style | FIPS 140-2 Level 3, PCI HSM v3, PCI PTS HSM v3 | Payments only — PIN block, EMV, 3DS, card auth |

An **HSM** — hardware security module — is a tamper-resistant device that performs cryptographic operations and never lets the private key material leave the device in plaintext. **FIPS 140** is the U.S. government standard that certifies how robustly an HSM resists tampering; Levels 2 and 3 differ mainly in physical-tamper response (Level 3 zeroizes keys when intrusion is detected). Premium, Managed HSM, and Cloud HSM all run on **Marvell LiquidSecurity** HSMs (LiquidSecurity 1 and 2; the LS2 family received its FIPS 140-3 Level 3 validation in June 2024). Payment HSM runs on a separate model certified to PCI HSM v3.

### The decision criteria, in priority order

1. **Compliance level required.** FIPS 140-2 Level 1 → Standard. FIPS 140-3 Level 3 → Premium, Managed HSM, or Cloud HSM. Payments → Payment HSM only.
2. **Key sovereignty** (does the customer need to be the only party that can ever access the key material?). Yes → Managed HSM, Cloud HSM, or Payment HSM. No → Standard or Premium.
3. **Single-tenancy required.** Same answer as sovereignty.
4. **Use case.** Generic Azure PaaS / CMK → Premium or Managed HSM. TLS offload, Oracle TDE in IaaS, code signing → Cloud HSM (the only option with full native PKCS#11 / JCA / JCE / CNG / KSP). Payment PIN processing → Payment HSM only.
5. **Object types.** Standard and Premium store keys, secrets, and certificates. Managed HSM stores **keys only**. Cloud HSM stores keys and certificates. Payment HSM stores keys.
6. **Authentication plane.** Standard, Premium, and Managed HSM authenticate via **Microsoft Entra ID** (Microsoft's identity service, formerly Azure Active Directory). Cloud HSM and Payment HSM use native HSM credentials — leaving the Entra identity plane changes the audit and access story significantly.
7. **Operational responsibility.** Patching, business continuity within a region, and cross-region disaster recovery are gradually pushed onto the customer as you move from Standard to Payment HSM. Managed HSM specifically requires **manual** cross-region DR (see security domain below).

### Managed HSM and the security domain

The decisive Managed HSM concept is the **security domain**: a cryptographic blob created during an attested initialization ceremony that contains the HSM's data — key encryption material, signed RBAC role definitions, and the keys needed to restore an instance. It is encrypted with a quorum of customer-supplied RSA public keys using **Shamir's Secret Sharing** (a scheme that splits a secret into N shares, any M of which can reconstruct it — typically a 3-of-5 or similar **M-of-N** split). The HSM verifies its own state, encrypts the security domain to those keys, and returns the blob; Microsoft never has the private keys.

**Loss of the security domain means total destruction of every key in the HSM.** Microsoft has no recovery path. A lost domain is also a *compromised* domain — practical implication: all keys must be rotated and re-issued in a fresh instance under a new URL. This is the strongest "operator-cannot-touch-this" property in Azure key management and the single biggest reason to choose Managed HSM over Premium when the threat model requires it.

### Customer-managed keys across Azure services

Microsoft's CMK support page lists roughly 80 services, all using the same envelope-encryption pattern: a service-managed DEK, wrapped by the customer's KEK (the CMK) in Key Vault. Storage (Blob, File, Queue, Table, Disk, Backup, NetApp Files), Cosmos DB, SQL Database, SQL Managed Instance, MySQL Flexible Server, PostgreSQL Flexible Server, Synapse, AKS, App Service, Functions, Sentinel, Defender for Cloud, and many more support both Key Vault and Managed HSM as the KEK home.

Notable gaps and quirks worth knowing (verified against Microsoft's CMK support matrix, April 2026):

- **No Managed HSM support** for Microsoft Foundry, Content Safety, Document Intelligence in Foundry Tools, Azure Language in Foundry Tools, Bot Service, Health Bot, Container Registry, Container Instances, Logic Apps, IoT Hub Device Provisioning, and Azure Machine Learning. These remain Key Vault-only. If the requirement is "all keys in a customer-controlled HSM," these services don't satisfy it yet.
- **Power BI Embedded** is BYOK (bring-your-own-key) in preview only.
- **Azure Cache for Redis** supports CMK only on Enterprise / Enterprise Flash tiers.
- **Key-size constraints are per-service.** SQL Database / Synapse want RSA 3072; Data Lake Store is locked to RSA 2048. Plan rotation around the service's accepted sizes.
- **Rotation re-wraps; revocation cryptographically shreds.** Rotating the CMK creates a new key version under the same key URI, and the service re-wraps the existing DEK; data is not re-encrypted. Revoking the CMK (delete, access-policy revoke, or disabling the version) makes the wrapped DEK undecryptable — the data still exists, but no one can read it.

### Operational baseline for any HSM-backed key

- **Soft delete** is on by default for all newly created vaults and **cannot be disabled** once enabled (a permanent, one-way property), with a configurable 7-to-90-day retention window (default 90).
- **Purge protection** is *not* on by default; it must be enabled explicitly. When enabled, even the vault owner cannot purge during the retention window. Most compliance audits flag this gap.
- **Network isolation** through **Private Endpoint** (a private IP inside the customer's VNet that maps to the service), vault firewalls, and a "trusted Azure services" exception. Managed HSM additionally supports per-HSM network ACLs.
- **RBAC vs. access policies.** Key Vault has both legacy access policies and modern **Azure RBAC** (role-based access control evaluated by the Azure control plane); Microsoft is steering customers toward RBAC. Managed HSM uses *local* RBAC — role assignments live inside the HSM itself, not in Azure RBAC, because the HSM is the root of trust for its own authorization decisions.
- **Audit.** Both Key Vault and Managed HSM emit data-plane logs to Log Analytics, Event Hubs, or Storage. Every key operation is logged.

## 17. Secure Key Release: Closing the Loop

§§1–2 introduced silicon-rooted hardware identity. §16 chose where the key lives. **Secure Key Release (SKR)** is the mechanism that ties the two ends together by gating key release on a fresh attestation token.

SKR is a feature of Azure Key Vault Premium and Managed HSM. A key is created with `exportable=true` and an attached **release policy** — a JSON document that pattern-matches MAA claims: the TEE type, the image's **SVN** (security version number), and SGX-specific values like **MRENCLAVE** (the hash of the enclave's measured contents) and **MRSIGNER** (the hash of the public key that signed the enclave), plus the publisher, version, and debug flag. On Managed HSM, using the key requires the role `Managed HSM Crypto Service Release User`.

The flow:

1. A confidential VM or enclave performs remote attestation against MAA.
2. MAA, behind the scenes, validates the TEE quote against the collateral served by THIM.
3. MAA issues a signed JWT containing the platform's claims and any relying-party nonce.
4. The workload presents the JWT to Key Vault or Managed HSM.
5. The HSM evaluates the release policy against the JWT claims and releases the wrapped key only if the claims match.
6. The workload unwraps the DEK inside its TEE; the HSM never sees the plaintext key outside the TEE-bound transport.

The chain end-to-end, made explicit:

> silicon root of trust → measured boot log → host attestation → THIM-served vendor collateral → MAA-issued JWT → SKR policy match → key release → DEK unwrap inside the TEE → data decrypted inside the TEE.

Each step inherits trust from the one below it. This is what makes confidential VM disk encryption credible against a malicious operator threat model — a host that has not produced a valid attestation literally cannot obtain the key.

**Customer Lockbox is unrelated to all of this.** It governs how Microsoft support engineers obtain temporary access to customer data; it does not interact with Key Vault, CMK, or SKR. The two are sometimes confused because both involve "approving access to something sensitive." Customer Lockbox is covered in §20.3 below.

---

## Part IV — Shared Responsibility and Customer Data

The contractual layer that sits on top of all three earlier Parts: how Microsoft and the customer split security work, what each side commits to do, and what limits Microsoft places on its own ability to reach into customer data.

The shared-responsibility document is the contract: it draws a line down the technology stack and says who patches what, who configures what, who is accountable for what. The Azure security overview catalogs Microsoft's side of the line — the layered security services Azure ships and the ones the customer is expected to wire up. The customer-data-protection document zooms in on a specific commitment Microsoft makes on its side: keeping Microsoft itself from interfering with customer workloads and data. (The other half of "isolation" — keeping customers from interfering with each other on shared infrastructure — is in Part II above.)

## 18. The Shared Responsibility Framework

### The matrix

Cloud security duties are split between Microsoft and the customer along a layered stack. The boundary moves depending on the service category:

| Layer | On-prem | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Customer data | Customer | Customer | Customer | Customer |
| Configurations & settings | Customer | Customer | Customer | Customer |
| Identities & users | Customer | Customer | Customer | Customer |
| Client devices | Customer | Customer | Customer | Shared |
| Applications | Customer | Customer | Shared | Shared |
| Network controls | Customer | Customer | Shared | Microsoft |
| Operating system | Customer | Customer | Microsoft | Microsoft |
| Physical hosts | Customer | Microsoft | Microsoft | Microsoft |
| Physical network | Customer | Microsoft | Microsoft | Microsoft |
| Physical datacenter | Customer | Microsoft | Microsoft | Microsoft |

A useful intuition for the categories:

- **IaaS** (Infrastructure as a Service, e.g., Azure Virtual Machines): renting an empty plot with utilities. Microsoft gives you raw VMs, disks, virtual networks; you install and patch the operating system, runtime, and application.
- **PaaS** (Platform as a Service, e.g., Azure App Service, Azure Functions, Azure SQL Database): renting an unfurnished house. Microsoft runs the operating system and runtime; you deploy code and configuration.
- **SaaS** (Software as a Service, e.g., Microsoft 365, Dynamics 365): renting a serviced hotel room. Microsoft runs the entire application; you log in and use it.
- **On-premises**: your datacenter, your problem at every layer.

### Things that never transfer

Regardless of service category, four areas always belong to the customer:

- **Data classification and protection decisions** — only you know what is sensitive.
- **Identity and account lifecycle** — Microsoft Entra ID (the cloud identity service, formerly Azure Active Directory) is operated by Microsoft, but enrollment, multi-factor authentication (MFA) enforcement, conditional access policies, and account hygiene are yours. The matrix's "Customer" label for identity hides this nuance: Microsoft *runs* the service, you *configure* it.
- **Endpoint security** — laptops and phones touching the cloud are yours.
- **Access management** — role-based access control (RBAC), role assignments, conditional access.

### "Responsible for" is not "configurable by"

The most common shared-responsibility mistake is conflating those two. Microsoft is responsible for hypervisor security; you cannot configure it. You are responsible for data classification; nobody else can do it for you. The matrix tells you the *accountability* split, not the *control* split.

### Common failure modes

- Spinning up an IaaS VM and assuming Azure auto-patches the operating system. It doesn't.
- Treating identity as Microsoft's job because Entra ID is hosted. You still own MFA enforcement, conditional access, and account hygiene.
- In PaaS, forgetting that application code, secrets, and app configuration are still yours.
- Assuming SaaS removes endpoint responsibility. It doesn't.

## 19. Microsoft's Side: Defense-in-Depth Services

Once the line is drawn, the next question is what Microsoft actually delivers below it. The Azure security overview groups customer-configurable services into six functional areas. Each addresses a different layer of the stack.

### Identity — the new perimeter

**Microsoft Entra ID** (the Azure cloud identity directory; renamed from Azure Active Directory in July 2023) provides single sign-on, multi-factor authentication, conditional access, and identity governance.

Sub-features worth naming:

- **Privileged Identity Management (PIM)** — just-in-time, time-bound, approval-gated activation of administrator roles, so that elevated permissions are granted only when needed and only for a bounded window. Requires a Microsoft Entra ID P2 or Entra ID Governance license.
- **Managed identities** — Azure-managed credentials attached to compute resources so they can authenticate to other Azure services without storing secrets in code.
- **Conditional Access** — a policy engine that applies signals (user, device, location, sign-in risk) to gate logins.

### Perimeter and network

- **Azure DDoS Protection** — auto-tuned mitigation of distributed denial-of-service attacks at the network and transport layers (OSI layers 3 and 4). Two tiers: Network Protection (per virtual network) and IP Protection (per public IP).
- **Azure Firewall** — stateful firewall delivered as a managed service in three tiers (Basic, Standard, Premium). Premium adds inspection of encrypted TLS traffic, an intrusion detection and prevention system (IDPS), and URL filtering.
- **Web Application Firewall (WAF)** — runs in Application Gateway or Azure Front Door. Pre-configured rule sets defend against the OWASP Top 10 (the Open Web Application Security Project's published list of the most common web-app vulnerability classes, e.g., injection, broken authentication, cross-site scripting).
- **Azure Virtual Network Manager** — central authority for security admin rules that override individual NSGs across many virtual networks.

(Detailed network-isolation primitives — VNets, NSGs, Service vs. Private Endpoints, VPN, ExpressRoute — live in §11.)

### Compute

- **Trusted Launch** — adds Secure Boot, a virtual Trusted Platform Module (vTPM, a software TPM 2.0 instance per VM for measurement and attestation keys), and boot integrity monitoring on Generation 2 Azure VMs. The same primitives that protect the host fleet (§§4–5), exposed inside the guest. **Flag:** verify current GA status against Microsoft Learn before relying on it as default.
- **Confidential computing** — memory-encrypted VMs and enclaves: AMD SEV-SNP, Intel TDX, Intel SGX, and confidential GPU workloads on NVIDIA H100. See §9 above and [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/).
- **Defender for Servers** — endpoint detection and response (EDR) layered on Microsoft Defender for Endpoint.
- **Encryption at host** — VM disk encryption that supersedes the older Azure Disk Encryption (ADE). ADE retires September 15, 2028; existing ADE workloads must migrate to encryption at host before then. (See §14 for ADE specifics.)

### Storage and data

Access to storage is controlled by Azure RBAC plus **Shared Access Signatures (SAS)** — short-lived URL tokens that grant a specific client a specific permission set on a specific resource for a specific window, without sharing the storage account key. Storage Service Encryption is on by default for new accounts. Storage Analytics logs all authentication requests. (Detailed encryption mechanics are in §14.)

### Application

- **App Service Authentication / Authorization** — drop-in identity for web apps without code changes.
- **App Service Environments (ASE v3)** — single-tenant deployment of the App Service stack into your virtual network.
- Customer-driven penetration testing is allowed without prior notification, subject to Microsoft's Rules of Engagement.

### Operations — the cross-cutting layer

- **Microsoft Defender for Cloud** (formerly Azure Security Center plus Azure Defender; merged and renamed at Microsoft Ignite, November 2021). Microsoft positions it as a Cloud-Native Application Protection Platform (CNAPP): a single product that combines Cloud Security Posture Management (CSPM, which continuously scans cloud configurations against best practices) with Cloud Workload Protection (CWPP, runtime threat protection for VMs, containers, storage, databases, and serverless functions). The free tier emits prioritized recommendations; workload-specific plans (Defender for Servers, Containers, Storage, Databases) are paid add-ons.
- **Microsoft Sentinel** (formerly Azure Sentinel; renamed November 2021). Cloud-native SIEM (Security Information and Event Management — log collection, correlation, and analytics) plus SOAR (Security Orchestration, Automation, and Response — automated playbooks that act on detections), with built-in user/entity behavior analytics and threat intelligence.
- **Azure Monitor** plus **Log Analytics workspaces**, queried with **KQL** (Kusto Query Language, Microsoft's read-only SQL-like query language for telemetry data). Without diagnostic settings sending logs to Log Analytics, there is no forensic trail.
- **Azure Policy** — JSON-defined rules evaluated against resource properties. *Initiatives* bundle policies into compliance baselines.
- **Just-in-time (JIT) VM access** (a Defender for Cloud feature) — locks inbound management ports (SSH 22, RDP 3389) at the NSG, opens them on authorized request for a bounded window, then auto-closes.

### Renamed services to map across older docs

| Old name | Current name | Year |
|---|---|---|
| Azure Active Directory | Microsoft Entra ID | 2023 |
| Azure Security Center + Azure Defender | Microsoft Defender for Cloud | 2021 |
| Azure Sentinel | Microsoft Sentinel | 2021 |
| Online Services Terms (OST) | Microsoft Product Terms | rolling since July 2020 |

### Defaults vs. configurables

The overview implicitly distinguishes **default platform security** (DDoS auto-protection, encryption-at-rest defaults for Storage and SQL, Entra ID authentication, built-in threat detection in Defender for Cloud's free tier) from **advanced services the customer must enable** (paid Defender plans, Sentinel, Confidential VMs, Private Endpoints, Customer Lockbox, PIM). Treat the first set as baseline; treat the second set as design decisions per workload.

## 20. Customer Data Protection: Constraints on Microsoft Itself

§11 covers how Microsoft keeps customers apart from each other. This section answers a different question: *how does Microsoft keep itself from accessing customer data?*

### The legal stance

Microsoft's commitment is contractual: under the **Data Protection Addendum (DPA)**, Microsoft is the *processor* and the customer is the *controller* (GDPR terms — the controller decides why and how personal data is used; the processor handles it on the controller's behalf under documented instructions). The page's claim that "Microsoft doesn't claim ownership over customer data" is a legal statement, not a physical one — it means Microsoft contractually commits not to use, mine, or share the data outside service-delivery scope. Physical isolation is enforced by the controls below, not by ownership.

The contractual stack to know:

- **DPA** — defines processor/controller obligations and security commitments. Available at `aka.ms/DPA`.
- **Microsoft Product Terms** — the umbrella terms document. **Replaced the Online Services Terms (OST)**; any reference to "OST" is outdated.
- **GDPR** alignment is delivered through the DPA's EU/EEA-specific terms.

### Microsoft personnel access

Stated policy: access to customer data is **denied by default**. When access is needed for a support case, three controls apply:

- **Just-in-time access** — privilege granted for a bounded window, audited.
- **Multi-factor authentication** is required, from secured workstations.
- **No Microsoft accounts on customer VMs** by default.

The operator-side mechanics — MCIO, just-in-time access, Secure Admin Workstations — are described in §10.4. Customer Lockbox, below, is the customer's pre-approval gate on top of those defaults.

### Customer Lockbox

Customer Lockbox is the explicit pre-approval workflow on top of just-in-time access. When a Microsoft support engineer requests access to customer data, a designated approver in the customer's organization (Subscription Owner, Entra Global Administrator, or Customer Lockbox Approver role) must accept or deny the request through the Azure portal.

Specifics:

- The request expires after **12 hours** if no one in the customer organization actions it. It remains visible in the queue for up to **4 days** during that escalation window.
- Once approved, access is granted only for the duration in the request (with a 4-hour maximum) and auto-revoked after.
- **Customer Lockbox is opt-in for Azure** — it is not on by default. Without it, just-in-time access still applies but customer pre-approval does not.

This is a meaningful default to be aware of: just-in-time *limits* Microsoft access, but Customer Lockbox *gates* it on customer consent.

### Data residency: where the bits live

Two related but distinct concepts:

- **Data residency** = the physical location of the bits.
- **Data sovereignty** = which legal regime governs them. Azure offers residency choice, but sovereignty is harder. The U.S. CLOUD Act (the 2018 *Clarifying Lawful Overseas Use of Data Act*) lets U.S. authorities compel U.S.-based providers to disclose data they hold regardless of where it is stored — Microsoft itself has publicly acknowledged it cannot offer an absolute guarantee against this for non-U.S. customer data.

Three storage replication options affect residency:

- **LRS (Locally Redundant Storage)** — three copies, one facility, one region.
- **ZRS (Zone-Redundant Storage)** — three copies across two or three availability zones.
- **GRS (Geo-Redundant Storage)** — six copies; three in the primary region plus three in a paired region hundreds of miles away. **GRS is the default at storage-account creation.**

If residency is a hard requirement, switch from GRS to LRS or ZRS deliberately — the default replicates your data into a *second region*.

### Hardware disposition

When data-bearing devices retire, Microsoft sanitizes them on-site following **NIST SP 800-88** guidelines (the U.S. National Institute of Standards and Technology's standard for media sanitization, which defines three escalating categories: *Clear*, *Purge*, and *Destroy*), with physical and logical chain-of-custody tracking through final disposition. A disk delete triggers a NIST 800-88 *Clear* before space is reallocated.

### Government data requests

Stated commitments: no direct or unfettered government access; no provision of encryption keys; a court order or warrant is required for content. For enterprise data, Microsoft redirects requesters to the customer wherever possible. Microsoft publishes a biannual **Law Enforcement Requests Report**; well under 1% of demands target enterprise customers (the H1 2025 report records 168 such requests against many tens of thousands of consumer-data requests).

### Subscription termination timeline

- Billing stops immediately on cancellation.
- A retention window of 30 to 90 days allows recovery.
- The subscription auto-deletes once that window closes (90 days at the latest).
- Immutable or retention-locked data persists read-only until its retention policy expires, even after the subscription is gone.

---

## Part V — Through-Lines and Practical Takeaways

## 21. The Through-Line

Reading across the four Parts, several connections become explicit:

- **The chain.** Silicon RoT (§§1–2) verifies firmware (§§3–4), firmware verifies the bootloader (§4), the bootloader runs a kernel under code-integrity policy with HVCI (§6), and dm-verity gates every disk read (§6). The hypervisor (§8) launches only after this is all verified, then extends isolation laterally between tenants. Hashes (§7) are the single primitive every layer reduces to.
- **The boundary.** The hypervisor (§8) is the load-bearing isolation primitive. *Every* customer-facing isolation guarantee — multi-tenant VM separation (§11.2), encryption at host (§14.4 — encryption happens on the host before data leaves for storage), host attestation (§12 — the agent runs in the Hyper-V root partition) — references this same boundary. Confidential computing (§9) is the only Azure compute mode in which the customer does not have to trust this boundary.
- **The contract.** Shared responsibility (§18) splits accountability; Customer Lockbox (§20.3) plus just-in-time access (§10.4) split who can reach customer data; CMK with revocation (§14.5) gives the customer a kill switch on the keys that decrypt it.
- **The Fabric Controller's own use of encryption** (§10.3) for application images and hardware credentials shows that the patterns recommended to customers — encryption at rest, key separation, restricted custodians — are applied internally to the platform itself.

### Where the trust chain ends — the customer's choice

Default encryption-at-rest with a platform-managed key trusts the entire Azure stack. CMK in Managed HSM removes Microsoft from key custody. Confidential VMs with attestation-gated SKR (§17) remove Microsoft from runtime key access too. The customer chooses how far to extend trust by picking the corresponding combination of compute mode, key manager, and SKR policy.

### Sovereignty vs. security in key management

All five HSM options in §16 are cryptographically strong; the meaningful differences are who holds the security domain, who patches, where authentication lives, and which compliance regime applies. **Key management on Azure is a sovereignty choice, not a security choice.**

## 22. Practical Takeaways

### Defaults Azure provides without customer action

- Secure Boot enabled at factory, tooled to stay enabled across the buildout pipeline.
- Project Cerberus on Azure motherboards as the hardware RoT.
- TPM 2.0 measured-boot logs feeding host attestation.
- Code-integrity policy gating Windows host kernel loads, validated at build time by Code Signature Validation, staged at deploy, paged on at runtime.
- Hypervisor-enforced VM-to-VM isolation. There is no opt-out.
- Just-in-time operator access. Customer Lockbox layered on top is opt-in.
- Storage Service Encryption with AES-256-GCM, always on, all redundancy tiers, all blob types and access tiers.
- TDE on Azure SQL by default since June 2017.
- Server-side encryption on managed disks with platform-managed keys.
- Encryption at host on temp / ephemeral disks for Vv5+ VM SKUs (older SKUs require explicit enablement).
- MACsec on Microsoft's inter-datacenter backbone.

### Defaults customers should know about

- **Trusted Launch** for Generation-2 Azure VMs adds Secure Boot, vTPM, and boot-integrity monitoring inside the guest — verify support per SKU/image before relying on it.
- **HVCI / Memory Integrity** is default-on (alongside VBS and Credential Guard) for clean installs of Windows 11 22H2 and later, and Windows Server 2025, on hardware that meets the requirements; older Windows Server SKUs (2019, 2022) require explicit enablement.
- **dm-verity** in Azure Linux is opt-in via `ReadOnlyVerityRoot`; combine with `RootHashSignatureEnable` so the root hash is bound to the kernel's trusted keyring. Azure Linux's OS Guard variant ships this on by default for the container host.
- **Soft delete** on Key Vault is on and cannot be disabled. **Purge protection** is *off* by default — turn it on if compliance demands it.
- **ADE retires September 15, 2028.** Migrate ADE-encrypted VMs to encryption at host before then.
- **TLS 1.2 minimum on Azure Storage** is enforced globally on February 3, 2026.
- **Infrastructure encryption** for Storage must be enabled at account creation; you cannot retro-enable it.
- **GRS is the default** at storage-account creation — it replicates data into a paired region; switch to LRS/ZRS deliberately if residency is constrained.

### Day-one baseline

- **Microsoft Entra ID** with MFA, conditional access, and RBAC. Identity is the new perimeter.
- **NSGs on every subnet** as the cheapest baseline traffic control.
- **Encryption at rest** — on by default; verify the customer-managed-key story if you operate under regulatory constraints.
- **Microsoft Defender for Cloud (free tier)** for a posture score and recommendations.
- **Azure Policy** with at least one regulatory initiative assigned at the management-group scope.
- **Diagnostic settings forwarding to a Log Analytics workspace** — without this, no forensic trail.

### Per-workload decisions

- **Tenant isolation choice** — pick once, deliberately. Default multi-tenant works for most workloads; move up the spectrum (Isolated SKU, Dedicated Host, Confidential VM) only when the threat model demands it.
- **Network isolation choice** — Private Endpoints over Service Endpoints whenever traffic must avoid public IPs.
- **Application isolation choice** — ASE v3 over multi-tenant App Service for regulated workloads.
- **Defender plans, Sentinel, Confidential VMs, Private Endpoints, PIM, JIT VM access** — paid or higher-tier; enable per workload as the threat model demands.
- **Customer Lockbox** — opt in if you want pre-approval over Microsoft engineer access during support.
- **Replication tier (LRS / ZRS / GRS)** — pick deliberately; the default is GRS, which crosses region boundaries.
- **CMK** — use when you need revocation power, separation of duties between key custodians and DBAs, or compliance such as FedRAMP High or PCI-DSS key control.

### Most-common shared-responsibility traps

- Assuming Azure auto-patches your IaaS VM operating system. It doesn't.
- Treating identity configuration as Microsoft's job. It isn't.
- Forgetting that PaaS still leaves application code, secrets, and configuration to you.
- Letting the GRS default carry data into a second region when residency is contractually constrained.

### When to break the chain on purpose

- Use a **Confidential VM** (DCasv5 / ECasv5 on AMD SEV-SNP, DCesv6 / ECesv6 on Intel TDX) with a confidential OS disk, and attach the disk to a key in Managed HSM with an SKR policy that requires a fresh MAA token. The DEK is unwrapped inside the TEE only when the platform attests as compliant. See [Secure Key Release]({{ site.baseurl }}/docs/core/secure-key-release/).
- Use a **Confidential GPU** (the `NCCadsH100v5` series, e.g., `Standard_NCC40ads_H100_v5`, NVIDIA H100 NVL on AMD SEV-SNP) when the same threat model applies to GPU workloads. See [GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/).

### When each KMS makes sense

- **Key Vault Standard** for prototypes and low-stakes secrets.
- **Key Vault Premium** for cloud-native production workloads needing HSM-backed keys without sovereignty concerns.
- **Managed HSM** for regulated workloads (banking, PCI DSS) where the customer must hold the security domain.
- **Cloud HSM** for lift-and-shift IaaS migrations from on-prem HSMs (full native PKCS#11, JCA, CNG).
- **Payment HSM** if and only if you are doing PCI HSM-level payment processing.

### On suspected compromise

- Rotate first, reconfigure, *then* disable the old key.
- Use RSA-OAEP-256 for wrapping. Avoid plain RSA-OAEP (SHA-1 based).

### Operator hygiene

- Use Privileged Access Workstations for subscription owners and admins. An endpoint compromise bypasses everything else.
- Keep **Managed HSM security-domain quorum private keys** on separate encrypted media in separate physical locations. Loss is unrecoverable.
- **Monitor Key Vault and Managed HSM data-plane logs** to Log Analytics. Without that, no forensic trail.
- Configure Key Vault access logs and Azure Rights Management Service (RMS) usage logging.

### On hash hygiene

- Treat any signature over a SHA-1 digest as untrusted. Use SHA-256 or stronger across TLS, code signing, and PCR banks.

### On TCB recovery

- Treat **TCB recovery events** as scheduled work — when Intel or AMD bump SVNs, plan to re-attest workloads using SKR. THIM's slower baseline buys time but doesn't eliminate the need.

### Connecting to confidential computing

Standard multi-tenant VMs, dedicated hosts, and isolated SKUs (§11.2) all defend against neighbor tenants but ultimately trust the hypervisor and the host operating system. Confidential VMs (AMD SEV-SNP, Intel TDX) and confidential GPU workloads (NVIDIA H100) remove that trust by encrypting VM memory in hardware and proving system state via remote attestation. For background, see [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/), [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), and [Intel TDX]({{ site.baseurl }}/docs/core/tdx/).

## Sources

### Azure security architecture and infrastructure
- [Introduction to Azure security](https://learn.microsoft.com/en-us/azure/security/fundamentals/overview)
- [Azure information system components and boundaries](https://learn.microsoft.com/en-us/azure/security/fundamentals/infrastructure-components)
- [Hypervisor security on the Azure fleet](https://learn.microsoft.com/en-us/azure/security/fundamentals/hypervisor)
- [Isolation in the Azure Public Cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/isolation-choices)
- [Shared responsibility in the cloud](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility)
- [Protection of customer data in Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/protection-customer-data)
- [Customer Lockbox for Microsoft Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/customer-lockbox-overview)
- [Microsoft Products and Services Data Protection Addendum (DPA)](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA)
- [What are Azure availability zones?](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
- [Azure Dedicated Hosts overview](https://learn.microsoft.com/en-us/azure/virtual-machines/dedicated-hosts)
- [App Service Environment overview](https://learn.microsoft.com/en-us/azure/app-service/environment/overview)
- [Choose an Azure compute service](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction)
- [What is Microsoft Sentinel?](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [Azure storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Data-bearing device destruction (Microsoft Service Assurance)](https://learn.microsoft.com/en-us/compliance/assurance/assurance-data-bearing-device-destruction)
- [NIST SP 800-88 Rev. 1 — Guidelines for Media Sanitization](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-88r1.pdf)

### Trust chain (firmware, boot, runtime)
- [Secure boot — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/secure-boot)
- [Firmware security — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/firmware)
- [Platform code integrity — Azure Security](https://learn.microsoft.com/en-us/azure/security/fundamentals/code-integrity)
- [dm-verity — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/verity.html)
- [UEFI — Wikipedia](https://en.wikipedia.org/wiki/UEFI)
- [UEFI Specification 2.11 (UEFI Forum, December 2024)](https://uefi.org/specs/UEFI/2.11/)
- [Project Cerberus — Open Compute Project](https://www.opencompute.org/projects/security)
- [Project Mu — Microsoft](https://microsoft.github.io/mu/)
- [Secure Hash Algorithms — Wikipedia](https://en.wikipedia.org/wiki/Secure_Hash_Algorithms)
- [SHAttered — first SHA-1 collision (CWI Amsterdam / Google, 2017)](https://shattered.io/)
- [BootHole (CVE-2020-10713) — Eclypsium write-up](https://eclypsium.com/blog/theres-a-hole-in-the-boot/)
- [Microsoft guidance on the BlackLotus campaign (CVE-2022-21894)](https://www.microsoft.com/en-us/security/blog/2023/04/11/guidance-for-investigating-attacks-using-cve-2022-21894-the-blacklotus-campaign/)
- [LogoFAIL — Binarly research on UEFI image-parser flaws](https://www.binarly.io/blog/finding-logofail-the-dangers-of-image-parsing-during-system-boot)
- [App Control for Business and AppLocker overview](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview)
- [Trusted Launch for Azure VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/trusted-launch)
- [Credential Guard / VBS default-on, Windows 11 22H2+](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/)
- [Azure Linux — read-only verity roots](https://github.com/microsoft/azurelinux/blob/3.0/toolkit/docs/security/read-only-roots.md)
- [Azure Linux with OS Guard (announcement, May 2025)](https://techcommunity.microsoft.com/blog/linuxandopensourceblog/azure-linux-with-os-guard-immutable-container-host-with-code-integrity-and-open-/4437473)

### Silicon, attestation, encryption, key management
- [System on a chip — Wikipedia](https://en.wikipedia.org/wiki/System_on_a_chip)
- [Caliptra — open-source silicon root of trust](https://github.com/chipsalliance/Caliptra)
- [Caliptra 2.1 RTL release — CHIPS Alliance](https://www.chipsalliance.org/news/caliptra2-1/)
- [Azure Cobalt 100-based VMs are now generally available](https://azure.microsoft.com/en-us/blog/azure-cobalt-100-based-virtual-machines-are-now-generally-available/)
- [Announcing Cobalt 200: Azure's next cloud-native CPU](https://techcommunity.microsoft.com/blog/azureinfrastructureblog/announcing-cobalt-200-azure%E2%80%99s-next-cloud-native-cpu/4469807)
- [DC family confidential VM sizes (DCasv5, DCesv5, DCesv6)](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dc-family)
- [NCCadsH100v5 series — confidential GPU VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/gpu-accelerated/nccadsh100v5-series)
- [Announcing general availability of Azure Intel TDX confidential VMs (DCesv6 / ECesv6)](https://techcommunity.microsoft.com/blog/azureconfidentialcomputingblog/announcing-general-availability-of-azure-intel%C2%AE-tdx-confidential-vms/4495693)
- [Firmware measured boot and host attestation](https://learn.microsoft.com/en-us/azure/security/fundamentals/measured-boot-host-attestation)
- [Trusted Hardware Identity Management](https://learn.microsoft.com/en-us/azure/security/fundamentals/trusted-hardware-identity-management)
- [Azure Attestation overview](https://learn.microsoft.com/en-us/azure/attestation/overview)
- [Encryption overview](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-overview)
- [Azure data encryption at rest](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-atrest)
- [Azure Storage Service Encryption for data at rest](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)
- [Data security and encryption best practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/data-encryption-best-practices)
- [Double Encryption in Microsoft Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/double-encryption)
- [How to choose the right key management solution](https://learn.microsoft.com/en-us/azure/security/fundamentals/key-management-choose)
- [Azure services that support customer-managed keys](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-customer-managed-keys-support)
- [Secure Key Release with Azure Key Vault and Confidential Computing](https://learn.microsoft.com/en-us/azure/confidential-computing/concept-skr-attestation)
- [About the security domain in Managed HSM](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/security-domain)
- [Azure Managed HSM and Key Vault Premium are now FIPS 140-3 Level 3](https://techcommunity.microsoft.com/blog/microsoft-security-blog/azure-managed-hsm-and-azure-key-vault-premium-are-now-fips-140-3-level-3/4418975)
- [Server-side encryption of Azure managed disks](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption)
- [Migrate from Azure Disk Encryption to encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-migrate)
- [About encryption for Azure ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-encryption)
- [Enable infrastructure encryption for double encryption of data](https://learn.microsoft.com/en-us/azure/storage/common/infrastructure-encryption-enable)
- [Enforce a minimum required version of TLS for Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/transport-layer-security-configure-minimum-version)
- [IEEE 802.1AE — MACsec (Wikipedia)](https://en.wikipedia.org/wiki/IEEE_802.1AE)
- [Azure Storage TLS 1.0/1.1 retirement, Feb 3 2026](https://techcommunity.microsoft.com/blog/azurestorageblog/tls-1-0-and-1-1-support-will-be-removed-for-new--existing-azure-storage-accounts/4026181)
- [FIPS 140-3 Transition Effort — NIST CSRC](https://csrc.nist.gov/projects/fips-140-3-transition-effort)
- [Azure Cloud HSM — Microsoft Azure](https://azure.microsoft.com/en-us/products/cloud-hsm)
