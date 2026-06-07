---
title: "Paravisor: Secure Execution in Confidential Computing"
parent: Core Concepts
nav_order: 7
---

# Paravisor: Secure Execution in Confidential Computing

## The problem the paravisor solves

[Confidential computing]({{ site.baseurl }}/docs/core/confidential-computing-concepts/) improves the security of cloud VMs by putting the VM inside a hardware-enforced **trust boundary** — and the hypervisor is deliberately placed *outside* that boundary. The consequence is a gap: the trusted hypervisor services VMs normally rely on (device emulation, interrupt management, a virtual TPM, and so on) can no longer come from outside. The straightforward alternative is to run **"fully enlightened" guest OSes** — modified to understand and manage every aspect of being a confidential VM — and to keep updating those modifications as the underlying technology evolves. That is a heavy ongoing cost, and for legacy guests it is not realistic at all.

A **paravisor** closes the gap from inside. It is an execution environment that runs **inside the guest VM**, at a **higher privilege level than the guest OS**, and provides the virtualization and device services the guest expects. It implements the confidential-computing enlightenments on the guest's behalf, so the guest "doesn't seem like a confidential guest because it doesn't need to act like one" — existing and legacy OSes run securely without modification. Paravisors work in both confidential and non-confidential environments; in confidential ones, the privilege levels are enforced by the confidential-computing hardware.

## Where the paravisor sits

The layering inside a confidential VM looks like this:

```
Hardware
  └─ Real hypervisor (e.g., Hyper-V — a true Type 1)   ← outside the trust boundary
      └─ Confidential VM (trust boundary)
          ├─ Paravisor (e.g., OpenHCL)                  ← higher privilege, inside the guest
          └─ Guest OS (Windows / Linux)
```

The paravisor is neither **Type 1** nor **Type 2** in the standard [virtualization taxonomy]({{ site.baseurl }}/docs/core/virtualization/#type-1-and-type-2). It is not Type 1 because it does not boot on bare metal or control physical hardware — the real hypervisor underneath does. It is not Type 2 because it does not run as an app on a host OS — it runs *inside the guest*, below the guest OS in privilege. The Type 1 / Type 2 taxonomy classifies a hypervisor by its position relative to hardware and any host OS; a paravisor is defined by its position relative to **the guest**.

| | Type 1 hypervisor | Type 2 hypervisor | Paravisor (e.g., OpenHCL) |
|---|---|---|---|
| Sits | Directly on hardware | On a host OS | **Inside the guest VM**, above the guest OS in privilege |
| Hosts VMs on physical hardware? | Yes | Yes | No — services one guest from within |
| Boots | Before any OS | After host OS | With the VM, under a real hypervisor |
| Purpose | Run / isolate many VMs | Run VMs on a desktop OS | Provide trusted device / virtualization services inside a confidential trust boundary |

### Privilege levels per platform

The paravisor needs a privilege level the guest OS cannot reach. Each platform supplies one:

- **Hyper-V** — runs in **Virtual Trust Level 2 (VTL2)**, with the guest in VTL0. **Virtual Secure Mode (VSM)** is what makes the levels real; in confidential contexts, VSM is enforced hardware-platform-agnostically.
- **Intel TDX** — runs in the **L1 VMM** of the confidential VM.
- **AMD SEV-SNP** — runs in **VMPL0** (Virtual Machine Privilege Level 0 — hardware-isolated privilege layers within a VM).

## OpenHCL — Microsoft's open-source paravisor

Microsoft built the **first paravisor in the industry** — used for years in Azure under earlier names. **OpenHCL** is its new open-source incarnation, developed in the open at `microsoft/openvmm` on GitHub. It runs on **x86-64 and ARM64** and supports both **Intel TDX** and **AMD SEV-SNP**. As of October 2024 more than **1.5 million VMs were running OpenHCL in Azure per month**, and it ships in Azure Boost SKUs and future Azure confidential VM SKUs.

### What OpenHCL provides to a confidential VM

OpenHCL services confidential and non-confidential VMs alike. Four capabilities matter most:

1. **Device emulation** via standard interfaces — most importantly a **vTPM** and serial devices. The vTPM in particular cannot be implemented with guest-side enlightenments alone, so the paravisor is the natural home for it.
2. **Device translation** — for example, **NVMe → para-virtualized SCSI** — so hardware devices can be assigned directly to the VM for **accelerated I/O** without any guest-OS changes.
3. **Diagnostics support** for debugging confidential VMs, where traditional debugging is hard (the host can't see in, by design).
4. **Support for non-enlightened guests** — Windows and older Linux — via existing standard architectural interfaces. It also adds value to fully enlightened guests through the other three services above.

Two further capabilities are visible to the guest as part of its CVM contract: an **encrypted VM Guest State (VMGS) file** that stores guest state inaccessible to the host, and **interrupt and exception management** where the paravisor intercepts and validates interrupts from the hypervisor and handles platform-specific exceptions. **Remote attestation** can be conducted either by the paravisor or by the guest OS, depending on the security requirements.

### Architecture: three components

OpenHCL is built from three pieces, deliberately small:

1. **OpenVMM** — a modular, cross-platform VMM written in **Rust** (for memory safety), running primarily as user-mode processes. The same VMM serves confidential and non-confidential guests, which avoids fragmenting Microsoft's virtualization stack across two codebases and closes confidential-VM feature gaps in lock-step with the mainstream path.
2. A **boot loader**.
3. A **small customized Linux kernel** — minimal Kconfig, small binary, low RAM — supporting the VMM, so the VMM itself can be mostly standard Rust with its toolchain and crate ecosystem.

The design goals are memory safety (Rust everywhere it can be), efficient performance (a minimalist kernel with low overhead), and future flexibility (the VMM's OS-agnostic design leaves room for non-Linux kernel substrates later — see the OpenHCL-vs-SVSM section below).

## Two ways to run a guest in a confidential VM

There are two architectural approaches, and the paravisor exists because one of them is impractical at scale.

| | Fully enlightened | Paravisor-assisted |
|---|---|---|
| Guest OS changes | Modified to handle all confidential-VM aspects; continually updated | None — runs unmodified, even legacy OSes |
| Who handles enlightenments | The guest itself | The paravisor, on the guest's behalf |
| Interfaces | New / custom (e.g., the SVSM calling convention) | Existing standard architectural interfaces |
| Azure today | — | All IaaS confidential VMs |

Azure runs **all IaaS confidential VMs via a paravisor** today, which gives the widest possible guest support: Windows versions roughly a decade old (Windows 10 client, Windows Server 2019), Linux kernels as old as **5.19 (SNP)** or **6.6 (SNP + TDX)**, and older kernels still via backported patches.

### Why Windows specifically needs a paravisor

Windows is the load-bearing example, because two of its core dependencies are exactly what the paravisor is good at supplying.

The first is **APIC (interrupt-controller) emulation**. Normally the hypervisor does this — but in a confidential VM the hypervisor is outside the trust boundary, so something else inside the VM has to provide it. The candidates are the paravisor or the hardware itself, and early platforms (notably 3rd-Gen AMD EPYC) did not support APIC virtualization in hardware. Windows's architecture relies directly on the APIC for interrupt management: some interrupt handling is **inlined in drivers** rather than routed through the kernel, so the dependency is not something the OS can paper over. Linux, by contrast, exposes **kernel service routines** for interrupt state, so Linux drivers don't depend on the APIC directly — which is why fully-enlightened Linux is at least possible while fully-enlightened Windows is not.

The second is the **TPM**. Windows relies on a TPM for security features, and a **vTPM for a confidential VM cannot be implemented with enlightenments alone** — it is implemented in the paravisor. The plan is for Windows to **continue to be supported via paravisor** (not fully enlightened); for future Linux, both approaches are being evaluated.

## OpenHCL vs. COCONUT-SVSM

These are related but different solutions to the same hole left by pushing the hypervisor out of the trust boundary. An **SVSM** (Secure VM Service Module — COCONUT-SVSM is the open-source one) stores secrets and provides services to **fully enlightened guests** via **new interfaces**: the guest has to discover the SVSM and call it with a **custom calling convention**, which is why an SVSM solution requires guest modification.

OpenHCL is positioned differently. It provides services through **existing standard architectural interfaces** — devices "just work" through standard enumeration (PCI virtualization, the existing vTPM contracts) — with **no custom calling contract** for the guest to learn. That is what makes unmodified Windows and legacy Linux viable in the same VM shape.

A natural future direction is that OpenHCL **adopts the Rust-based COCONUT-SVSM as its kernel layer** (replacing the customized Linux kernel) for stronger memory safety, once COCONUT-SVSM grows the features OpenHCL needs.

## Roadmap

Several open work items determine where the paravisor goes next:

- **TDX support in Azure.** TD partitioning was co-developed by Intel and Microsoft, shipped with 5th-Gen Xeon, and backported to 4th-Gen. Azure was the first CSP with a TDX public preview and the first with Windows guest support on TDX. The Azure-side TDX SKU rollout is tracked in [§9 of the Azure Security Architecture note]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#9-where-the-trust-chain-ends--confidential-vms-and-gpus). The conceptual TDX background is in [Intel TDX]({{ site.baseurl }}/docs/core/tdx/).
- **Arm CCA support (Realms).** The co-created **Planes** feature lets a paravisor and a guest coexist at different privilege levels in one Realm — the analog of VTL2 / VMPL0 / L1 on the Arm side.
- **AMD SEV-SNP feature completeness** via the VMPL hierarchy.
- **KVM as host.** Today OpenHCL runs only on the Microsoft hypervisor; KVM is on the roadmap.
- **Red Hat collaboration** for cross-OS confidential interoperability.

## Key acronyms

VMM (Virtual Machine Monitor) • VTL (Virtual Trust Level) • VSM (Virtual Secure Mode) • VMPL (Virtual Machine Privilege Level) • APIC (Advanced Programmable Interrupt Controller) • vTPM (virtual TPM) • SVSM (Secure VM Service Module) • CCA (Arm Confidential Compute Architecture) • VMGS (VM Guest State)
