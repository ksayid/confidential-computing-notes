---
title: "Paravisor: Secure Execution in Confidential Computing"
parent: Core Concepts
nav_order: 6
---

The paravisor is a specialized layer for confidential computing, designed to enable secure guest OS operations in a Confidential Virtual Machine (CVM) without reliance on the hypervisor. This makes it compatible with legacy and modern OS versions, allowing minimal modifications.

## Core Functionality and Technical Details
### Trusted Execution and Privilege Levels
The paravisor operates at a high privilege level to manage TEE enlightenments:
- **Hyper-V**: Virtual Trust Level 2 (VTL2)
- **Intel TDX**: L1 VM level
- **AMD SEV-SNP**: VMPLO

This privileged position allows it to securely implement necessary TEE functions, supporting the OS with features it would otherwise expect from the hypervisor.

### Guest OS Enlightenments
These are specialized instructions or functionalities that allow the OS to perform secure tasks without requiring the hypervisor’s intervention. This helps maintain confidentiality by avoiding trust dependencies on the hypervisor.

### Key Confidential Computing Features
- **VM Guest State (VMGS) File**: An encrypted file storing guest state data inaccessible to the host.
- **Interrupt and Exception Management**: The paravisor intercepts and validates interrupts from the hypervisor and manages platform-specific exceptions.
- **Remote Attestation**: Supports security requirements by allowing either the paravisor or guest OS to conduct attestation.

## Paravirtualization and Unmodified Guest OS Support
The paravisor introduces paravirtualized devices for enhanced compatibility and performance, supporting minimally modified drivers for networking, storage, and more.

### Unmodified Linux Compatibility
For Linux guests, no kernel or filesystem changes are needed beyond adding drivers for attestation, allowing these OSes to run practically unaltered within CVMs.

## Enhanced Architecture with OpenHL
OpenHL is an advanced paravisor framework that introduces improvements for device support, performance, and security within confidential computing.

### Expanded Device Emulation and Translation
OpenHL supports broader device compatibility, including translating devices (for example, NVMe to SCSI) for optimized storage and network performance on updated Azure SKUs.

### Modular Structure
- **Virtual Machine Monitor (VMM)**: Written in Rust, emphasizing memory safety, operates primarily in user mode for managing processes.
- **Custom Linux Kernel**: A lightweight kernel that reduces system overhead, providing just enough support to run the VMM securely.

#### Core Goals
- **Memory Safety**: Rust minimizes memory-related vulnerabilities within the VMM.
- **Efficient Performance**: A minimalistic kernel supports the VMM with low overhead.
- **Future Flexibility**: The VMM’s OS-agnostic design enables potential deployment on other operating systems beyond Linux.

## Comparative Insight with Coconut and SBSM
While Coconut (ASVSM) and SBSM are solutions for fully enlightened guests, OpenHL’s paravisor focuses on limited-enlightenment guests with minimal OS modifications. All three solutions support the virtual TPM (vTPM), but they cater to distinct configurations and security requirements.

## Anticipated Future Enhancements
Potential future capabilities include bounce buffering and memory management so the paravisor can support private/shared memory operations without device-specific drivers.

In essence, the paravisor—bolstered by OpenHL—serves as a robust framework for confidential computing by securely managing guest OS needs in an isolated environment and advancing device compatibility, performance, and TEE feature support within CVMs.