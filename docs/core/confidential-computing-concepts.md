---
title: Confidential Computing Concepts
parent: Core Concepts
nav_order: 1
---

**Confidential computing** refers to technologies and practices that isolate and protect data during processing, preventing unauthorized access—even by the owner of the hardware or a **cloud service provider** (CSP). This is primarily achieved using **trusted execution environments** (TEEs) and associated security mechanisms such as attestation, secure boot, and robust key management.

## Goals
### Confidentiality
* **Protection of data at rest and in transit**: Encryption ensures data remains protected even if intercepted.
    * Example: Encrypted database backups in an S3 bucket plus TLS-encrypted connections for data in transit. 
* **Isolation of computations**: TEEs ensure that sensitive code and data remain inaccessible to privileged software layers or untrusted parties.
    * Example: In finance, an enclave may securely compute credit scores without exposing raw credit data to the bank’s system administrators.

### Integrity
* **Prevention of unauthorized modifications**: Cryptographic attestation, checksums, and secure boot help verify that only trusted software is running.
    * Example: A device uses secure boot to ensure only a signed operating system image can load. Any tampering with the bootloader is detected and aborts the boot process.
* **Secure boot**: Ensures the system boots with trusted firmware and software, guarding against malicious changes.
    * Example: Modern laptops that use UEFI Secure Boot check signatures of the OS loader.
* **Data integrity checks**: Use of hashes, digital signatures, and other mechanisms to detect tampering.
    * Example: A container image registry that checks each image’s SHA-256 hash against a known “golden” reference before deployment.

### Availability
* **Always accessible**: Systems should be designed to ensure continuous service (redundancy, load balancing, failover mechanisms).
    * Example: A microservices architecture on Kubernetes that automatically re-schedules failed pods to keep services running.
* **Fault tolerance**: Replication, clustering, and robust resource management help mitigate hardware failures or other disruptions.
    * Example: A distributed database like Cassandra replicates data across multiple nodes and data centers to survive localized outages.

## Concepts
### Trusted Execution Environment (TEE)
A **TEE** is a secure and isolated environment within a computer system where sensitive data and code can be processed in a protected and confidential manner. 
* **Isolation**: The TEE’s memory is segregated from the rest of the system, preventing unauthorized access or tampering.
    * Example: Intel SGX enclaves allocate a region of memory (Enclave Page Cache) that is encrypted and cannot be read or modified by other system processes.
* **Hardware or Software**: TEEs can be implemented in hardware (e.g., Intel SGX, AMD SEV, Arm TrustZone) or software hypervisors.
    * Example: AMD SEV (Secure Encrypted Virtualization) encrypts each VM’s memory with distinct keys to isolate them from the hypervisor.

### Secure Enclave
A **secure enclave** is a hardware-based implementation of a TEE. It typically:
* **Runs on CPU hardware**: Intel SGX, Apple Secure Enclave, etc.
    * Example: Apple’s Secure Enclave on iPhones is used for fingerprint or face data; it stores biometric data in a separate chip region inaccessible to the main CPU OS.
* **Provides strict isolation**: Protects code and data from other processes, including privileged system processes.
    * Example: Intel SGX enclaves allow only encrypted reads by the CPU microcode, so even the OS kernel cannot inspect enclave memory.
* **Broader vs. narrower**: A TEE can be software-based, whereas a secure enclave is specifically hardware-enforced.

### Trusted Computing Base (TCB)
The **TCB** is the collection of hardware, software, and firmware components essential for enforcing a system’s security policies. When we talk about a "trusted" computing base, we don't necessarily mean that the system is secure, but that these components are critical for the system’s security. They are the root of trust, because the system assumes they are secure enough to be trusted. We must, after all, start trusting somewhere. This is actually what defines a TCB and why it must be as minimal as possible.
* **Critical for security**: Weaknesses or vulnerabilities in the TCB compromise the entire system’s integrity.
    * Example: If a hypervisor that’s part of the TCB has a critical bug, an attacker could potentially compromise every virtual machine.
* **Size matters**: Smaller TCBs are easier to audit and reason about.
    * Example: A single monolithic kernel is a large TCB, whereas a microkernel with minimal components is a much smaller TCB.

### Attestation
**Attestation** is the process of verifying the integrity and authenticity of a TEE or secure enclave before performing sensitive operations.
* **Proof of legitimacy**: Demonstrates that the TEE is genuine and unmodified.
    * Example: When an application starts an SGX enclave, it requests a quote from the CPU. This quote can be sent to a remote verifier to confirm the enclave is legit.
* **Cryptographic**: Often involves digital signatures, certificates, or platform-specific endorsements.
    * Example: Intel SGX uses an Intel Attestation Service that signs an enclave’s measurement so a remote party can validate that the enclave matches expected software.

### Confidential Virtual Machines (CVMs)

**What is a Confidential Virtual Machine (CVM)?**  
A CVM is a virtual machine that uses hardware-level memory encryption to protect its runtime data (memory).  
- It ensures that neither the host operating system, the hypervisor, nor other VMs on the host can read the VM’s memory, even though they share the same physical machine.

**How is this achieved?**  
- The CPU itself provides support for memory encryption (e.g., AMD SEV-SNP).  
- When the CPU encrypts a CVM’s memory pages, only the CVM “knows” the decryption key; the hypervisor does not.  
- The encryption keys stay in secure registers inside the CPU and never leave it.

**Why is it significant?**  
- Traditional VMs rely on trusting many layers (firmware, hypervisor, host OS). If any of those is compromised, the VM can be breached.  
- Confidential computing (CVMs, enclaves, etc.) reduces the TCB by removing the hypervisor and host OS from the circle of trust.  
- It extends the idea of protecting data at rest and in transit to also include data in use (memory encryption).

### TCB and Remote Attestation in CVMs
- **TCB**: With CVMs, hardware memory encryption and remote attestation reduce how many components you must trust.
- **Remote Attestation**: 
  - You measure (hash) the boot chain (e.g., bootloader, kernel) at VM startup.  
  - Compare these measurements to expected values—if they match, you “attest” that the VM’s software stack is unmodified.  
  - Only after successful attestation do you provide sensitive data to the CVM.

### Practical Benefits of CVMs
- **Isolation from a malicious hypervisor**: Even if an attacker gains control over the hypervisor, they cannot decrypt the CVM’s memory.
- **Attestation**: Ensures you’re running the code you trust before loading sensitive data.
- **Lower risk for co-tenant attacks**: Other VMs on the same physical host cannot inspect or tamper with your memory.

### CIA Triad in TEEs (Refresher)
- **Confidentiality**: Memory encryption ensures data inside a TEE or CVM is not visible from the outside.  
- **Integrity**:
  - Attestation verifies a trusted state at boot.  
  - Encryption mechanisms like AES-GCM can detect malicious bit-flips in memory.  
- **Availability**: TEEs/CVMs do not inherently guarantee availability—cloud providers can still suspend or terminate the VM. Availability relies on provider SLAs, redundancy, etc.

### Performance Overhead
- Enabling confidential computing (CVMs, enclaves) typically introduces a small overhead (1–5%, sometimes up to 10% in heavy I/O scenarios).  
- Often acceptable in scenarios dealing with highly sensitive data (finance, healthcare, secure ML, etc.).

### Remote Attestation & Supply Chain
- Remote attestation offers cryptographic proof of what’s running inside the environment (e.g., OS, binaries).  
- Useful for verifiable build runners (CI/CD): you can store an attestation report proving your build toolchain is unmodified.  
- Strengthens the software supply chain by ensuring no tampering with the build environment.

### Trusted Platform Modules (TPMs) vs. Hardware Security Modules (HSMs)
- **TPMs**:
  - Relatively inexpensive; widely available on modern PCs.  
  - Very limited storage (a few keys), perform basic crypto operations (signing, sealing).  

- **HSMs**:
  - More expensive, typically used in large enterprises or regulated industries.  
  - Perform cryptographic operations on dedicated hardware; keys never leave the secure device.  
  - Often have anti-tampering mechanisms to protect secrets if physically attacked.

## Offerings in Confidential Computing
### Capability Level
* Direct use of chip-level CC features via OS/hardware interfaces.
* Example: Using Intel SGX instructions directly in your code via the Intel SGX SDK.

### SDKs
* Lower-level libraries or language-specific toolkits that enable CC features for developers.
* Example: The Open Enclave SDK that abstracts away hardware-specific details for TEE development.

### Platform Offerings
* Packaged solutions (VM capabilities, container/Kubernetes integrations) that incorporate CC capabilities under the hood.
* Example: Azure Confidential VMs with AMD SEV, which let you run standard workloads while getting memory encryption automatically.

### Packaged CC Offerings
* Specialized CC applications, products, or services (ISVs focused on confidential computing).
* Example: A secure machine-learning platform that processes encrypted training data in enclaves.

### Packaged Non-CC Offerings
* General-purpose applications or services that might use CC features from any of the above but are not strictly CC-focused.
* Example: A CRM platform that "optionally" uses enclaves for especially sensitive customer data but does not exclusively rely on them.

## Ideal requirements for a Trustworthy TCB
A cornerstone of Confidential Computing (CC) is having a well-defined trust model, which mandates a fully traceable and attestable TCB. The challenge: many CC vendors provide large and complex TCBs, making it difficult or impossible to verify each component.
* **Open Source**: Transparency enables community review and auditing, reducing the likelihood of hidden vulnerabilities.
* **Stability**: Frequent updates make comprehensive security reviews impractical, so stable codebases are preferred.
* **Security Audits**: All TCB components should undergo rigorous auditing to detect and patch vulnerabilities.
* **Attestation**: Each TCB component must be measured and verifiable during attestation, helping confirm that no unauthorized changes exist.

### Example Problem: UEFI/BIOS
Many cloud services provide UEFI/BIOS components for virtual machines, but these often are proprietary, unavailable for external audit, or otherwise excluded from the CC attestation chain. Moreover, "lift-and-shift" approaches—migrating entire VM images without modification—can bloat the TCB, muddying the chain of trust.

### Possible Solutions
* **Minimize the TCB:** Use specially crafted VMs or direct SDK integrations that explicitly include only necessary components.
* **Leverage Attestable Platforms**: Opt for platform or application offerings designed from the ground up to meet open-source, audit, and attestation requirements.

### Availability of First- or Third-Party Attestation
Even if the TCB itself is trustworthy, it must be attested (verified) by a party other than the system operator or cloud provider. Generally, there are two models:

1. **First-Party Attestation**
    * The workload owner runs an attestation server, verifying their own TEEs.
    * Keeps the CSP or hardware operator out of the trust chain.
2. **Third-Party Attestation**
    * A neutral, trusted entity performs the attestation.
    * Minimizes the risk of conflicts of interest and ensures an unbiased security assessment.

**Potential Conflict**: Relying on the CSP’s own attestation service can introduce a conflict of interest. Unless there is a legally separate business unit with its own governance, a single CSP acting as both the platform operator and the attestation authority can undermine the independence required for truly confidential workloads. Consequently, many enterprises prefer to use truly independent attestation—either run by themselves (first-party) or by a trusted, external third-party.

