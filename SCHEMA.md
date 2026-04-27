# SCHEMA

This document codifies the structure of the knowledge base so that future additions don't reintroduce the duplication this site has been refactored to avoid. It defines what each file owns, the cross-link convention, decision rules for new content, and the templates for adding a new cloud provider or topic area.

A one-line-per-file table of contents (for human readers and for LLM sessions) lives in [`index.md`](index.md). This file (`SCHEMA.md`) describes the structural rules; `index.md` describes what's currently in the structure.

## Top-level structure

```
docs/
├── core/         # Cloud-agnostic confidential computing fundamentals
├── containers/   # Container ecosystem in CC context
├── tools/        # Frameworks/tools commonly used alongside CC
└── misc/         # Per-cloud references + cloud-agnostic supplements + reading list
```

Section names are stable. New content fits into one of these four; do not invent new top-level sections without an explicit decision.

## Domain ownership: file-by-file

Each row below states what a file owns ("OWNS") and lists topics that a reader might expect to find here but should look for elsewhere ("NOT here"). When something is "NOT here," there is a cross-link to the canonical home.

### `docs/core/` — cloud-agnostic CC fundamentals

| File | OWNS | NOT here |
|---|---|---|
| `confidential-computing-concepts.md` | Generic CC concepts: CIA goals, TEE/enclave/TCB definitions, generic attestation, generic CVMs, generic TPM-vs-HSM comparison, CC offerings taxonomy. | Vendor-specific TEE detail (→ `sgx.md`, `tdx/`); Azure-specific HSM tiers (→ `docs/misc/microsoft-azure-security.md` Part III). |
| `tee-security-principles.md` | Design checklist for trustworthy TEEs (measure all inputs, attestation freshness, etc.). | Specific TEE implementations. |
| `tees-in-practice.md` | Applied TEE landscape: crypto, AI agents, MCP servers, hybrid TEE+ZK trajectory. | Vendor-specific implementation detail. |
| `sgx.md` | Intel SGX: enclaves, ECalls/OCalls, EDL, attestation APIs, lifecycle, EPC memory protection. | TDX (entirely separate technology, → `tdx/`). |
| `tdx/index.md` | Intel TDX overview: SEAM, TDX Module, TME-MK, Secure EPT, attestation, threat model. | TDX ABI mechanics (→ `tdx/abi.md`); device assignment (→ `tdx/trusted-io.md`); live migration (→ `tdx/live-migration.md`). |
| `tdx/abi.md` | TDX Module binary interface: SEAMCALL/TDCALL, TD_PARAMS, REPORTMACSTRUCT, MBMD, etc. | Conceptual TDX overview (→ `tdx/index.md`). |
| `tdx/trusted-io.md` | TDISP, SPDM, IDE for TDX-attached devices. | NVIDIA-vendor-specific GPU CC (→ `gpu-confidential-computing.md`). |
| `tdx/live-migration.md` | TDX 1.5 Migration TDs, MSKs, post-copy memory phase. | Generic live migration (→ `live-migration.md`). |
| `paravisor.md` | Paravisor architecture, OpenHCL, VMGS, supporting unmodified guests inside CVMs. | The CVM technologies themselves (→ `tdx/`, generic SEV-SNP context elsewhere). |
| `gpu-confidential-computing.md` | NVIDIA H100 / Blackwell GPU-CC: FSP/GSP/SEC2/CEs, CPR, SPDM session keys, BAR0 Decoupler, attestation, known gaps. | Standards-based trusted I/O (→ `tdx/trusted-io.md`); CPU-side CVM mechanics (→ `tdx/`). |
| `secure-key-release.md` | Cross-cloud SKR concept: KMS roles, RATS background-check model, environment assertion, encrypted-filesystem pattern. | Azure-specific KMS tiers and the full SKR-on-Azure flow (→ `docs/misc/microsoft-azure-security.md` §§16–17). |
| `confidential-ai.md` | Running AI workloads inside TEEs: memory encryption story, end-to-end attested inference, OHTTP-based confidential inferencing example. | Generic CVM concepts (→ `confidential-computing-concepts.md`); GPU specifics (→ `gpu-confidential-computing.md`); Azure-specific MAA/THIM detail (→ `docs/misc/microsoft-azure-security.md` Part III). |
| `live-migration.md` | Generic live migration: pre-copy/post-copy/hybrid, brownout/blackout, residual dependencies, why CVM migration is harder. | TDX-specific migration (→ `tdx/live-migration.md`). |

### `docs/containers/` — container ecosystem in CC context

| File | OWNS | NOT here |
|---|---|---|
| `containers.md` | Container fundamentals: namespaces, cgroups, OCI, Docker architecture, OverlayFS, runtime taxonomy, Hyper-V/Kata isolation. | Specific Kata details (→ `kata-containers.md`); CoCo-specific (→ `confidential-containers.md`). |
| `kata-containers.md` | Kata Containers and Kata Confidential Containers: per-pod micro-VMs, attestation flow, CoCo on AKS, security policy via OPA. | Generic container basics (→ `containers.md`). |
| `confidential-containers.md` | CoCo project as a whole; Azure Container Instances confidential containers (LCOW + AMD SNP, ephemeral disk encryption, CCE policy). | The Kata-specific implementation (→ `kata-containers.md`). |
| `kubernetes.md` | Kubernetes in the CC context: per-cluster vs per-pod TEE, etcd secrets, multi-tenant trust, Helm fundamentals. | Container runtime details (→ `containers.md`). |

### `docs/tools/` — frameworks/tools used alongside CC

| File | OWNS | NOT here |
|---|---|---|
| `grpc.md` | RPC concepts, marshalling, naming, failure semantics. | gRPC's specific role inside CoCo (→ `containers/kata-containers.md` etc., when relevant). |
| `open-policy-agent.md` | OPA introduction: Rego, Kubernetes admission, API auth, config validation. | OPA used inside Kata's policy enforcement (→ `containers/kata-containers.md`). |

### `docs/misc/` — per-cloud references + cloud-agnostic supplements

Each cloud occupies **two slots**: an Overview, and a single consolidated Security Architecture note that walks the full security story end-to-end in five Parts. Microsoft Azure is currently the only cloud covered.

| File | OWNS | NOT here |
|---|---|---|
| `microsoft-azure-overview.md` | What Azure is as a cloud platform: cloud-as-utility model, compute/storage/networking/databases/AI taxonomy, Resource Manager, history of renames, what's gone. | Anything security/trust/encryption-specific (→ `microsoft-azure-security.md`). |
| `microsoft-azure-security.md` | The single consolidated Azure security note. Five Parts, §§1–22:<br>**Part I (§§1–9) — Trust Chain:** silicon catalog (canonical for **CVM SKU families** and the **SKU naming convention**), hardware roots of trust (Cerberus, Pluton, anti-rollback), UEFI, Secure Boot, **measured boot + TPM 2.0 + PCRs (canonical)**, code integrity / HVCI / VBS / dm-verity, hashes (SHA family), **Hyper-V architecture (canonical)**, confidential VMs/GPUs.<br>**Part II (§§10–11) — Infrastructure & Tenant Isolation:** datacenter geography, two-network split, Fabric Controller, operator access (MCIO, JIT, SAW/PAW); customer's grouping hierarchy, compute isolation spectrum, "trust grows monotonically" TCB table, network isolation, per-service isolation.<br>**Part III (§§12–17) — Encryption & Key Management:** host attestation, MAA, THIM, envelope encryption, encryption at rest and in transit, double encryption, **5-way HSM decision (canonical for Marvell LiquidSecurity)**, Secure Key Release on Azure.<br>**Part IV (§§18–20) — Shared Responsibility & Customer Data:** responsibility matrix, defense-in-depth services (**Trusted Launch canonical**, Defender, Sentinel, Entra ID + PIM, JIT VM access), customer data protection (DPA, **Customer Lockbox canonical**, residency vs. sovereignty, NIST 800-88 hardware disposition).<br>**Part V (§§21–22) — Through-Lines and Practical Takeaways.** | The Azure product taxonomy and history (→ `microsoft-azure-overview.md`); cross-cloud RATS framing of SKR (→ `docs/core/secure-key-release.md`); generic CC concepts (→ `docs/core/confidential-computing-concepts.md`). |
| `cloud-fundamentals.md` | Cloud-agnostic supplementary topics: autoscaling, background jobs. | Anything Azure- or CC-specific. |
| `scratch.md` *(Reading List & Resources)* | External links — videos, papers, blog posts, tools, books, courses, vendor docs, researcher pages. | Original explanations (those go in topic-specific files). |

## Cross-link convention

The repo has one canonical home per concept; everywhere else it appears, there must be a brief summary that ends in a link to the canonical home. The link must use Jekyll's `{{ site.baseurl }}` prefix so it works under the GitHub Pages base path.

### Link syntax

For sibling/cross-section links, always:

```markdown
[link text]({{ site.baseurl }}/docs/<section>/<file-without-.md>/)
```

For deep-linking into the merged `microsoft-azure-security.md`, append a heading anchor:

```markdown
[link text]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#anchor-id)
```

Examples:

- `[Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/)`
- `[the 5-way HSM decision]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#16-choosing-a-key-manager-the-five-way-decision)`
- `[Intel TDX]({{ site.baseurl }}/docs/core/tdx/)`
- `[Secure Key Release]({{ site.baseurl }}/docs/core/secure-key-release/)`

The trailing slash matters — paths use Jekyll `permalink: pretty` style. Do not use `.md` extensions in links between rendered pages. Anchor IDs are the lowercase-hyphenated form of the heading text (e.g., `## 16. Choosing a Key Manager: The Five-Way Decision` → `#16-choosing-a-key-manager-the-five-way-decision`).

### When to link out vs. duplicate

If you are tempted to redefine a concept that already has a canonical home in another file, write a 1–2 sentence summary instead and link to the canonical home. The summary must:

1. Carry just enough context for the surrounding paragraph to make sense.
2. Not redefine, re-list, or re-elaborate facts that live elsewhere.
3. End with a link to the canonical location using `{{ site.baseurl }}`.

Example of a good summary-with-link:

> "The Azure SKU families that ship these technologies (and the matching confidential GPU SKUs) are catalogued in §9 of the [Microsoft Azure Security Architecture]({{ site.baseurl }}/docs/misc/microsoft-azure-security/#9-where-the-trust-chain-ends--confidential-vms-and-gpus) note."

Example of a *bad* summary-with-link (redefines what's at the link):

> "Azure ships AMD SEV-SNP on DCasv5 / ECasv5 SKUs and Intel TDX on DCesv6 / ECesv6 SKUs (5th-gen 'v5' / 6th-gen 'v6' generations, 'C' for confidential, 'as' for AMD, 'es' for Intel). See [Azure security]({{ site.baseurl }}/...)."

The bad version restates the catalog content alongside the link — that's the duplication this convention exists to prevent.

### Internal cross-links inside a single file

Anchors within the same file (`#section-id`) are allowed and don't need `{{ site.baseurl }}`. Inside the merged `microsoft-azure-security.md`, prefer prose section references ("§9", "see Part III") over bare anchor links — they survive heading-text edits.

## Where new content should go

Use this decision tree before writing a new heading or new file.

1. **Is it cloud-agnostic and conceptual** (TEEs, attestation, key release flows, AI patterns, container fundamentals, RPC concepts)? → `docs/core/`, `docs/containers/`, or `docs/tools/`.
2. **Is it specific to a cloud platform** (Azure, AWS, GCP, …)? → one of that cloud's two slots in `docs/misc/`. The Overview slot owns "what the platform is"; the Security Architecture slot owns everything trust/security/encryption/responsibility-related.
3. **Is it a generic cloud-engineering supplement** (autoscaling, queueing, etc.) not tied to confidential computing? → `docs/misc/cloud-fundamentals.md`.
4. **Is it an external link or reading recommendation**? → `docs/misc/scratch.md` (Reading List).

### Per-cloud structure: two slots, one consolidated security note

Every cloud provider gets **two files** under `docs/misc/`:

| Slot | File pattern | Belongs here |
|---|---|---|
| **1. Overview** | `<cloud>-overview.md` | What the platform is, product taxonomy, history. |
| **2. Security Architecture** | `<cloud>-security.md` | The full security story, organized as five Parts: (I) Platform Trust Chain — silicon, RoT, UEFI, Secure Boot, measured boot, code integrity, hashes, hypervisor, confidential VMs/GPUs; (II) Infrastructure & Tenant Isolation — datacenter hierarchy, networks, control-plane orchestrator, operator access, isolation choices; (III) Encryption & Key Management — host attestation, attestation collateral service, encryption at rest and in transit, double encryption, KMS/HSM lineup, secure key release; (IV) Shared Responsibility & Customer Data — responsibility matrix, defense-in-depth services, customer data protection commitments; (V) Through-Lines and Practical Takeaways. |

The consolidated file uses **continuous numbered sections** (§§1–22 in Azure's case) and **horizontal-rule Part dividers** so readers can scan for the part of the security story they need without losing the through-line.

### Anti-rules (these create duplication; don't)

- Do not split the Security Architecture into multiple files. The whole point of the consolidation is that trust → isolation → encryption → responsibility is one narrative.
- Do not duplicate the SKU catalog. CVM SKU families and the SKU naming convention live exclusively in §9 of `<cloud>-security.md`.
- Do not redefine envelope encryption, the 5-way HSM table, the SKR flow, the hypervisor partition model, measured-boot mechanics, the shared-responsibility matrix, or Customer Lockbox outside their canonical homes. Reference them with a link.
- Do not add a "Comparison" page that re-tabulates content across Parts. The comparison should live as cross-references in prose at the section that owns the topic.

## How to add a new cloud provider

To add (say) AWS or GCP coverage, mirror the Azure pattern:

1. **Create the two per-cloud files** under `docs/misc/`:
   - `aws-overview.md` *(nav_order assigned to keep Azure first; e.g., 5)*
   - `aws-security.md` *(next nav_order; e.g., 6)*

   Reuse the same internal Part structure as Azure. When a third cloud is added and the file count grows, either continue with prefixed flat files or switch all clouds to subfolders (`docs/misc/azure/`, `docs/misc/aws/`, `docs/misc/gcp/`) — but keep the **two-slot** structure unchanged inside each cloud. Do not split clouds across multiple top-level sections.

2. **Each slot's frontmatter** uses `parent: Reference` and a fresh nav_order; titles follow the Azure pattern (e.g., `AWS Overview`, `AWS Security Architecture`).

3. **Internal Part structure of `<cloud>-security.md`** matches Azure's: continuous section numbering across Parts I–V; horizontal-rule Part dividers; canonical homes preserved (§9 for CVM SKUs, §§12–13 for attestation, §16 for KMS, §17 for SKR, §19 for Trusted Launch, §20 for Customer Lockbox). Adapt section *titles* and content to the cloud, but keep the structural shape so cross-cloud comparisons remain easy.

4. **Cross-cloud common concepts** (envelope encryption *as a concept*, attestation roles, RATS model, the SKR flow shape) live in `docs/core/` and are linked from the per-cloud notes. Do not re-derive them inside `<cloud>-security.md`.

5. **Update the cross-references**:
   - Add the two new files to [`index.md`](index.md) (one line per file, with the Part-summary bullets for the security note).
   - Add the new files to the file-by-file table in this `SCHEMA.md`.
   - Update `README.md`'s "What's Covered" section if the new cloud changes the high-level pitch.
   - Update `docs/misc/index.md`'s description.

6. **Verify**: `bundle exec jekyll build` should run cleanly. Spot-check a few cross-links (especially anchor links into `<cloud>-security.md`).

7. **Update [`CHANGELOG.md`](CHANGELOG.md)** with the new cloud's files and any moves.

## How to add a new topic area

Most additions are not a new cloud — they're a new sub-topic. Use this process:

1. **Decide if it's already covered.** Search `index.md` and the SCHEMA table for the topic. If a file already owns it, add to that file (often a new subsection of `<cloud>-security.md`).
2. **Pick the canonical home.** If genuinely new, the home is determined by the decision tree above (cloud-agnostic vs. cloud-specific; conceptual vs. operational; which Part of `<cloud>-security.md` owns the slice).
3. **Write the canonical treatment in one place.** Do not pre-emptively cross-link from elsewhere; let cross-references emerge as they're needed.
4. **When another file would benefit from referencing it**, add a 1–2 sentence summary at the referencing site that links to the canonical home using `{{ site.baseurl }}`, with an anchor when the target is inside a long file.
5. **Update [`index.md`](index.md)** with a one-line entry for the new file (or amend the existing file's entry if you only added a section).
6. **Update [`SCHEMA.md`](SCHEMA.md)** if the new topic's domain affects the file-ownership table — particularly the "NOT here" column of nearby files.

## How to verify the schema is being followed

Quick checks an LLM session or human reviewer can run:

- `grep -rn "DCasv5\|DCesv6\|NCC.*H100" docs/` should return matches only in `docs/misc/microsoft-azure-security.md`.
- `grep -rn "envelope encryption\|DEK.*KEK" docs/` should land overwhelmingly in the same file.
- `grep -rn "Customer Lockbox" docs/` should land canonically in `microsoft-azure-security.md` (Part IV); other matches should be 1–2 sentence references.
- `grep -rn "Marvell LiquidSecurity" docs/` should land canonically in `microsoft-azure-security.md` §16.
- All cross-section links should use `{{ site.baseurl }}/docs/...` and a trailing slash. Quick check: `grep -rn "site.baseurl" docs/` should show the canonical pattern.
- `bundle exec jekyll build` should complete with no errors.

If any of these checks surface unexpected duplication, the dedupe pattern in [`CHANGELOG.md`](CHANGELOG.md) (the "Concept → canonical home → trimmed appearances" tables) is the model to follow when fixing it.
