---
title: Confidential Containers
layout: default
---

[← Back to Main Page]({{ "/" | relative_url }})

* TOC
{:toc}


# Confidential Containers

Confidential Containers extend familiar container workflows with hardware-based
protection for data **in use**. Instead of running directly on a shared host
kernel, each container—or group of containers—executes inside its own
lightweight virtual machine backed by a Trusted Execution Environment (TEE).
This ensures the container's memory is encrypted and isolated from the host
operating system and hypervisor, thwarting snooping by even highly privileged
attackers.

By combining the portability of containers with the security of confidential
computing, platform operators can offer stronger guarantees around privacy and
integrity. Remote attestation allows workloads to prove they are running in an
approved environment before secrets or keys are released. The result is a
familiar container experience with defense-in-depth protections well suited for
sensitive or multi-tenant scenarios.

## Overview (CoCo)
Confidential Containers (CoCo) integrates Kubernetes with confidential computing primitives to run each pod inside a confidential VM. It builds on Kata Containers or AWS Firecracker, but adds memory encryption and remote attestation to ensure stronger isolation for each container. The result is the regular "pod" interface, but behind the scenes each workload gets hardware-level isolation.

## Key Building Blocks

* **Hardware isolation** – Technologies such as AMD SEV-SNP and Intel TDX
  encrypt guest memory and provide attestation reports that prove the VM's
  integrity.
* **Container runtimes** – Projects like Kata Containers and the `cc-runtime`
  launch pods inside small VMs to take advantage of the underlying TEE
  capabilities.
* **Policy enforcement** – Integrations with tools like Open Policy Agent (OPA)
  let operators define fine-grained rules for what a confidential pod may do
  once running inside the TEE boundary.

## Azure Container Instances
### Key Technology
#### LCOW (Linux Containers on Windows)
* What is LCOW? LCOW allows running Docker containers on Windows by hosting a minimal Linux VM (often called a “utility VM”) under the covers. This VM runs the actual Linux containers, providing a real Linux kernel and userland.
* Why it Matters
  * LCOW can provide an isolated Linux environment on a Windows host.
  * Combining LCOW with confidential computing features (e.g., AMD SNP) can enable “confidential containers” that run inside an encrypted environment.
* What is ContainerPlat?  
    * ContainerPlat is the umbrella term for the components enabling container workloads on Windows hosts. It includes ContainerD and various custom integrations that support running containers in Windows VMs (WCOW) or Linux VMs (LCOW). These capabilities come together to provide a smooth developer experience when running container workloads on Windows, with LCOW specifically offering a real Linux kernel environment in a lightweight VM (the “utility VM”).


#### AMD Secure Nested Paging (SNP)
* What is AMD SNP? AMD SNP is a feature of AMD’s EPYC processors (starting with 3rd Gen Milan) that encrypts a VM’s memory and provides attestation.
  * Memory Encryption: The memory of the VM is encrypted by hardware.
  * Attestation: The hardware can produce an attestation report demonstrating the VM’s integrity and configuration.
* Why it Matters? 
  * Ensures data in the VM is protected from outside threats, including a potentially malicious hypervisor or root user on the host.
  * Provides cryptographic proof that the VM’s memory contents have not been tampered with.

### High-level Design
* **Why Combine LCOW and AMD SNP?**  
  By running Linux containers inside an LCOW utility VM that is protected by AMD SNP, we get both the flexibility of a real Linux environment and the hardware-level memory encryption and attestation guarantees that SNP provides. This fusion yields truly “confidential containers” that are functionally equivalent to regular containers, but with in-use data protection and cryptographic evidence of integrity.

1. LCOW + AMD SNP
  * LCOW provides a lightweight Linux VM for running containers.
  * AMD SNP provides memory encryption and attestation for that VM.
2. Ephemeral Disk Encryption
  * A scratch read-write disk is protected by a combination of dm-crypt and dm-integrity attached via SCSI.
  * The dm-crypt key is created inside the TEE/VM and is not preserved across VM lifecycles (ephemeral storage).
  * OverlayFS can use this ephemeral storage for the top writable layer in a container’s root filesystem.
3. Result
  * You get a Linux environment (“utility VM”) that is both fully functional for containers and protected by hardware-based encryption.
  * The ephemeral nature of the disk ensures that sensitive data won’t persist once the VM/container is torn down.

  * **Additional Ephemeral Storage Insights**  
  - The ephemeral scratch disk encryption key is generated by trusted code inside the Utility VM (UVM).  
  - The encryption key is not shared externally and is destroyed at the end of the VM’s life.  
  - This design ensures writes to the container’s OverlayFS top layer remain confidential and are never exposed to untrusted parties or persisted across sessions.


### Remote guest attestation
Confidential containers on ACI provide support for remote guest attestation which is used to verify the trustworthiness of your container group before creating a secure channel with a relying party. Container groups can generate an SNP hardware attestation report, which is signed by the hardware and includes information about the hardware and software. This generated hardware attestation report can then be verified by the Microsoft Azure Attestation service via an open-source sidecar application or by another attestation service before any sensitive data is released to the TEE.

* **Attestation Workflow Details**  
  - The TEE can provide a cryptographic measurement of what code is running.  
  - Before secrets (e.g., encryption keys) are released, a remote party can validate the TEE’s measurement to ensure it matches the expected code.  
  - This “challenge and response” mechanism forms the basis of trust for confidential computing, guaranteeing that data is only provided to a verified secure environment.

---

## Confidential Computing Enforcement (CCE) Policy

A **Confidential Computing Enforcement (CCE)** policy provides code integrity for the container group. The policy specifies:

- **Container image**  
- **Environment variables**  
- **Any volume mounts**  
- **Container-specific privileges**

When the container group starts up, if any of these properties do not match the defined policy, the deployment fails to prevent sensitive data from being exposed. You can generate this policy from the Azure CLI `confcom` extension using a user-specified ARM template. The policy is injected into the ARM template before container group deployment.

## In-Use Data (Memory and CPU) Protections

All data **in use** (in memory and CPU registers) is protected by a hardware-based Trusted Execution Environment (TEE) running on AMD SEV-SNP hardware. Azure Container Instances (ACI) will refuse to start the container unless the hardware confirms, via cryptographic signing, that it is providing SEV-SNP protections for confidential computing to the container. ACI supports verifiable execution policies that allow customers to verify the integrity of their workloads and help prevent untrusted code from running.

## Key Capabilities of Confidential Containers

- **Lift-and-shift** existing standard container images with **no code changes** into a TEE environment.  
- Extend or build new applications that have confidential computing awareness.  
- Remotely challenge the runtime environment for cryptographic proof that states what was initialized, as reported by the secure processor.  
- Provide strong assurances of data confidentiality, code integrity, and data integrity in a cloud environment with hardware-based confidential computing offerings.  
- Help isolate your containers from other container groups/pods, as well as from the VM node OS kernel.

## When to Use Confidential VMs vs. Confidential Containers

You might deploy your solution on **Confidential VMs** if:

1. You have legacy applications that **cannot be modified or containerized**, yet you need memory protection while data is being processed.  
2. You are running **multiple applications requiring different operating systems** on a single piece of infrastructure.  
3. You want to emulate an **entire computing environment**, including OS resources.  
4. You are **migrating existing VMs** from on-premises to Azure.

Conversely, you might consider **Confidential Containers** when you have containerized workloads or can containerize your application and want to benefit from hardware-based isolation at the container level, rather than the full VM scope.

## Hardware-Based TEE in ACI

Confidential containers on Azure Container Instances run within a **Hyper-V isolated TEE**. This TEE includes a memory encryption key that is generated and managed by an AMD SEV-SNP capable processor. Data in use in memory is encrypted with this key to help protect against data replay, corruption, remapping, and aliasing-based attacks. This hardware-based approach adds a layer of security by ensuring that even Azure operators with elevated privileges cannot inspect or modify data in memory.

## Verifiable Execution Policies

Confidential containers on Azure Container Instances can use **verifiable execution policies** to control what software and actions are allowed within the TEE. These policies:

- Are authored by the customer using provided tooling.  
- Contain cryptographic proofs that are checked by the TEE before allowing the workload to run.  
- Help prevent unexpected application modifications that could potentially leak or compromise sensitive data.  

By enforcing these verifiable execution policies, customers maintain control over their container security posture while leveraging the convenience and scalability of Azure Container Instances.

## Containerization Background

Further, container definitions adhering to the OCI (Open Containers Initiative)  
Distribution Specification specify dependencies as layers that can be shared  
between different containers, making them amenable to caching and thus speeding  
up deployment while reducing storage costs for multiple containers.

The success of containerization technology for on-premises systems has led  
to major cloud providers developing their own CaaS (containers as a service)  
offerings, which provide customers with the ability to maintain and deploy  
containers in the public cloud. In CaaS offerings, containers run in a per-group  
UVM (utility virtual machine), which provides hypervisor-level isolation  
between containers running from different tenants on the same host. While the  
container manager and container shim running on the host are responsible for  
pulling images from the container registry, bringing up the UVM, and  
orchestrating container execution, an agent running in the UVM (the guest agent)  
coordinates the container workflow as directed by the host-side container shim.
<script src="{{ '/assets/js/dark-mode.js' | relative_url }}"></script>
