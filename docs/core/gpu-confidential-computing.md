---
title: NVIDIA GPU Confidential Computing
parent: Core Concepts
nav_order: 7
---

# NVIDIA GPU Confidential Computing

NVIDIA's GPU Confidential Computing (GPU-CC), introduced with the Hopper generation (H100) and continued in Blackwell, extends a confidential VM's protected boundary out across PCIe and into the GPU. Code, data, and model weights stay encrypted in transit and isolated from the host operating system, the hypervisor, cloud administrators, and other tenants — so an entire AI pipeline (CPU side and GPU side) can run on someone else's hardware with the same trust assumptions as a confidential VM alone. End-user adoption is transparent: existing CUDA applications run unchanged. Behind the scenes, a stack of dedicated processors inside the GPU does the work.

## The trust boundary

GPU-CC only makes sense paired with CPU-side confidential computing. The CPU side ([Intel TDX]({{ site.baseurl }}/docs/core/tdx/) or AMD SEV-SNP) creates a confidential VM (CVM) whose memory is encrypted with a hardware-managed key the host can't see. The GPU is "passed through" — exclusively assigned — to that CVM. GPU-CC then extends the trust line out to the GPU itself.

```
   ┌─────────────────────── HOST (untrusted) ───────────────────────┐
   │  hypervisor • host OS • BMC • firmware • cloud admins          │
   │                                                                │
   │  ┌──────────────────────────────┐    ┌─────────────────────┐   │
   │  │ Confidential VM (trusted)    │    │  GPU (trusted)      │   │
   │  │  app + CUDA + drivers        │◄══►│  CPR (private mem)  │   │
   │  │  CVM memory encrypted by     │    │  FSP / GSP / SEC2   │   │
   │  │  CPU-managed key             │    │  Copy Engines       │   │
   │  └──────────────────────────────┘    └─────────────────────┘   │
   │                  ▲     encrypted PCIe (untrusted)     ▲        │
   │                  └────────────────────────────────────┘        │
   └────────────────────────────────────────────────────────────────┘
```

Inside the protected zone:

- The CVM's memory and CPU registers (encrypted by TDX/SEV-SNP).
- The GPU's **Compute Protected Region (CPR)** — a sealed area of GPU memory only authorized GPU engines can touch.
- The GPU's internal security processors and their NVIDIA-signed firmware.

Outside, and treated as actively hostile:

- Hypervisor, host OS, system management mode, the BMC.
- The PCIe bus itself — assumed observable and tamperable.
- All other peripherals and other tenants on the box.

The rule that follows: nothing crossing PCIe between CVM and GPU may be in plaintext. Either the data is encrypted with a session key both sides agreed on, or it's signed for integrity, or both.

## The hardware inside the GPU

Four engines do the GPU-side work. All four are RISC-V microcontrollers running NVIDIA-signed firmware.

| Engine | Role |
|---|---|
| **FSP** (Foundation Security Processor) | Runs the very first stage of secure boot. Verifies and launches the GSP. |
| **GSP** (GPU System Processor) | Control plane: runs the resource manager, terminates the SPDM session that derives session keys, handles the RPC channel to the kernel-mode driver. |
| **SEC2** (Secure Processor) | Sets up the CPR, generates attestation reports, decrypts and verifies workloads, runs memory scrubbing. Reachable from both kernel-mode and user-mode software. By design, SEC2 can decrypt and verify but never encrypts. |
| **Copy Engines (CEs)** | Move data in and out of the CPR with hardware AES. Data leaving the CPR for shared memory gets encrypted; data entering the CPR from shared memory gets decrypted and integrity-checked. Eight logical CEs on H100. |

## How GPU-CC comes up

Bringing the GPU into a trustworthy state involves four phases that happen automatically when the kernel-mode driver enables GPU-CC mode.

### 1. Secure boot chain

Each component cryptographically verifies the next:

```
CEC EROT (motherboard root of trust) → FSP → GSP → SEC2
```

An external root of trust on the motherboard and the FSP's own boot ROM both verify the FSP firmware before it runs. The FSP then verifies the GSP firmware images shipped with the kernel-mode driver, and once the GSP is up, it brings up SEC2. (On Blackwell, parts of this sequence have shifted to consolidate trust management inside SEC2.)

### 2. Key derivation

The kernel-mode driver opens a Security Protocol and Data Model (SPDM) session with the GSP and negotiates a shared master secret. From that secret, around 40 session keys are derived:

- **6 keys** protect GSP communications (RPCs, DMA transfers, fault notifications).
- **6 keys** protect SEC2 channels (workload submission and memory scrubbing, split between user-mode and kernel-mode).
- **32 keys** protect the eight Copy Engines (separate keys per direction, per privilege level).

Each channel between CVM and GPU has its own key, with its own counter to defeat replay attacks.

### 3. PCIe firewall

In normal mode the host can read and write GPU control registers and GPU memory through PCIe Base Address Registers (BAR0 for control, BAR2 for memory). When GPU-CC turns on, the **BAR0 Decoupler** activates: nearly all control registers and all GPU memory in the CPR become unreachable from the host. About a thousand registers — for management functions like power and reset — remain accessible. Everything else returns zero, blurring whether a register is unmapped or merely blocked.

### 4. Attestation

The CVM-side verifier proves the GPU is real and running approved firmware before trusting it:

1. Pull the device certificate chain (anchored to a Device Identity Key burned into the GPU at manufacture, signed up through NVIDIA's Root CA).
2. Verify the chain (replacing the network-fetched root with a locally trusted copy) and check each certificate against NVIDIA's revocation service.
3. Pull a signed attestation report containing measurements (cryptographic hashes) of the loaded firmware.
4. Compare those measurements against expected "golden" values from NVIDIA's Reference Integrity Manifest (RIM) service.

If everything matches, the user transitions GPU-CC into READY state and starts sending workloads.

## How data moves at runtime

Because the host can no longer touch the CPR directly, every CPU-to-GPU transfer routes through a **staging buffer** — a shared memory region accessible to both sides:

```
CVM private memory ──► encrypt with session key ──► staging buffer
                                                         │
                                                         ▼
                            Copy Engine reads, decrypts into CPR
```

Six distinct data paths exist, each protected differently:

| Path | What flows over it | Protection |
|---|---|---|
| **CPU↔GSP RPC** | Driver→GSP commands and responses | Encrypted payloads; queue metadata partially exposed |
| **CPU↔GSP DMA** | Bulk memory transfers via the GSP | Encrypted; transfer size leaks via timing |
| **GPU memory faults** | Fault notifications GSP→CPU | Encrypted in shadow buffers; fault arrival observable |
| **UVM (Unified Virtual Memory)** | Pushbuffers and command queues for the unified CPU-GPU address space | Pushbuffers encrypted into CPR via SEC2-bootstrapped channels; some queue metadata exposed |
| **Memory scrubbing** | Wipe commands when freeing GPU memory | Signed (HMAC) but **not encrypted** |
| **CUDA** | Kernel code, kernel arguments, user data, launch configurations | Closed-source; assumed to use SEC2 + CE keys for everything across PCIe |

The general pattern: bulk data moves through Copy Engines (encrypted with per-channel session keys); commands and small structures move through SEC2 channels (signed with HMAC, often also encrypted). Inside the CPR, everything is plaintext — it has to be for the GPU cores to use it.

## What it defends against

| Threat | Status |
|---|---|
| Compromised hypervisor or host OS reading CVM/GPU memory | **Defended** |
| Cloud administrator with root access | **Defended** |
| Snooping or tampering with PCIe traffic | **Defended** for code/data contents |
| Replay of old encrypted commands or memory | **Defended** (per-channel IVs) |
| Malicious GPU firmware update or VBIOS flash | **Defended** (secure boot + signed images) |
| Denial of service (host refuses to schedule the CVM/GPU) | **Out of scope** — the host can always crash the system |
| Decapsulation, probing on-package memory | **Out of scope** — physically destructive, presumed too risky for adversaries |
| Bugs in the workload itself (CVM software or GPU kernels) | **Out of scope** — GPU-CC does not harden user code |
| Compromise of the CVM | **Catastrophic** — the CVM holds the keys; GPU-CC's guarantees collapse if the CVM falls |

The CVM is the load-bearing piece. GPU-CC inherits whatever trust assumptions the underlying TDX or SEV-SNP makes; recent vulnerabilities found in either of those propagate up.

## Known gaps

Independent security analysis of the H100 implementation (Gu et al., IBM Research and Ohio State, [arXiv:2507.02770](https://arxiv.org/abs/2507.02770), responsibly disclosed to NVIDIA's PSIRT) found the following residual leaks. Most are metadata or side-channel issues; none break confidentiality of bulk data, but they erode integrity and observability guarantees at the margins:

- **RPC metadata in plaintext.** Command and status payloads on the GSP RPC channel are encrypted, but surrounding metadata (queue read/write pointers, command identifiers) sit in shared memory in the clear. An attacker can see which RPCs are happening and can reorder, replay, or skip them by tampering with pointers.
- **Transfer-size timing channel.** Bulk DMA between CVM and GSP has execution times that scale visibly with transfer size, leaking how much data is moving even when the data itself stays encrypted.
- **Memory-fault PUT pointer exposure.** The position of the latest fault notification is exposed via BAR0, revealing fault rate and timing. Low risk because faults are infrequent.
- **SEC2 command-queue structures unprotected.** The GPFIFO ring, GPPUT pointer, and tracking semaphores SEC2 uses for command submission live in shared memory unencrypted. SEC2's command-signing requirement may prevent forged execution, but attackers can still observe and try to manipulate them.
- **Memory scrubbing signed but not encrypted.** Scrub commands are integrity-protected but not confidential; attackers can see scrub activity and tamper with completion semaphores.
- **CUDA path is opaque.** The CUDA runtime and user-mode driver are closed-source, so the actual protection of kernel launches and bulk data movement can't be independently verified — only inferred from what GPU-CC must require.

The recurring theme: NVIDIA encrypted the obvious things (data and commands) but left the surrounding scaffolding (queue pointers, semaphores, metadata, timing) exposed.

## Performance and availability

Performance overhead in practice is small and dominated by PCIe traffic, not GPU compute. Independent benchmarks on H100 show overhead typically below 7%, with the largest models (e.g., 70B-parameter LLMs) seeing essentially no overhead because compute-bound workloads spend little time on the encrypted PCIe path. The smallest models, the lowest batch sizes, and workloads with frequent model swapping see the highest overhead. NVIDIA reports near-parity with non-confidential mode on Blackwell.

GPU-CC instances are available on the major clouds (Azure, GCP, AWS, OCI) on H100 hardware, with Blackwell-based offerings rolling out.

## Practical limitations

- **Multi-GPU is constrained.** First-generation H100 GPU-CC supports only a single GPU passed through to a CVM; NVLink-protected multi-GPU configurations are part of newer NVIDIA Confidential Computing documentation but availability lags single-GPU.
- **Trusted I/O (TDISP) isn't yet the integration point.** GPU-CC predates and bypasses [TDISP]({{ site.baseurl }}/docs/core/tdx/trusted-io/), the PCI-SIG standard for trusted device assignment. NVIDIA uses its own SPDM-based handshake instead. Future GPUs are likely to converge with TDISP as the TDX and SEV-TIO ecosystems mature.
- **The CVM is the weakest link.** Hardening the CVM image (minimal kernel, no extra services, attested boot) matters as much as anything GPU-CC itself does.
- **Closed source.** Significant parts of GPU-CC live in NVIDIA-signed firmware and proprietary CUDA components. End users must trust NVIDIA's correctness; independent audit is partial at best.
- **AES-GCM IV exhaustion is a long-tail concern.** Session keys use AES-GCM, whose security depends on never reusing an IV. Per-channel IV counters make this a long-term design consideration rather than a near-term issue, but extremely long-lived sessions on heavily-used channels are theoretically a concern.

## Related Pages

- [Confidential AI]({{ site.baseurl }}/docs/core/confidential-ai/) — running AI workloads inside TEEs, including the CPU-side context GPU-CC pairs with.
- [Intel TDX]({{ site.baseurl }}/docs/core/tdx/) — one of the two CPU-CC technologies that hosts the CVM.
- [Intel TDX — Trusted I/O (TDISP)]({{ site.baseurl }}/docs/core/tdx/trusted-io/) — the standards-based future for confidential I/O, contrasted with NVIDIA's vendor-specific approach.

## Sources

- Gu et al., *Blueprint, Bootstrap, and Bridge: A Security Look at NVIDIA GPU Confidential Computing*, [arXiv:2507.02770](https://arxiv.org/abs/2507.02770).
- Mohan et al., *Confidential Computing on NVIDIA Hopper GPUs: A Performance Benchmark Study*, [arXiv:2409.03992](https://arxiv.org/abs/2409.03992).
- [NVIDIA Secure AI with Blackwell and Hopper GPUs (whitepaper)](https://docs.nvidia.com/nvidia-secure-ai-with-blackwell-and-hopper-gpus-whitepaper.pdf).
- [NVIDIA AI Security with Confidential Computing](https://www.nvidia.com/en-us/data-center/solutions/confidential-computing/).
