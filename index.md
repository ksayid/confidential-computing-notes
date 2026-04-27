---
title: Home
nav_order: 0
---

A living notebook on trusted execution, secure containers, and cloud security.

For the structure that governs *where* content lives and *how* to add to it without reintroducing duplication, see [`SCHEMA.md`](SCHEMA.md). For the history of structural changes, see [`CHANGELOG.md`](CHANGELOG.md).

## Sections

- **[Core Concepts]({{ site.baseurl }}/docs/core/)** — cloud-agnostic CC fundamentals
- **[Container Technologies]({{ site.baseurl }}/docs/containers/)** — containers in the CC context
- **[Tools & Frameworks]({{ site.baseurl }}/docs/tools/)** — tools used alongside CC
- **[Reference]({{ site.baseurl }}/docs/misc/)** — per-cloud references + supplements

## Index of all content pages

One line per file, in reading order.

### Core Concepts — `docs/core/`

- **[Confidential Computing Concepts]({{ site.baseurl }}/docs/core/confidential-computing-concepts/)** — CIA goals, TEE/enclave/TCB definitions, generic attestation, generic CVMs, generic TPM-vs-HSM comparison, CC offerings taxonomy.
- **[TEE Security Principles]({{ site.baseurl }}/docs/core/tee-security-principles/)** — design checklist for trustworthy TEEs (measure all inputs, attestation freshness, complementary techniques like RA-TLS, threat-model questions for builders and buyers).
- **[TEEs in Practice]({{ site.baseurl }}/docs/core/tees-in-practice/)** — applied TEE landscape: crypto infrastructure (Coinbase, Flashbots), AI agents and MCP servers, x402 + TEEs + attestation, hybrid TEE+ZK trajectory.
- **[Intel SGX]({{ site.baseurl }}/docs/core/sgx/)** — process-level enclaves: ECalls/OCalls, EDL, Open Enclave attestation APIs, EPC memory protection, enclave lifecycle.
- **[Intel TDX]({{ site.baseurl }}/docs/core/tdx/)** — VM-level CC: Trust Domains, SEAM, TDX Module, TME-MK encryption, Secure EPT, attestation, threat model. Index page; sub-pages below.
  - **[Intel TDX — ABI Reference]({{ site.baseurl }}/docs/core/tdx/abi/)** — TDX Module binary interface: SEAMCALL/TDCALL, TD_PARAMS, REPORTMACSTRUCT, MBMD, page lifecycle states, migration bundles.
  - **[Intel TDX — Trusted I/O (TDISP)]({{ site.baseurl }}/docs/core/tdx/trusted-io/)** — bringing devices (GPUs, NICs, accelerators) into the TD trust boundary using PCI-SIG TDISP, SPDM, and IDE.
  - **[Intel TDX — Live Migration]({{ site.baseurl }}/docs/core/tdx/live-migration/)** — TDX 1.5 cross-platform migration: Migration TDs, Migration Session Keys, AES-GCM-256 bundles, post-copy memory phase.
- **[Paravisor: Secure Execution in Confidential Computing]({{ site.baseurl }}/docs/core/paravisor/)** — paravisor architecture, OpenHCL framework, VMGS file, supporting unmodified guests inside CVMs.
- **[NVIDIA GPU Confidential Computing]({{ site.baseurl }}/docs/core/gpu-confidential-computing/)** — H100/Blackwell GPU-CC: FSP/GSP/SEC2/Copy Engines, Compute Protected Region, SPDM session keys, BAR0 Decoupler, attestation flow, known gaps.
- **[Secure Key Release]({{ site.baseurl }}/docs/core/secure-key-release/)** — cross-cloud SKR concept: KMS roles (attester / KBS / verifier / KMS), RATS background-check model, environment assertion, encrypted-filesystem pattern, ACI sidecar example.
- **[Confidential AI]({{ site.baseurl }}/docs/core/confidential-ai/)** — running AI workloads inside TEEs: memory encryption, end-to-end attested inference, OHTTP-based confidential inferencing example, AES-GCM IV considerations.
- **[Live Migration]({{ site.baseurl }}/docs/core/live-migration/)** — generic live migration: pre-copy/post-copy/hybrid, brownout/blackout phases, residual dependencies, dirty tracking, why CVM migration is harder.

### Container Technologies — `docs/containers/`

- **[Containers]({{ site.baseurl }}/docs/containers/containers/)** — container fundamentals: namespaces, cgroups, OCI specs, Docker architecture, OverlayFS, runtime taxonomy, Hyper-V vs. Kata isolation models.
- **[Kata Containers]({{ site.baseurl }}/docs/containers/kata-containers/)** — Kata Containers and Kata Confidential Containers: per-pod micro-VMs, attestation flow, AKS CoCo deep dive, OPA-based security policy.
- **[Confidential Containers]({{ site.baseurl }}/docs/containers/confidential-containers/)** — CoCo project; Azure Container Instances confidential containers (LCOW + AMD SNP, ephemeral disk encryption, CCE policy, verifiable execution policies).
- **[Kubernetes]({{ site.baseurl }}/docs/containers/kubernetes/)** — Kubernetes in the CC context: per-cluster vs. per-pod TEE, etcd secrets, multi-tenant trust, Helm fundamentals.

### Tools & Frameworks — `docs/tools/`

- **[gRPC]({{ site.baseurl }}/docs/tools/grpc/)** — RPC concepts: function-call semantics across machines, marshalling, naming services, failure semantics.
- **[Open Policy Agent]({{ site.baseurl }}/docs/tools/open-policy-agent/)** — OPA: Rego, Kubernetes admission control, API authorization, configuration validation.

### Reference — `docs/misc/`

#### Microsoft Azure (per-cloud bundle: Overview + Security Architecture)

- **[Microsoft Azure Overview]({{ site.baseurl }}/docs/misc/microsoft-azure-overview/)** — what Azure is: cloud-as-utility model; compute, storage, networking, databases, AI taxonomies; Azure Resource Manager; rebrandings and retirements.
- **[Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/)** — the consolidated security note. Five Parts running continuously across §§1–22:
  - *Part I — The Platform Trust Chain* (§§1–9): silicon catalog (canonical for **confidential VM SKU families and the SKU naming convention**), hardware roots of trust (Cerberus, Pluton, anti-rollback), UEFI, Secure Boot, **measured boot + TPM 2.0 + PCRs** (canonical), code integrity / HVCI / VBS / dm-verity, hashes (SHA family), **Hyper-V architecture** (canonical), confidential VMs and GPUs.
  - *Part II — Infrastructure and Tenant Isolation* (§§10–11): datacenter geography, two-network split, Fabric Controller, operational access (MCIO, JIT, SAW/PAW); customer's grouping hierarchy, compute isolation spectrum, "trust grows monotonically" TCB table, network isolation, per-service isolation (Storage/SQL/App Service/AKS).
  - *Part III — Encryption and Key Management* (§§12–17): host attestation, MAA, THIM; envelope encryption; encryption at rest and in transit; double encryption; 5-way HSM decision (canonical for **Marvell LiquidSecurity HSM details**); Secure Key Release on Azure.
  - *Part IV — Shared Responsibility and Customer Data* (§§18–20): shared-responsibility matrix; defense-in-depth services (Entra ID + PIM, Defender for Cloud, Sentinel, Azure Firewall/DDoS/WAF, **Trusted Launch** canonical, JIT VM access); customer data protection (DPA, **Customer Lockbox** canonical, residency vs. sovereignty, NIST 800-88 hardware disposition, government data requests).
  - *Part V — Through-Lines and Practical Takeaways* (§§21–22).

#### Cloud-agnostic supplements

- **[Cloud Fundamentals]({{ site.baseurl }}/docs/misc/cloud-fundamentals/)** — autoscaling (vertical vs. horizontal, design considerations) and background jobs (event-driven and schedule-driven). Not CC-specific.
- **[Reading List & Resources]({{ site.baseurl }}/docs/misc/scratch/)** — external links: videos, papers, blog posts, tools/repos, books/courses, vendor docs, researcher pages.

## Repo-root files

- **[`README.md`](README.md)** — project overview, local development setup, contribution guide.
- **[`SCHEMA.md`](SCHEMA.md)** — structural rules: file ownership, cross-link convention, where new content goes, how to add a new cloud or topic.
- **[`CHANGELOG.md`](CHANGELOG.md)** — history of structural changes (restructuring + deduplication passes).
- **[`CLAUDE.md`](CLAUDE.md)** — instructions for AI assistants working in this repo.
