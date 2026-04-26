---
title: Microsoft Azure Security Architecture
parent: Reference
nav_order: 4
---

Three Microsoft Learn documents — on Azure infrastructure, encryption best practices, and double encryption — describe a single layered system. The infrastructure docs define **where the trust boundaries sit**: between the public production network and Microsoft's corporate network, between the hypervisor and a customer's guest operating system, between the orchestration layer and the hardware it manages, between Microsoft personnel and customer data. The encryption docs describe **what protects data when those boundaries are crossed or distrusted**: encryption at rest in storage, encryption in transit on the wire, and key management that decides who can unlock what. Double encryption answers a sharper question: **what if a single boundary, algorithm, or key custodian fails?** This note synthesizes all three into one map.

## 1. The Infrastructure Stack and Its Trust Boundaries

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

### Per-node architecture: the hypervisor as the load-bearing boundary

Each node runs a **type-1 hypervisor** — virtualization software that runs directly on bare metal rather than on top of a host operating system — derived from Hyper-V. Above it sit three kinds of operating-system images:

- **Host OS** — a hardened Windows Server variant in the privileged root partition. It mediates all disk and network access on behalf of guest VMs and is not publicly accessible.
- **Native OS** — runs directly on storage tenants (e.g., Azure Storage), without a hypervisor.
- **Guest OS** — runs inside the customer's VM.

Microsoft's hypervisor-security doc states explicitly that *"machine boundaries are enforced by the hypervisor, which doesn't depend on the operating system security."* This is the load-bearing claim of Azure tenant isolation: if you trust the hypervisor and the host OS, you trust that one customer's VM cannot reach another's. The hypervisor itself only earns that trust because it is launched on top of a verified boot chain — silicon root of trust, UEFI Secure Boot, code integrity, measured boot — covered in the [platform trust chain]({{ site.baseurl }}/docs/misc/microsoft-azure-platform-trust-chain/) note. Confidential computing — Intel TDX, AMD SEV-SNP, Intel SGX — exists precisely because some customers do not want to extend that trust to the hypervisor or host OS at all (see [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/)).

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

That last point is worth pausing on: even Microsoft's own infrastructure uses key separation and encryption-at-rest internally. The same patterns the encryption-best-practices doc recommends to customers are applied to the platform itself.

### Operational access: just-in-time, not persistent

Layered on top of the technical boundaries are organizational ones. Azure separates several operational roles, each with bounded access:

- **MCIO** owns physical buildings, perimeter networking, and bare-metal setup. Customers never interact with them.
- **Service teams** own the engineering of each service and are on-call 24/7. **No physical hardware access by default.**
- **Datacenter engineers** handle physical security and hardware break-fix. **No customer data access.**
- **Incident triage, deployment, and live-site engineers** may touch customer data, but only via **just-in-time access** — the privilege is granted for a bounded window and audited; persistent access is limited to non-customer systems.

Operators connect from **Secure Admin Workstations (SAWs)**: dedicated, hardened machines with administrative accounts kept separate from regular user accounts. (Customers can mirror this pattern with **Privileged Access Workstations**, or PAWs. In Microsoft's own usage SAW and PAW are largely interchangeable, with PAW sometimes reserved for the highest-privilege tier.)

The takeaway from Section 1: the platform has well-defined boundaries between networks, between tenants on a node, between operational roles, and between Microsoft personnel and customer data. The encryption story below is layered on top of these boundaries — not in place of them.

## 2. Encryption at Each Layer

### Three states of data

Encryption protects data in three states:

| State | Meaning | Typical defenses |
|---|---|---|
| **At rest** | Static data on physical media | AES-256 at the storage cluster |
| **In transit** | Data moving over a network | TLS, IPsec VPN, MACsec at the link layer |
| **In use** | Data being processed in memory | Confidential VMs (AMD SEV-SNP, Intel TDX, Intel SGX) — addressed elsewhere in this repo, not in the encryption-best-practices doc |

The encryption docs cover at-rest and in-transit thoroughly, but treat in-use only conceptually — pointing at hardware-managed keys without naming the technologies. The rest of this repo (see [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/), [TDX Architecture]({{ site.baseurl }}/docs/core/tdx-architecture/), [SGX]({{ site.baseurl }}/docs/core/sgx/)) goes deeper on data in use.

### Key management: the look-alike services

Several Azure services handle keys, and they sound similar but differ in important ways. An **HSM** — hardware security module — is a tamper-resistant device that performs cryptographic operations and never lets the private key material leave the device in plaintext. **FIPS 140** is the U.S. government standard that certifies how robustly an HSM resists tampering; Levels 2 and 3 differ mainly in physical-tamper response (Level 3 zeroizes keys when intrusion is detected). FIPS 140-3 is the current revision; FIPS 140-2 will be moved to NIST's historical list in late 2026.

| Service | What it actually is |
|---|---|
| **Azure Key Vault (Standard)** | Software-protected key and secret store. Multi-tenant. |
| **Azure Key Vault Premium** | Same API as Standard, but keys are stored in a shared HSM. New keys created on "HSM Platform 2" are FIPS 140-3 Level 3; older Platform 1 keys are FIPS 140-2 Level 2. |
| **Azure Managed HSM** | Single-tenant, customer-controlled HSM pool. FIPS 140-3 Level 3. The customer owns the *security domain* — a cryptographic backup of the HSM partition; lose it, lose every key forever. Use when you need a customer-owned root of trust. |

The lineup widens to five options once Azure Cloud HSM (lift-and-shift IaaS HSM) and Azure Payment HSM (PCI-HSM-certified) are included, and the connection to confidential computing — Secure Key Release that gates key handover on attestation — is covered in the [hardware trust and key management]({{ site.baseurl }}/docs/misc/microsoft-azure-hardware-trust-and-keys/) note.

Two related but distinct concepts are worth disentangling:

- **BYOK (Bring Your Own Key)** — the customer generates the key in their own on-prem HSM and imports it into Azure. Describes *how the key got there*.
- **Platform-managed key (PMK) vs Customer-managed key (CMK)** — describes *who controls the key in production*. PMKs are created, rotated, and used transparently by Microsoft. CMKs live in the customer's Key Vault and **wrap** (encrypt) the service's data encryption key (DEK); revoking a CMK effectively destroys access to the data because the DEK can no longer be unwrapped.

### At rest, by layer

- **Storage Service Encryption (SSE)** — always on, AES-256, applied at the storage cluster. Cannot be disabled.
- **Managed disks** — three distinct mechanisms that often get confused:
  - *Server-side encryption (SSE for disks)* — always on at the storage cluster.
  - *Encryption at host* — the VM's host server encrypts data before it leaves to storage, covering temp disks and VM caches that storage-side SSE alone misses. Microsoft positions this as the recommended option for new VMs.
  - *Azure Disk Encryption (ADE)* — older, runs inside the guest using BitLocker (Windows) or dm-crypt (Linux). **Scheduled for retirement on September 15, 2028**; after that date, ADE-encrypted disks will fail to unlock on VM reboot. Migrate to encryption at host.
- **Database — Transparent Data Encryption (TDE)** — default for Azure SQL Database, Managed Instance, and Synapse. Encrypts page-level database files using a database encryption key (DEK), which is itself wrapped by a TDE protector. With CMK, the protector is a 2,048- or 3,072-bit RSA key in Key Vault or Managed HSM (4,096-bit is not supported for TDE).

Algorithms and standards: **AES-256** for data at rest across Storage, disks, and SQL; **RSA / RSA-HSM** for key wrapping; **RSA-OAEP-256 is the recommended wrapping algorithm**. Plain RSA-OAEP defaults to SHA-1 for its hash and mask-generation functions and is treated as legacy because of known weaknesses in SHA-1.

### In transit

- **HTTPS** for all portal traffic and Storage REST API calls.
- **Site-to-site VPN** — connects an on-prem network to an Azure virtual network using IPsec.
- **Point-to-site VPN** — connects a single workstation to a virtual network.
- **ExpressRoute** — a dedicated private circuit between an on-prem datacenter and Azure. Notably, ExpressRoute traffic **is not encrypted by default**; MACsec is available on ExpressRoute Direct as an opt-in customer-configured feature, and the encryption-best-practices doc recommends layering TLS or IPsec on top.

The doc recommends TLS but does not pin a minimum version. Azure Storage will require TLS 1.2 or higher starting February 3, 2026 (TLS 1.0 and 1.1 are being retired); TLS 1.3 is also supported, though it cannot yet be enforced as the minimum for Storage accounts.

### Key rotation: a subtlety

Rotating a **key encryption key (KEK)** — the upper-level key that wraps DEKs — **does not re-encrypt the underlying data**. It re-wraps the DEKs with the new KEK version. Both the old and new KEK versions must remain enabled until the re-wrap completes.

For suspected key compromise, **order matters**:

1. Rotate to a new key.
2. Reconfigure dependent services to use the new key.
3. *Then* disable or delete the old key.

Disabling first breaks the dependent services and leaves DEKs still wrapped by the compromised key — so an attacker holding the old key material can still decrypt.

This subtlety is exactly the kind of operational detail the [Secure Key Release]({{ site.baseurl }}/docs/core/secure-key-release/) note explores in the broader context of attestation-gated keys.

## 3. Double Encryption: When One Layer Is Not Enough

### Threat model

The encryption layers above defend against the obvious threats: a stolen disk, a wiretapped fiber, a misplaced credential. Double encryption defends against three sharper failure modes:

- **Configuration error** in one encryption layer.
- **Implementation bug** in a single algorithm.
- **Compromise** of a single key.

Independence between the two layers is the load-bearing requirement: **separate keys, separate key hierarchies, separate operators, ideally different algorithms**. If both layers shared a key or algorithm, a single break would unwind both, and the protection would be cosmetic.

### At rest: two layers

| Layer | What it is | Key holder |
|---|---|---|
| **Service-level** | Encryption at rest using a CMK in Key Vault (customer's choice of BYOK or generated). | Customer |
| **Infrastructure-level** | A second, lower layer using a PMK with a *different* algorithm and a *separate* Microsoft-managed key. Always on by default for services that support it. | Microsoft |

For Azure Storage specifically, the infrastructure-encryption toggle **must be enabled at storage account creation** — it cannot be turned on later. This is a forward-looking decision, not a recoverable one.

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

## 4. The Through-Line

Reading the three docs together, several connections become explicit:

- The **hypervisor boundary** (Section 1) is what separates a customer's guest OS from Azure's host OS. **Encryption at host** (Section 2) crosses this boundary, encrypting data on the host before it leaves for storage.
- The **just-in-time operator access model** (Section 1) and **CMK with revocation** (Section 2) are complementary: Microsoft personnel cannot persistently access customer data, and the customer holds a kill switch on the keys that decrypt it.
- The **MACsec encryption between datacenters** (Section 3) is the encryption inside the inter-datacenter network mentioned in passing in Section 1.
- The **Fabric Controller's own use of encryption** for application images and hardware credentials (Section 1) shows that the patterns recommended to customers — encryption at rest, key separation, restricted custodians — are applied internally to the platform itself.
- **Confidential computing** fills the gap the encryption docs leave: data in use. Encryption at rest and in transit assumes the host is trusted; trusted execution environments (TEEs) remove that assumption. See [Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/) and [TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/).

## 5. Practical Takeaways

**Defaults to keep on:**
- Storage SSE (cannot meaningfully disable anyway).
- TDE for Azure SQL.
- Encryption at host for VM disks.

**When to use CMK:**
- You need revocation power.
- You need separation of duties between key custodians and database administrators.
- You need to satisfy compliance requirements such as FedRAMP High or PCI-DSS key control.

**When to use Managed HSM (instead of Key Vault Premium):**
- You need a single-tenant, customer-controlled root of trust.
- You accept the operational burden of guarding the security domain.

**Wrapping algorithm:**
- Use RSA-OAEP-256. Avoid plain RSA-OAEP (SHA-1 based).

**On suspected compromise:**
- Rotate first, reconfigure, *then* disable the old key.

**Operator hygiene:**
- Use Privileged Access Workstations for subscription owners and admins. An endpoint compromise bypasses everything else.

**Monitoring:**
- Configure Key Vault access logs and Azure Rights Management Service (RMS) usage logging.

## Sources

- [Azure information system components and boundaries](https://learn.microsoft.com/en-us/azure/security/fundamentals/infrastructure-components)
- [Data security and encryption best practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/data-encryption-best-practices)
- [Double Encryption in Microsoft Azure](https://learn.microsoft.com/en-us/azure/security/fundamentals/double-encryption)
- [What are Azure availability zones?](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
- [Hypervisor security on the Azure fleet](https://learn.microsoft.com/en-us/azure/security/fundamentals/hypervisor)
- [Server-side encryption of Azure managed disks](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption)
- [Migrate from Azure Disk Encryption to encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-migrate)
- [Managed HSM and Key Vault Premium FIPS 140-3 Level 3 announcement](https://techcommunity.microsoft.com/blog/microsoft-security-blog/azure-managed-hsm-and-azure-key-vault-premium-are-now-fips-140-3-level-3/4418975)
- [About encryption for Azure ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-encryption)
- [Enable infrastructure encryption for double encryption of data](https://learn.microsoft.com/en-us/azure/storage/common/infrastructure-encryption-enable)
- [Enforce a minimum required version of TLS for Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/transport-layer-security-configure-minimum-version)
- [IEEE 802.1AE — MACsec (Wikipedia)](https://en.wikipedia.org/wiki/IEEE_802.1AE)
