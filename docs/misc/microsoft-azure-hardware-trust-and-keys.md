---
title: Microsoft Azure Hardware Trust, Encryption, and Key Management
parent: Reference
nav_order: 7
---

Eight documents — Wikipedia on the System-on-a-Chip, Microsoft Learn on measured boot and host attestation, Trusted Hardware Identity Management (THIM), the encryption-overview / encryption-at-rest / Storage Service Encryption pages, and the two key-management pages (the decision guide and the customer-managed-key support matrix) — describe a single chain. Where the [platform trust chain]({{ site.baseurl }}/docs/misc/microsoft-azure-platform-trust-chain/) note covers the *vertical* boot path from silicon up to hypervisor, this note follows the *consequences* of that chain: how Azure verifies a host is healthy, how it serves the certificates and revocation lists that hardware vendors publish for attestation ("attestation collateral"), how encryption is layered on top of a host that has passed those checks, and how customers choose where keys live and how they are released. The connecting idea is that every encryption guarantee Microsoft makes ultimately rests on the platform's own integrity — the trust chain is what makes that guarantee credible.

## 1. The Silicon Layer: System-on-a-Chip and the Root of Trust

A **System-on-a-Chip (SoC)** integrates most or all components of a computing system — CPU cores, memory controller, I/O interfaces, accelerators, power management, and security blocks — onto a single die. This contrasts with the older multi-chip model where a CPU, memory controller, and southbridge (the I/O hub on PC motherboards) sat in separate packages connected by board-level buses. Modern server chips (AMD EPYC, Intel Xeon, Microsoft's Azure Cobalt 100) are SoCs in this sense.

The implication for trust: when the security processor — typically a Trusted Platform Module (TPM, the standardized chip that holds keys and measures boot state) or its equivalent — is co-integrated with the CPU on the same die, the **hardware root of trust (RoT)** is silicon physically next to the cores it protects, not a discrete chip on the motherboard talking over a bus that can be sniffed or replaced. This is what makes Project Cerberus on Azure motherboards (Microsoft's open-source RoT controller, described in the [platform trust chain]({{ site.baseurl }}/docs/misc/microsoft-azure-platform-trust-chain/) note) and the on-die security blocks on AMD and Intel server SoCs cryptographically meaningful as anchors.

### Server SoCs in the Azure fleet

- **AMD EPYC** — integrates the Platform Security Processor (PSP), an ARM Cortex-A5 secure core. AMD's confidential-VM technology, **SEV-SNP** (Secure Encrypted Virtualization with Secure Nested Paging — encrypts and integrity-protects guest memory from the host), runs on Azure's DCasv5 / ECasv5 SKUs (Microsoft's product naming for VM size families).
- **Intel Xeon Scalable** — integrates the Converged Security and Management Engine (CSME). It supports two confidential-computing technologies: **SGX** (Software Guard Extensions, which carves small encrypted "enclaves" inside a process) on DCsv2 / DCsv3, and **TDX** (Trust Domain Extensions, which encrypts entire VMs the way SEV-SNP does on AMD). The TDX general-availability family is DCesv6 / ECesv6 (announced GA on 5th-Gen Xeon in West US and West US 3 and rolling outward); DCesv5 / ECesv5 reached preview only.
- **Microsoft Cobalt 100** — Azure's first in-house ARM-based server SoC, 128 cores on Arm Neoverse N2. VMs went GA in October 2024 and are deployed across roughly 32 Azure regions and growing. A successor, Cobalt 200 (132 Neoverse-V3 cores on TSMC 3 nm), was announced in late 2025 with broader rollout in 2026.
- **Caliptra** — an open-source silicon root-of-trust IP block (not a chip; a reusable hardware design dropped into an SoC), founded in 2022 by AMD, Google, Microsoft, and NVIDIA, now a CHIPS Alliance project. Caliptra 2.0 added post-quantum primitives (NIST FIPS 204 ML-DSA signatures and ML-KEM key exchange via the Adams Bridge accelerator); Caliptra 2.1 is the latest RTL release. **Flag:** Microsoft has called Caliptra a "committed intercept" for first-party cloud silicon and AMD server silicon, but the first chips integrating it are only beginning to appear in 2026, so treat Caliptra as in-adoption rather than ubiquitous on Azure production hosts.

## 2. Measured Boot, Briefly

With the silicon root of trust in place, the next layer is recording *what actually ran on top of it*. The [platform trust chain]({{ site.baseurl }}/docs/misc/microsoft-azure-platform-trust-chain/) note covers measured boot in detail; the minimal recap needed here is that the TPM 2.0 starts each boot with empty Platform Configuration Registers (PCRs — small TPM registers that accumulate measurements). Each stage of boot (firmware, bootloader, kernel, drivers, hypervisor) hashes the next stage and *extends* the running PCR value by the rule `PCR_new = hash(PCR_old || measurement)`. The result is a TCG event log (a Trusted Computing Group standardized log of every measurement) plus a set of PCR digests that summarize what actually executed. Each link is recorded *before* it runs, so a compromised later stage cannot rewrite an earlier measurement.

This is the *recording* mechanism. The next layer is the *evaluation* mechanism that decides what those measurements mean.

## 3. Host Attestation: Fleet-Internal Verification

In Azure terminology, **host attestation** is the fleet-internal check that a physical Azure host is in a known-good state *before* it is allowed to receive customer workloads, accept fleet credentials, or talk to the rest of the control plane. It is distinct from **guest (or workload) attestation**, where a workload running inside a Confidential VM proves the state of *its* **trusted execution environment** (TEE — a CPU-enforced isolated runtime, e.g. SGX, SEV-SNP, or TDX) to a relying party. Host attestation is invisible to customers; guest attestation is what they consume.

### What gets measured and what happens on failure

The host attestation service runs inside each Azure cluster in a "specialized locked-down environment" and validates a compliance statement from each host against a Microsoft-defined attestation policy. The Microsoft Learn page enumerates the specific items checked:

- **Secure Boot state** — that UEFI Secure Boot is enabled, plus the contents of the four UEFI key databases: the signature database (db, allowed signers), the revoked-signatures database (dbx), the Key Exchange Key database (KEK, in this UEFI sense — *not* the encryption-key wrapper introduced later in this note), and the Platform Key (PK, the top-level UEFI signing key).
- **Debug controls** — production hosts must boot with kernel debuggers disabled.
- **Code-integrity policy** — which kernel-mode drivers and binaries are signed and trusted.
- Implicit in the PCR set: firmware, bootloader, hypervisor, and kernel measurements.

If any item fails (TCG event-log mismatch, Secure Boot off, debugger enabled, code-integrity violation), the cluster blocks all communication to and from the host and triggers an incident workflow. The host stays isolated until a post-mortem clears it.

### The TPM key hierarchy used in attestation

Three TPM keys do the work in any attestation flow, host or guest:

- **Endorsement Key (EK)** — an asymmetric key burned into the TPM at manufacture, certified by an EK certificate from the TPM vendor. It uniquely identifies one specific TPM. The EK is a key-exchange / decryption key, not a signing key, and is privacy-sensitive (using it directly correlates a machine across requests).
- **Attestation Key (AK)** — formerly called the Attestation Identity Key (AIK). A restricted signing key derived inside the TPM and bound to the EK via a credential-activation protocol. The AK is what actually signs PCR **quotes** (defined next). Multiple AKs can exist as privacy-preserving aliases of one EK.
- **Quote** — a TPM-signed structure containing the requested PCR digests and a caller-supplied **nonce** (a one-time random number used as a freshness challenge that defeats replay attacks), signed by the AK. Verifying a quote tells the relying party that (a) the measurements came from a real TPM, (b) they are bound to that TPM's EK chain, and (c) they are fresh.

### End-to-end flow

1. The host boots; PCRs accumulate measurements through firmware, bootloader, kernel, drivers, and hypervisor.
2. A **host attestation agent** in the Hyper-V root partition (in Hyper-V, the privileged management partition that owns hardware access — controlled by Microsoft, not customer-visible) collects the TCG event log and asks the TPM for an AK-signed quote over the relevant PCRs, with a nonce supplied by the cluster's attestation service.
3. The cluster's host attestation service verifies the quote signature against the host's enrolled AK and EK chain, replays the event log against the PCRs, and compares the result to the cluster's policy.
4. On success, the cluster's PKI issues credentials sealed to the host's PCRs — only that host can unseal them, so a clone or man-in-the-middle cannot reuse them.
5. The host is now allowed to take customer workloads.

### Microsoft Azure Attestation (MAA): the customer-facing service

Customers see a different surface called **Microsoft Azure Attestation (MAA)**. MAA is generally available, with a regional shared provider in every Azure region where confidential computing is offered. It verifies TEE evidence from SGX enclaves, AMD SEV-SNP confidential VMs, Intel TDX confidential VMs, VBS enclaves (Windows Virtualization-Based Security in-process enclaves), and TPM-based platforms, and issues a signed **JSON Web Token (JWT — the JSON-based signed-token format from RFC 7519)**. The token's signing keys are published at an **OpenID Connect (OIDC)** / **JSON Web Key Set (JWKS)** metadata endpoint — the same standards OAuth identity providers use; relying parties fetch the certificate by its key ID (`kid`), verify the signature, and then evaluate the claims. Standard claims include `x-ms-attestation-type`, `x-ms-policy-hash`, platform-specific claims (`x-ms-sevsnpvm-*`, `x-ms-isolation-tee.*`), and `rp_data` — a relying-party-supplied nonce for freshness.

A **relying party** is whichever component consumes the MAA token to make a trust decision: typically Azure Key Vault (deciding whether to release a key), an application gating a secret, Azure Policy evaluating compliance, or Microsoft Defender for Cloud verifying posture.

**Flag:** older docs reference services like the "TVM Service" or "Host Verification Service." Treat these as superseded by the modern MAA + host-attestation split. Likewise, EPID-based SGX attestation (Intel's older Enhanced Privacy ID scheme) on legacy DCsv2 SKUs is being phased out industry-wide in favor of ECDSA-based attestation through Intel's **Data Center Attestation Primitives (DCAP)** library and THIM (next section).

## 4. Trusted Hardware Identity Management: The Collateral Layer

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
- **Azure Key Vault Premium and Managed HSM Secure Key Release (SKR)** — transitive consumer; SKR validates an MAA token, and MAA in turn relied on THIM to validate the underlying quote.

For a compact producer/consumer/verifier view across quote, event log, collateral, token, policy, and key release, see [Artifact Map](#8-artifact-map).

**Flag:** the documented endpoint is global (`global.acccache.azure.net`); per-region URLs are not published, and the docs do not describe regional failover behavior beyond the in-host IMDS cache. Treat THIM as a global anycast surface backed by a per-host cache.

## 5. Encryption Layered on the Trusted Foundation

Once a host has cleared host attestation and is running with valid platform identity, the next layer up is the encryption applied to data on that host. Microsoft's three encryption pages (`encryption-overview`, `encryption-atrest`, `storage-service-encryption`) describe a uniform model: every Azure **resource provider** (the API surface that implements a given service — Storage, SQL, Cosmos DB, etc.) applies encryption at rest the same way, and the customer's choice is mostly *who controls the key*.

### Envelope encryption: the universal pattern

Azure uses **envelope encryption** almost everywhere — a two-tier scheme where one key encrypts data and a second key encrypts the first key. The two key types:

- **Data Encryption Key (DEK)** — a symmetric AES-256 key that encrypts a partition or block of data. A single resource has many DEKs; per-block separation limits the blast radius of a compromised key.
- **Key Encryption Key (KEK)** — wraps (encrypts) the DEKs. Lives in Azure Key Vault or Managed HSM and ideally never leaves it. (In the encryption literature this is sometimes also called the **Customer Master Key (CMK)** when the customer owns it.)

Envelope encryption is used for performance — bulk crypto needs DEKs local to the service, but a Key Vault round-trip per I/O operation would be prohibitive. The KEK gives a single revocation point: disabling the KEK cryptographically erases all dependent DEKs, since a wrapped DEK cannot be unwrapped. Wrapped DEKs are persisted as metadata alongside the encrypted data.

A subtle but important detail: when a service caches a DEK in memory for active operations, those cached keys are protected by host-level platform isolation. **Key Vault is the root of trust for cold keys, but warm DEKs live in host memory** — and the integrity of those warm DEKs reduces back to the SoC root of trust, measured boot, host attestation, and (for confidential workloads) TEE-enforced isolation. This is the explicit handoff point between the trust chain and the encryption stack.

### What's always on, no customer action

| Service | Default behavior | Algorithm |
|---|---|---|
| Azure Storage (Blob, File, Queue, Table) — Storage Service Encryption (SSE) | On, cannot be disabled | AES-256 in **GCM** (Galois/Counter Mode — an authenticated mode that produces both ciphertext and an integrity tag) |
| Managed disks, snapshots, images | Server-side encryption with platform-managed keys | AES-256 (mode unspecified in current Microsoft docs — flag) |
| Azure SQL Database (created since June 2017) | **Transparent Data Encryption (TDE — encrypts the database files; transparent to applications)** on by default | AES-256 |
| Azure Cosmos DB (non-volatile storage) | Service-managed encryption, default | AES-256 |
| Temp / ephemeral OS disks on Vv5+ VMs | Auto-encrypted via encryption at host | AES-256 |

Worth flagging: **temp disks on pre-v5 VM SKUs are not encrypted unless encryption at host is on**. This is a real gap many customers miss. **Azure Disk Encryption (ADE)** — the older guest-side option that uses dm-crypt on Linux and BitLocker on Windows — is **scheduled for retirement on September 15, 2028**; after that date, ADE-encrypted VMs will fail to unlock on reboot, so ADE workloads need to migrate to encryption at host before then.

### What customers can configure

- **Customer-managed keys (CMK).** The KEK lives in Azure Key Vault Premium or Managed HSM. The customer holds it; the service generates and keeps the DEK. Most managed-disk and SQL CMK uses RSA 2048, 3072, or 4096 bits; Synapse and SQL TDE want RSA 3072 specifically; Data Lake Store is pinned to RSA 2048.
- **Customer-provided keys (CPK).** Specific to Azure Blob Storage. The customer sends the encryption key on each REST request; Azure never persists it. A SHA-256 hash of the key is persisted alongside the blob so subsequent requests can verify the same key is supplied. **Side effect:** Microsoft Defender for Storage cannot scan CPK-encrypted blobs, since it has no key.
- **Encryption scopes** (Azure Storage, generally available). A scope is a key boundary smaller than a storage account: a default scope per container, with per-blob override. Each scope independently chooses Microsoft-managed or customer-managed keys, so a single storage account hosting multiple tenants can keep one tenant's blobs unreadable to another by giving each tenant its own scope and KEK.
- **Confidential disk on a Confidential VM.** Disk encryption keys are bound to the VM's **virtual TPM (vTPM — a TPM exposed to the guest by the hypervisor)**, and keys are released to the VM only after platform attestation succeeds. Confidential OS-disk encryption is supported only for OS disks below 128 GB (a real constraint for lift-and-shift migrations of larger system disks). This is the path that closes the loop with Section 7 below.
- **Encryption at host.** Encrypts temp disks and OS / data disk caches *on the VM host* before data flows to Storage, using no guest CPU. Replaces ADE and is Microsoft's recommended option for new VMs.

### Infrastructure encryption: the always-on second layer

Azure Storage offers an optional second AES-256 pass underneath the service-layer encryption, using a separate Microsoft-managed key and a different mode. The infrastructure layer is always Microsoft-managed (a customer cannot CMK the bottom layer), and **must be enabled at storage-account creation** — it cannot be retro-enabled. Microsoft asserts the two layers use different algorithms, but the second layer's specifics are not published.

### Algorithm and FIPS status

- **Azure Storage server-side encryption** uses AES-256-GCM (explicitly stated by Microsoft).
- **Storage client-side encryption v1** used AES in **CBC (Cipher Block Chaining)** mode and is deprecated due to an implementation vulnerability in that mode; v2 uses AES-GCM.
- **Managed disk SSE** is documented only as "256-bit AES, FIPS 140-2 compliant"; the mode is not stated by Microsoft. Don't assume **XTS** (the tweakable AES mode standard for full-disk encryption, used by BitLocker) just because BitLocker uses it.
- **FIPS 140-2 (the older NIST cryptographic-module standard) moves to NIST's "historical" list on September 21, 2026.** Modules already validated to 140-2 may keep operating, but new validations after that date must be FIPS 140-3 — the successor standard.
- **Azure Managed HSM and Azure Key Vault Premium are now FIPS 140-3 Level 3 validated** on the new "HSM Platform 2" hardware (Marvell LiquidSecurity 2, certified June 2024). Older Platform 1 keys remain at FIPS 140-2 Level 2 until rolled.

## 6. Choosing a Key Manager: The Five-Way Decision

Once the CMK pattern is in place, the next question is *where the customer-held KEK actually lives*. The existing [security architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/) note covers Key Vault Standard, Premium, and Managed HSM at a high level. Microsoft's current decision page widens the lineup to five options. The previously prominent Azure Dedicated HSM is no longer in the comparison table — the IaaS-style HSM role is now filled by **Azure Cloud HSM** (general availability in 2025, also on Marvell LiquidSecurity), and Microsoft routes on-prem-HSM migrations there.

| Option | Tenancy | FIPS level | Primary role |
|---|---|---|---|
| **Key Vault Standard** | Multi-tenant PaaS | FIPS 140-2 Level 1 (software) | Cheap software vault for keys, secrets, certificates |
| **Key Vault Premium** | Shared HSM, PaaS | FIPS 140-3 Level 3 (Platform 2 keys); FIPS 140-2 Level 2 (legacy Platform 1) | HSM-backed keys without a dedicated HSM cost |
| **Azure Key Vault Managed HSM** | Single-tenant HSM pool, PaaS | FIPS 140-3 Level 3, PCI DSS, PCI 3DS | Customer-controlled HSM with key sovereignty; **keys only**, no secrets or certificates |
| **Azure Cloud HSM** | Single-tenant, IaaS-style | FIPS 140-3 Level 3 | Full **PKCS#11** / **JCA** / **CNG** support (the standard cryptographic APIs used by C, Java, and Windows clients respectively); on-prem HSM migration, Oracle TDE, code signing |
| **Azure Payment HSM** | Single-tenant, IaaS-style | FIPS 140-2 Level 3, PCI HSM v3, PCI PTS HSM v3 | Payments only — PIN block, EMV, 3DS, card auth |

Premium, Managed HSM, and Cloud HSM all run on **Marvell LiquidSecurity** HSMs (LiquidSecurity 1 and 2; the LS2 family received its FIPS 140-3 Level 3 validation in June 2024). Payment HSM runs on a separate model certified to PCI HSM v3.

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

## 7. Secure Key Release: Closing the Loop

Section 1 introduced silicon-rooted hardware identity. Section 6 chose where the key lives. **Secure Key Release (SKR)** is the mechanism that ties the two ends together by gating key release on a fresh attestation token.

For a compact producer/consumer/verifier view across all moving parts in this flow, see [Artifact Map](#8-artifact-map).

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

**Customer Lockbox is unrelated to all of this.** It governs how Microsoft support engineers obtain temporary access to customer data; it does not interact with Key Vault, CMK, or SKR. The two are sometimes confused because both involve "approving access to something sensitive."

## 8. Artifact Map

This section normalizes the SKR attestation pipeline into concrete artifacts and trust decisions.

| Artifact | Producer | Consumer(s) | Primary verifier |
|---|---|---|---|
| **Quote** (TEE evidence: SNP report / TDX or SGX quote) | TEE platform + quoting component inside the workload environment | MAA (direct), optionally customer-side verifier | MAA quote-verification pipeline (using vendor collateral) |
| **Event log** (TPM/TCG boot measurements, where applicable) | Boot chain + attestation agent on the measured host/VM | Host attestation service and any verifier that replays PCRs | Host attestation service / verifier replay logic |
| **Collateral** (PCK/VCEK chains, CRLs, TCB Info, QE identity) | Intel PCS / AMD KDS upstream, cached and served by THIM | MAA, DCAP/QPL-based verifiers, az-dcap-client | Verifier certificate-path and revocation checks; TCB policy logic |
| **Token** (MAA-signed JWT with claims, nonce, policy hash) | Microsoft Azure Attestation (MAA) | Key Vault Premium / Managed HSM SKR endpoint; relying-party apps | JWT signature validation (JWKS/OIDC metadata) + claim checks |
| **Policy** (SKR release policy / attestation policy hash expectation) | Key owner / security admin | Key Vault Premium / Managed HSM | Policy engine in HSM + relying-party comparison against token claims |

Sequence-style flow:

- Workload asks TEE to produce a **quote** bound to a fresh nonce.
- Workload submits quote to **MAA**.
- MAA fetches/uses **collateral** (via THIM) and verifies quote authenticity + TCB status.
- MAA issues signed **token** (JWT) containing claims including nonce (`rp_data`) and policy hash claim(s).
- Workload presents token to **SKR** (Key Vault Premium or Managed HSM).
- SKR evaluates **policy** against token claims.
- If policy matches, wrapped key is released and unwrapped in the TEE.

Failure points (and what breaks):

- **Expired collateral** → quote cannot be validated to current revocation/TCB state; MAA/verifier rejects attestation, so no token or token marked unusable for SKR.
- **Nonce mismatch** → freshness check fails (`rp_data` does not match challenge); replay resistance triggers rejection before key release.
- **Policy hash mismatch** → token's `x-ms-policy-hash` (or equivalent expected claim) differs from configured release policy expectation; SKR denies release.

## 9. The Through-Line

Two observations are worth pulling out of the layered narrative:

- **Key management is a sovereignty choice, not a security choice.** All five HSM options are cryptographically strong; the meaningful differences are who holds the security domain, who patches, where authentication lives, and which compliance regime applies.
- **The trust chain ends where the customer chooses it to end.** Default encryption-at-rest with a platform-managed key trusts the entire Azure stack. CMK in Managed HSM removes Microsoft from key custody. Confidential VMs with attestation-gated SKR remove Microsoft from runtime key access too. See the [security model]({{ site.baseurl }}/docs/misc/microsoft-azure-security-model/) note for how this maps onto shared responsibility.

## 10. Practical Takeaways

**Defaults Azure provides without customer action:**
- Storage Service Encryption with AES-256-GCM, always on, all redundancy tiers, all blob types and access tiers.
- TDE on Azure SQL by default since June 2017.
- Server-side encryption on managed disks with platform-managed keys.
- Encryption at host on temp / ephemeral disks for Vv5+ VM SKUs (older SKUs require explicit enablement).

**Defaults customers should know about:**
- **Soft delete** on Key Vault is on and cannot be disabled. **Purge protection** is *off* by default — turn it on if compliance demands it.
- **ADE retires September 15, 2028.** Migrate ADE-encrypted VMs to encryption at host before then.
- **TLS 1.2 minimum on Azure Storage** is enforced globally on February 3, 2026.
- **Infrastructure encryption** for Storage must be enabled at account creation; you cannot retro-enable it.

**When to choose each key manager:**
- **Key Vault Standard** for prototypes and low-stakes secrets.
- **Key Vault Premium** for cloud-native production workloads needing HSM-backed keys without sovereignty concerns.
- **Managed HSM** for regulated workloads (banking, PCI DSS) where the customer must hold the security domain.
- **Cloud HSM** for lift-and-shift IaaS migrations from on-prem HSMs (full native PKCS#11, JCA, CNG).
- **Payment HSM** if and only if you are doing PCI HSM-level payment processing.

**To break the chain on purpose:**
- Use a **Confidential VM** (DCasv5 / ECasv5 on AMD SEV-SNP, DCesv6 / ECesv6 on Intel TDX) with a confidential OS disk, and attach the disk to a key in Managed HSM with an SKR policy that requires a fresh MAA token. The DEK is unwrapped inside the TEE only when the platform attests as compliant. See [Secure Key Release]({{ site.baseurl }}/docs/core/secure-key-release/) and [GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/) for the broader pattern.

**Operational hygiene:**
- Treat **TCB recovery events** as scheduled work — when Intel or AMD bump SVNs, plan to re-attest workloads using SKR. THIM's slower baseline buys time but doesn't eliminate the need.
- Keep **Managed HSM security-domain quorum private keys** on separate encrypted media in separate physical locations. Loss is unrecoverable.
- **Monitor Key Vault and Managed HSM data-plane logs** to Log Analytics. Without that, no forensic trail.

## Sources

- [System on a chip — Wikipedia](https://en.wikipedia.org/wiki/System_on_a_chip)
- [Firmware measured boot and host attestation](https://learn.microsoft.com/en-us/azure/security/fundamentals/measured-boot-host-attestation)
- [Trusted Hardware Identity Management](https://learn.microsoft.com/en-us/azure/security/fundamentals/trusted-hardware-identity-management)
- [Encryption overview](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-overview)
- [Azure data encryption at rest](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-atrest)
- [Azure Storage Service Encryption for data at rest](https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption)
- [How to choose the right key management solution](https://learn.microsoft.com/en-us/azure/security/fundamentals/key-management-choose)
- [Azure services that support customer-managed keys](https://learn.microsoft.com/en-us/azure/security/fundamentals/encryption-customer-managed-keys-support)
- [Azure Attestation overview](https://learn.microsoft.com/en-us/azure/attestation/overview)
- [Secure Key Release with Azure Key Vault and Confidential Computing](https://learn.microsoft.com/en-us/azure/confidential-computing/concept-skr-attestation)
- [About the security domain in Managed HSM](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/security-domain)
- [Caliptra — open-source silicon root of trust](https://github.com/chipsalliance/Caliptra)
- [Azure Cobalt 100-based VMs are now generally available](https://azure.microsoft.com/en-us/blog/azure-cobalt-100-based-virtual-machines-are-now-generally-available/)
- [Azure Managed HSM and Key Vault Premium are now FIPS 140-3 Level 3](https://techcommunity.microsoft.com/blog/microsoft-security-blog/azure-managed-hsm-and-azure-key-vault-premium-are-now-fips-140-3-level-3/4418975)
- [Migrate from Azure Disk Encryption to encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-migrate)
- [Announcing general availability of Azure Intel TDX confidential VMs (DCesv6 / ECesv6)](https://techcommunity.microsoft.com/blog/azureconfidentialcomputingblog/announcing-general-availability-of-azure-intel%C2%AE-tdx-confidential-vms/4495693)
- [Announcing Cobalt 200: Azure's next cloud-native CPU](https://techcommunity.microsoft.com/blog/azureinfrastructureblog/announcing-cobalt-200-azure%E2%80%99s-next-cloud-native-cpu/4469807)
- [Caliptra 2.1 RTL release — CHIPS Alliance](https://www.chipsalliance.org/news/caliptra2-1/)
- [FIPS 140-3 Transition Effort — NIST CSRC](https://csrc.nist.gov/projects/fips-140-3-transition-effort)
- [Azure Storage TLS 1.0/1.1 retirement, Feb 3 2026](https://techcommunity.microsoft.com/blog/azurestorageblog/tls-1-0-and-1-1-support-will-be-removed-for-new--existing-azure-storage-accounts/4026181)
- [Azure Cloud HSM — Microsoft Azure](https://azure.microsoft.com/en-us/products/cloud-hsm)
