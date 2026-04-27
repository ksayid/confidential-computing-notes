# Changelog

This pass restructured file boundaries and reading order across the repo so that each file owns a clear, non-overlapping domain and foundational concepts come before things built on them. Paragraph-level deduplication was deferred to a later pass.

## Reading order, before vs. after

**`docs/misc/` (Reference) — before:**
1. Cloud Fundamentals
2. Reading List & Resources (`scratch.md`)
3. Microsoft Azure Overview
4. Microsoft Azure Security Architecture (`microsoft-azure-security.md`)
5. Microsoft Azure Security Model
6. Microsoft Azure Platform Trust Chain
7. Microsoft Azure Hardware Trust, Encryption, and Key Management

**`docs/misc/` (Reference) — after:**
1. Microsoft Azure Overview
2. Microsoft Azure Platform Trust Chain *(silicon → firmware → boot → kernel → drivers → hypervisor)*
3. Microsoft Azure Infrastructure & Tenant Isolation *(was "Security Architecture")*
4. Microsoft Azure Hardware Trust, Encryption, and Key Management
5. Microsoft Azure Security Model: Shared Responsibility & Customer Data
6. Cloud Fundamentals
7. Reading List & Resources

The new order puts platform trust and the hypervisor boundary before encryption and key management, in line with the principle that lower layers must be trustworthy before guarantees built on them are credible.

**`docs/core/` — nav_order updates** to group conceptual notes first, then specific TEE technologies, then cross-cutting flows and use cases:

| File | nav_order before | nav_order after |
|---|---|---|
| `confidential-computing-concepts.md` | 1 | 1 |
| `tee-security-principles.md` | 2 | 2 |
| `tees-in-practice.md` | 9 | 3 |
| `sgx.md` | 3 | 4 |
| `tdx/index.md` | 4 | 5 |
| `paravisor.md` | 5 | 6 |
| `gpu-confidential-computing.md` | 11 | 7 |
| `secure-key-release.md` | 6 | 8 |
| `confidential-ai.md` | 7 | 9 |
| `live-migration.md` | 8 | 10 |

`docs/containers/` and `docs/tools/` were left unchanged.

## File-by-file changes

### `docs/misc/index.md`
- Description rewritten to reflect the new reading order and signal that the section can host other clouds in the future under the same per-cloud pattern.

### `docs/misc/microsoft-azure-overview.md`
- `nav_order` 3 → 1.
- Content unchanged.

### `docs/misc/microsoft-azure-platform-trust-chain.md`
- `nav_order` 6 → 2.
- Title unchanged.
- Content unchanged except: cross-reference to "TDX Architecture" (broken path `tdx-architecture`) updated to "Intel TDX" pointing at `/docs/core/tdx/`.

### `docs/misc/microsoft-azure-security.md`  *(recast)*
- `nav_order` 4 → 3.
- **Title changed** from "Microsoft Azure Security Architecture" to "Microsoft Azure Infrastructure & Tenant Isolation".
- **Section 1 (Infrastructure Stack and Trust Boundaries)** — kept. Two networks, physical-to-logical hierarchy, hypervisor as load-bearing boundary, Fabric Controller, operational access (MCIO, JIT, SAW/PAW).
- **Section 2 (Encryption at Each Layer)** — *removed*; moved to `microsoft-azure-hardware-trust-and-keys.md`. The full encryption-at-rest and encryption-in-transit material (storage SSE, managed disks, ADE retirement, TDE, HTTPS/VPN/ExpressRoute, key rotation order, BYOK vs CMK vs PMK) lives there now.
- **Section 3 (Double Encryption)** — *removed*; moved to `microsoft-azure-hardware-trust-and-keys.md`. The TLS+MACsec layering and "when double encryption makes sense" guidance lives there now.
- **Tenant Isolation (new Section 2)** — *added*; moved in from `microsoft-azure-security-model.md`. Customer grouping hierarchy (tenant/subscription/management group/resource group), the five-rung compute isolation spectrum (multi-tenant VM → Isolated SKU → Dedicated Host → Confidential VM → CVM on Dedicated Host), VNet/NSG/Service Endpoint/Private Endpoint, and the Storage/SQL/App Service/AKS isolation summary, plus decision criteria.
- Through-line and Practical Takeaways rewritten to fit the new scope (datacenter + isolation, not encryption).
- Sources list trimmed to the relevant subset (Azure infrastructure, hypervisor security, isolation choices, dedicated hosts, ASE, DC family, NCC H100).

### `docs/misc/microsoft-azure-hardware-trust-and-keys.md`  *(expanded)*
- `nav_order` 7 → 4.
- Title unchanged.
- **Sections 1–4 (SoC, measured boot, host attestation, THIM)** — kept.
- **Section 5 (Encryption Layered on the Trusted Foundation)** — *expanded*. Pulled in from `microsoft-azure-security.md`:
  - "Three states of data" table (at rest / in transit / in use).
  - "Disks: three mechanisms often confused" subsection (SSE, encryption at host, ADE).
  - Database TDE specifics.
  - Full "In transit" subsection (HTTPS, VPN, ExpressRoute, MACsec on ExpressRoute Direct, TLS 1.2 Storage cutover).
  - "Algorithm and FIPS status" subsection — RSA-OAEP-256 wrapping recommendation, Storage v1 CBC deprecation, FIPS 140-2 → 140-3 transition timeline.
  - "Key rotation: a subtlety" subsection — KEK rotation re-wraps, doesn't re-encrypt; rotate-then-disable order on suspected compromise.
  - "BYOK vs CMK vs PMK" subsection — the distinction between *how the key got there* and *who controls it in production*.
- **Section 6 (Double Encryption)** — *added*; moved in from `microsoft-azure-security.md`. Service-level + infrastructure-level at-rest layers; TLS + MACsec in transit; when double encryption makes sense.
- **Sections 7–8 (Key Manager Five-Way Decision, SKR)** — kept; section numbers shifted by 1.
- **Cross-reference cleanup:** the security-architecture link (now "infrastructure & tenant isolation") and the security-model link both point at the right notes.
- Sources list expanded to include the encryption-best-practices, double-encryption, ExpressRoute, infrastructure-encryption, TLS-minimum, and MACsec sources that previously lived in `microsoft-azure-security.md`.

### `docs/misc/microsoft-azure-security-model.md`  *(recast)*
- `nav_order` 5 (unchanged numerically, but its position shifted because surrounding files moved).
- **Title changed** from "Microsoft Azure Security Model" to "Microsoft Azure Security Model: Shared Responsibility & Customer Data" (clearer scope).
- **Section 1 (Shared Responsibility — The Framework)** — kept.
- **Section 2 (Defense in Depth)** — kept. Network-isolation primitives (VNets, NSGs, Service vs. Private Endpoints, VPN, ExpressRoute) trimmed and replaced with a pointer to the infrastructure note, since that's now their canonical home.
- **Section 3 (Tenant Isolation)** — *removed*; moved to `microsoft-azure-security.md` (now the infrastructure & tenant isolation note). The compute isolation spectrum, hypervisor-mechanics paragraph, network isolation primitives, and Storage/SQL/AppService/AKS isolation all now live there.
- **Section 4 (Customer Data Protection — now Section 3)** — kept. DPA / Microsoft Product Terms, personnel-access defaults, Customer Lockbox, data residency vs. sovereignty, hardware disposition (NIST 800-88), government data requests, subscription termination timeline.
- **Section 5 (Practical Takeaways — now Section 4)** — kept; rewritten to drop tenant-isolation guidance (since that moved out) and clarify the link to confidential computing as a sovereignty/operator-trust choice rather than a tenant-isolation choice.
- Sources list trimmed to the relevant subset (overview, shared responsibility, customer-data protection, Customer Lockbox, DPA, Defender for Cloud, Sentinel, storage redundancy, NIST 800-88).

### `docs/misc/cloud-fundamentals.md`
- `nav_order` 1 → 6. Moved to the end of the Azure-specific chain because it is cloud-agnostic supplementary material rather than part of the primary Azure reading path.
- Content unchanged.

### `docs/misc/scratch.md` (Reading List & Resources)
- `nav_order` 2 → 7.
- Content unchanged.

### `docs/core/*` — nav_order only

| File | nav_order before | nav_order after | Why |
|---|---|---|---|
| `confidential-computing-concepts.md` | 1 | 1 | Foundational. |
| `tee-security-principles.md` | 2 | 2 | Foundational. |
| `tees-in-practice.md` | 9 | 3 | Conceptual/applied — reads better near the principles than at the end. |
| `sgx.md` | 3 | 4 | Specific tech, after the conceptual material. |
| `tdx/index.md` | 4 | 5 | Specific tech (with sub-pages for ABI, Trusted I/O, live migration). |
| `paravisor.md` | 5 | 6 | Builds on confidential VMs introduced in TDX/SEV-SNP. |
| `gpu-confidential-computing.md` | 11 | 7 | Specific tech; grouped with the other TEE techs rather than orphaned at the end. Also closes the gap at nav_order 10. |
| `secure-key-release.md` | 6 | 8 | Cross-cutting attestation→key flow that depends on understanding the underlying TEEs. |
| `confidential-ai.md` | 7 | 9 | Use case; reads after the techs. |
| `live-migration.md` | 8 | 10 | Operational concern; after the techs. |

No content changes in `docs/core/`.

### `docs/containers/`, `docs/tools/`
- Unchanged.

### `README.md`
- Reference-section description rewritten to reflect the new reading order and the cross-cloud growth intent.

### `index.md`
- Reference-section description shortened and aligned with the new scope.

### `CHANGELOG.md`
- New file (this one).

## What was deduplicated structurally vs. what still overlaps

**Deduplicated at file level (one canonical home now):**
- *Encryption at rest, encryption in transit, double encryption* — was in both `microsoft-azure-security.md` and `microsoft-azure-hardware-trust-and-keys.md`; now exclusively in the hardware-trust-and-keys note.
- *Compute isolation spectrum, network isolation, per-service isolation (Storage/SQL/App Service/AKS)* — was in both `microsoft-azure-security-model.md` and `microsoft-azure-security.md`; now exclusively in the security (infrastructure & tenant isolation) note.

**Still overlaps as a paragraph or two (acceptable for this pass):**
- Hypervisor and Hyper-V partition mechanics still surface in three places at different depths: deeply in `microsoft-azure-platform-trust-chain.md` Section 6 (the canonical home); briefly in `microsoft-azure-security.md` (where it explains the load-bearing isolation boundary); and once more in `microsoft-azure-hardware-trust-and-keys.md` Section 5 (where "encryption at host" needs to reference the hypervisor boundary). Each occurrence serves its surrounding argument; consolidating further would obscure the cross-references.
- Silicon root of trust appears in both `microsoft-azure-platform-trust-chain.md` Section 3 (Cerberus/Pluton, the boot-path RoTs) and `microsoft-azure-hardware-trust-and-keys.md` Section 1 (the SoC catalog: AMD EPYC + PSP, Intel Xeon + CSME, Cobalt, Caliptra). The angles are different — boot-path RoTs vs. CPU silicon catalog — but Caliptra in particular is named in both.
- TPM 2.0 / measured boot is covered in `microsoft-azure-platform-trust-chain.md` (Section 3, recording mechanism) and `microsoft-azure-hardware-trust-and-keys.md` (Section 2 recap, Section 3 attestation key hierarchy). The recap is intentional so the attestation flow reads standalone.
- Confidential VM SKU naming (DCasv5/ECasv5, DCesv6/ECesv6, NCC H100 v5) appears in several places — the trust chain note, the infrastructure note, the hardware-and-keys note, and several core notes. Consolidating into a single SKU table is a candidate for a future pass.

## What did not change

- File paths. All files were rewritten in place, not moved or renamed.
- The top-level section names "Core Concepts", "Container Technologies", "Tools & Frameworks", and "Reference".
- `docs/containers/`, `docs/tools/`, `docs/core/` content (only `nav_order` changed in `docs/core/`).
- Site theme, `_config.yml`, build pipeline.

---

# Deduplication pass

This pass implements the dedupe work the previous changelog deferred. For every concept that still appeared in more than one file, the most complete treatment was kept as canonical and every other appearance was replaced with a 1–2 sentence summary linking to the canonical location via `{{ site.baseurl }}`. **No facts were deleted** — anything trimmed from a non-canonical site was first verified to exist in (or moved to) the canonical home.

## Concept → canonical home → trimmed appearances

| Concept | Canonical home | Trimmed (now a brief summary + link) |
|---|---|---|
| **Confidential VM SKU families** (DCasv5 / ECasv5, DCesv5 / ECesv5, DCesv6 / ECesv6) and **confidential GPU SKU** (`NCCadsH100v5`, `Standard_NCC40ads_H100_v5`) | `docs/misc/microsoft-azure-hardware-trust-and-keys.md` Section 1 (the SoC catalog) | `docs/misc/microsoft-azure-platform-trust-chain.md` (Section 6.4 + Practical Takeaways + Sources); `docs/misc/microsoft-azure-security.md` (compute isolation spectrum + Sources) |
| **SKU naming convention** (`DC`/`EC` prefix, `v5`/`v6` generation, trailing `C` for confidential, `as`/`es` for SEV-SNP/TDX, `NCC` for confidential GPU) | `docs/misc/microsoft-azure-hardware-trust-and-keys.md` Section 1 | Was scattered as inline asides in `microsoft-azure-platform-trust-chain.md`; now in one place |
| **Hyper-V architecture** (type-1 hypervisor definition, partition model, VMBus, VSP/VSC, SLAT-based memory partitioning, isolation enforcement mechanisms) | `docs/misc/microsoft-azure-platform-trust-chain.md` Section 6 | `docs/misc/microsoft-azure-security.md` Section 1 (now a 3-sentence summary preserving only the unique fact: the **Native OS** running directly on storage tenants without a hypervisor) |
| **Azure Key Vault Container Types** (Standard / Premium / Managed HSM and the FIPS levels of each) | `docs/misc/microsoft-azure-hardware-trust-and-keys.md` Section 7 (the 5-way decision table: Standard, Premium, Managed HSM, Cloud HSM, Payment HSM) | `docs/core/secure-key-release.md` (the "Azure Key Vault Container Types", "Accessing Keys in Azure Key Vault", "Controlling Access to Keys" sections — collapsed into a single "Azure-specific KMS surfaces" paragraph that links to the canonical table; the Azure-specific operational facts about IMDS-based access tokens and per-environment-vault separation survive there) |
| **Trusted Launch** (Secure Boot + vTPM + boot integrity for Generation-2 Azure VMs; vTPM definition; flag about GA status) | `docs/misc/microsoft-azure-security-model.md` Section 2 (Compute services) | `docs/misc/microsoft-azure-platform-trust-chain.md` Practical Takeaways (now a 1-sentence pointer to the security-model note) |
| **HSM Platform 2 / Marvell LiquidSecurity 2 FIPS 140-3 Level 3 cert (June 2024)** | `docs/misc/microsoft-azure-hardware-trust-and-keys.md` Section 7 (in the "Marvell LiquidSecurity" paragraph below the 5-way table) | `docs/misc/microsoft-azure-hardware-trust-and-keys.md` Section 5 (the duplicate cert-date sentence in the FIPS subsection was removed; readers are pointed to Section 7 for HSM-specific FIPS levels) |

## Concepts checked and left as-is

These showed up in multiple files but each occurrence either covered a different angle or was already a 1-2 sentence reference. No edits needed.

- **Envelope encryption (DEK / KEK / CMK / PMK)** — canonical in `microsoft-azure-hardware-trust-and-keys.md` Section 5. The `KEK` mentions in `microsoft-azure-platform-trust-chain.md` are the *UEFI* Key Exchange Key (a different concept, explicitly disambiguated). Not a duplicate.
- **Measured boot / TPM 2.0 / PCR mechanics** — canonical in `microsoft-azure-platform-trust-chain.md` Section 3 (the "Measured boot and the TPM" subsection). `microsoft-azure-hardware-trust-and-keys.md` Section 2 is explicitly titled "Measured Boot, Briefly" and is a one-paragraph recap with link, which was already the right shape.
- **Customer Lockbox** — canonical in `microsoft-azure-security-model.md` Section 3. References elsewhere (`microsoft-azure-hardware-trust-and-keys.md`, `microsoft-azure-security.md`) are already brief disambiguators with links.
- **Tenant isolation** (mechanisms vs. operational spectrum) — these are two angles on the same area and were already split correctly in the previous pass: `microsoft-azure-platform-trust-chain.md` Section 6 owns the *hypervisor mechanisms* (memory partitioning, vCPU scheduling, I/O brokering); `microsoft-azure-security.md` Section 2 owns the *customer-facing isolation spectrum* (multi-tenant VM → Isolated SKU → Dedicated Host → Confidential VM). No content overlap, just adjacent topics.
- **Silicon root of trust** — different angles in two files: `microsoft-azure-platform-trust-chain.md` Section 3 covers boot-path RoTs (Cerberus, Pluton); `microsoft-azure-hardware-trust-and-keys.md` Section 1 covers the CPU SoC catalog (AMD EPYC + PSP, Intel Xeon + CSME, Cobalt, Caliptra). Caliptra is named in both because it is genuinely relevant to both lenses.
- **`Marvell LiquidSecurity`** — appears twice within `microsoft-azure-hardware-trust-and-keys.md` itself (Section 7's Cloud HSM paragraph and the Marvell paragraph below the 5-way table). Both serve a different purpose (one is "where Cloud HSM runs," the other is the cross-tier hardware story). Internal repetition only; left as is.
- **TPMs vs. HSMs** generic comparison in `docs/core/confidential-computing-concepts.md` is conceptual and cloud-agnostic; not the same scope as the Azure-specific HSM-tier material in hardware-trust-and-keys.
- **AMD SEV-SNP / Intel TDX as confidential-VM technologies** — used as building-block names across many files (core, container, misc). Each use is contextual rather than a definition; the canonical definitions (with full expansions and the SoC-level home) live in `microsoft-azure-hardware-trust-and-keys.md` Section 1 and the dedicated `docs/core/tdx/` folder.

## Per-file edits in this pass

### `docs/misc/microsoft-azure-hardware-trust-and-keys.md`
- Section 1: bolded the SKU family names (DCasv5 / ECasv5, DCesv6 / ECesv6) for visibility now that this is the only place they appear in long form. **Added** a paragraph capturing two facts that previously lived only in `microsoft-azure-platform-trust-chain.md`: the specific confidential GPU SKU example (`NCCadsH100v5`, `Standard_NCC40ads_H100_v5`) and the SKU naming convention (DC/EC prefix, vN generation, trailing C, as/es/NCC suffix codes).
- Section 5 (Algorithm and FIPS status): **removed** the duplicate "Azure Managed HSM and Azure Key Vault Premium are now FIPS 140-3 Level 3 validated on the new HSM Platform 2 hardware (Marvell LiquidSecurity 2, certified June 2024). Older Platform 1 keys remain at FIPS 140-2 Level 2 until rolled" — the cert date and Marvell hardware story already live in Section 7. Replaced with a one-sentence pointer.

### `docs/misc/microsoft-azure-platform-trust-chain.md`
- Section 6 ("Where the hypervisor's trust ends — confidential computing"): **removed** the SKU-naming bullets (DCasv5 / ECasv5, DCesv5 / ECesv5, DCesv6 / ECesv6 expansions). Replaced with one sentence pointing to the hardware-trust-and-keys SKU catalog.
- Section 9 ("Practical Takeaways"): **trimmed** the Confidential VM and Confidential GPU bullets that listed SKU families and the specific `Standard_NCC40ads_H100_v5` example. Replaced with one sentence pointing to the hardware-trust-and-keys catalog and one pointer to the GPU note. **Trimmed** the Trusted Launch bullet (definition + components moved to security-model note).
- Sources: removed duplicate links to "DC family confidential VM sizes" and "NCCadsH100v5 series" (now only in hardware-trust-and-keys).

### `docs/misc/microsoft-azure-security.md`
- Section 1 ("Per-node architecture"): **trimmed** the type-1 hypervisor definition and the Hyper-V partition-model framing. Kept the unique facts (the **Native OS** on storage tenants and the load-bearing-claim quote) and the link to platform-trust-chain.
- Section 2 (compute isolation spectrum): **removed** the paragraph listing DCasv5 / ECasv5 / DCesv5 / ECesv5 / DCesv6 / ECesv6 / NCC H100 v5 SKUs. Replaced with one sentence pointing to the hardware-trust-and-keys catalog.
- Sources: removed duplicate links to "DC family confidential VM sizes" and "NCCadsH100v5 series".

### `docs/core/secure-key-release.md`
- **Removed** the "Azure Key Vault Container Types" section (Vaults Standard/Premium, Managed HSM Standard B1) — duplicated the Section 7 table in hardware-trust-and-keys.
- **Removed** the "Accessing Keys in Azure Key Vault" and "Controlling Access to Keys" sections as standalone headings.
- **Replaced** all three with a single "Azure-specific KMS surfaces" paragraph that links to the hardware-trust-and-keys catalog for the lineup, but preserves the unique operational facts: IMDS-based access-token retrieval via managed identity, REST API invocation with `Bearer` header, list-permission semantics, and the per-environment vault separation hardening pattern.

### `CHANGELOG.md`
- This deduplication section.

## Verification

- `bundle exec jekyll build` runs cleanly after the edits.
- The "Concept → canonical home" table above is the explicit ledger of what moved; reviewing each row confirms every fact survived in exactly one place.
- The previous changelog's "Still overlaps" section (hypervisor mechanics, silicon RoT, TPM/measured-boot recap, CVM SKU naming) has been resolved either by trimming (CVM SKU naming, Hyper-V architecture in security.md) or by audit and confirmation that the existing split was correct on the merits (silicon RoT angles, measured-boot recap).

---

# Lint pass

A targeted audit of the repo for issues introduced or missed during the restructure and dedupe passes. Each issue found was fixed; the build remains clean.

## Broken cross-links

`docs/core/gpu-confidential-computing.md` had several links to non-existent slugs predating the restructure:

| Line | Before | After |
|---|---|---|
| 13 | `[Intel TDX](tdx-architecture)` | `[Intel TDX]({{ site.baseurl }}/docs/core/tdx/)` |
| 156 | `[TDISP](tdx-trusted-io)` | `[TDISP]({{ site.baseurl }}/docs/core/tdx/trusted-io/)` |
| 163 | `[Confidential AI](confidential-ai)` | `[Confidential AI]({{ site.baseurl }}/docs/core/confidential-ai/)` |
| 164 | `[Intel TDX — Architecture & Threat Model](tdx-architecture)` | `[Intel TDX]({{ site.baseurl }}/docs/core/tdx/)` |
| 165 | `[Intel TDX — Trusted I/O (TDISP)](tdx-trusted-io)` | same text, fixed link |

`docs/core/tdx/trusted-io.md` line 56 had a same-folder reference (`gpu-confidential-computing`) that did not resolve from the `tdx/` subfolder; replaced with the `{{ site.baseurl }}` form.

## Cross-link convention drift

Several files used relative `.md` paths or the `relative_url` Liquid filter instead of the SCHEMA's `{{ site.baseurl }}/docs/<section>/<file>/` convention. All normalized:

- `docs/core/confidential-ai.md` — same-folder `gpu-confidential-computing` link.
- `docs/core/live-migration.md` — `(tdx/live-migration.md)` → `{{ site.baseurl }}/docs/core/tdx/live-migration/`.
- `docs/core/tdx/index.md` — six relative links (`abi.md`, `trusted-io.md`, `live-migration.md`, `../secure-key-release.md`, plus an anchored `abi.md#…`) → `{{ site.baseurl }}` form.
- `docs/core/tdx/live-migration.md` — `(index.md)` and `(../live-migration.md)` → `{{ site.baseurl }}` form.
- `docs/containers/kubernetes.md` — `{{ "docs/containers/kata-containers/" | relative_url }}` → `{{ site.baseurl }}/docs/containers/kata-containers/`.

A verification script (`grep -rohE '\{\{ site\.baseurl \}\}/docs/[^)"]+' | check each target exists`) now reports zero broken links. Two remaining "broken" hits are documentation examples inside SCHEMA.md (`<section>/<file-without-.md>` placeholder and `docs/...` ellipsis); both are intentional teaching examples, not real links.

## Orphan pages (now linked)

Files only reachable from `index.md` and `SCHEMA.md` — fixed by adding inbound content links:

- **`docs/core/tee-security-principles.md`** — added a link from `tees-in-practice.md` intro: "This note follows the [TEE security principles] by looking at where TEEs actually show up…".
- **`docs/tools/open-policy-agent.md`** — OPA was named in two container files but not linked. Added `{{ site.baseurl }}` link in `confidential-containers.md` (Key Building Blocks) and in `kata-containers.md` (Security Policy section).
- **`docs/misc/microsoft-azure-overview.md`** and **`docs/misc/cloud-fundamentals.md`** — left as TOC-only inbound. The Azure overview is a slot-1 gateway page that is naturally referenced from the section landing rather than from the deeper slot 2/3/4/5 notes; cloud-fundamentals is cloud-agnostic supplementary material. Both are appropriate orphans by design.

## Concepts mentioned but never explained

Found and fixed:

- **OHTTP** (Oblivious HTTP) — used six times in `docs/core/confidential-ai.md` without ever being defined. Added a one-paragraph definition at the head of "Step-by-Step Example" referencing IETF RFC 9458 and explaining the relay-and-target split.
- **MAA without canonical link** — both `docs/core/confidential-ai.md` and `docs/core/secure-key-release.md` mentioned MAA at length without linking to where it is defined. Added a cross-link to `microsoft-azure-hardware-trust-and-keys.md` at the first MAA mention in each file.
- **"Azure Managed Attestation" misnaming** — `confidential-ai.md` line 88 said "Azure Managed Attestation (MAA)"; the actual product is "Microsoft Azure Attestation". Fixed.

## Frontmatter inconsistencies

| File | Issue | Fix |
|---|---|---|
| `docs/core/tdx/abi.md` | `nav_order: 2` (with no nav_order 1 sibling) | → `1` |
| `docs/core/tdx/trusted-io.md` | `nav_order: 3` | → `2` |
| `docs/core/tdx/live-migration.md` | `nav_order: 4` | → `3` |
| `docs/tools/grpc.md` | `title: GRPC` (inconsistent with prose & index reference to "gRPC") | → `gRPC` |

Section index nav_orders (`docs/core/index.md` = 1, `containers` = 2, `tools` = 3, `misc` = 4) are contiguous and match section titles. Top-level `Home` is `nav_order: 0`. All `parent:` fields match the section index `title:` values. Sub-folder pages (`docs/core/tdx/*.md`) correctly use `parent: Intel TDX` and `grand_parent: Core Concepts`.

After the tdx/ renumbering, the subfolder ordering is `1: ABI Reference`, `2: Trusted I/O (TDISP)`, `3: Live Migration` — contiguous from 1.

## Forward dependencies

Most apparent forward references are actually pointers across reading-order tiers (e.g., a core/conceptual file references the Azure-specific canonical home in `docs/misc/`). These are not "forward" in the harmful sense — they are the natural cross-cloud → canonical-home pattern the SCHEMA prescribes — but they need to be link-equipped so the reader can jump. The MAA + THIM + Managed HSM mentions in `confidential-ai.md` and `secure-key-release.md` were the main offenders here; they now link to the canonical Azure note (see "Concepts mentioned but never explained" above).

No genuine in-section forward dependencies were found: each `docs/core/` page is self-contained relative to its predecessors in nav_order.

## Remaining duplication

A second sweep after the last dedupe pass found no new "same concept explained in depth in 2+ places" cases beyond what the previous changelog already records. The Entra ID parenthetical "(formerly Azure Active Directory)" appears in four notes; each occurrence is a brief 3-word reminder rather than a definition, and serves the local context — left as-is.

## Verification

- `bundle exec jekyll build` runs cleanly after the lint fixes.
- Cross-link verification script passes (zero broken `{{ site.baseurl }}` links; the two remaining hits are intentional placeholder examples in SCHEMA.md).
- All content files have inbound links (either from `index.md`/`SCHEMA.md` or from another content file).
- Frontmatter is contiguous and parents match section titles across all four sections.

---

# Merge pass: four Azure security files into one

The previous restructure and dedupe passes produced four logically connected Azure files that told a single story (silicon → boot → hypervisor → isolation → encryption → keys → shared responsibility). This pass merges them into a single `microsoft-azure-security.md` so the through-line is preserved as one continuous reading, while keeping every fact intact.

## Files affected

**Deleted (content fully absorbed into `microsoft-azure-security.md`):**
- `docs/misc/microsoft-azure-platform-trust-chain.md`
- `docs/misc/microsoft-azure-hardware-trust-and-keys.md`
- `docs/misc/microsoft-azure-security-model.md`

**Created/rewritten:**
- `docs/misc/microsoft-azure-security.md` — now titled "Microsoft Azure Security Architecture", organized into five Parts (continuous §§1–22) and one consolidated Sources list.

**Reordered (unchanged content):**
- `docs/misc/cloud-fundamentals.md`: `nav_order` 6 → 3
- `docs/misc/scratch.md`: `nav_order` 7 → 4

`microsoft-azure-overview.md` keeps `nav_order: 1`; `microsoft-azure-security.md` keeps `nav_order: 2`. Misc nav_orders are now contiguous 1–4.

## Internal structure of the merged file

| Part | Sections | Owns |
|---|---|---|
| **I — The Platform Trust Chain** | §§1–9 | Silicon Layer (SoC catalog, Caliptra), Hardware Roots of Trust (Cerberus, Pluton, anti-rollback), UEFI, Secure Boot, Measured Boot + TPM 2.0 (canonical), Code Integrity / dm-verity, Hashes, Hypervisor (Hyper-V architecture canonical), Confidential VMs and GPUs (CVM SKU families canonical) |
| **II — Infrastructure and Tenant Isolation** | §§10–11 | Datacenter geography, two networks, Fabric Controller, operator access (MCIO/JIT/SAW/PAW), customer's grouping hierarchy, compute isolation spectrum, "trust grows monotonically" TCB-by-compute-mode table, network isolation, per-service isolation |
| **III — Encryption and Key Management** | §§12–17 | Host attestation (with MAA), THIM, encryption at rest and in transit, double encryption, 5-way HSM decision (Marvell LiquidSecurity canonical), Secure Key Release on Azure |
| **IV — Shared Responsibility and Customer Data** | §§18–20 | Shared-responsibility matrix, defense-in-depth services (Trusted Launch canonical), customer data protection (Customer Lockbox canonical) |
| **V — Through-Lines and Practical Takeaways** | §§21–22 | Single consolidated through-line and practical-takeaways sections; replaces the four per-file "Through-Line" + "Practical Takeaways" sections |

The merge preserved every fact from the four source files. Per the user's "consolidate, don't cut" instruction, content reduction came from:

1. **Removing four duplicate file-level intros** (each source file opened with a prose summary of itself; the merged file has a single intro at the top).
2. **Removing four "Through-Line" and four "Practical Takeaways" sections**, replaced by single Part V sections that draw connections across all four parts and consolidate takeaways without losing any specific guidance.
3. **Consolidating four Sources lists into one**, deduplicated by URL.

No facts were dropped — verified by reviewing the heading map of the original four files (Step 1 of the merge) against the merged file's section list.

## Cross-link updates

Every reference to the three deleted files was rewritten to point at the merged file with a section anchor.

| File | Old reference | New reference |
|---|---|---|
| `docs/core/confidential-ai.md` | `microsoft-azure-hardware-trust-and-keys/` (MAA / THIM / Managed HSM context) | `microsoft-azure-security/#12-host-attestation-fleet-internal-verification` |
| `docs/core/secure-key-release.md` (sidecar section) | `microsoft-azure-hardware-trust-and-keys/` (MAA explanation) | `microsoft-azure-security/#microsoft-azure-attestation-maa-the-customer-facing-service` |
| `docs/core/secure-key-release.md` (KMS lineup section) | `microsoft-azure-hardware-trust-and-keys/` (5-way KMS decision) | `microsoft-azure-security/#16-choosing-a-key-manager-the-five-way-decision` |
| `index.md` | Five separate per-slot bullets (Overview + 4 security slots) | Two bullets: Overview, plus a single Security Architecture entry with five Part-summary sub-bullets |
| `docs/misc/index.md` | "starting with Microsoft Azure (overview, platform trust chain, infrastructure & tenant isolation, hardware trust & encryption & key management, security model)" | "two slots per cloud — an Overview and a single consolidated Security Architecture note" |
| `README.md` | Five-slot description | Two-slot description with the five Parts spelled out |
| `SCHEMA.md` | "Five per-cloud slots" + corresponding ownership table rows + "How to add a new cloud provider" five-file template | "Per-cloud structure: two slots, one consolidated security note" + corresponding ownership row + "How to add a new cloud provider" two-file template |

Anchor links were verified by inspecting the rendered `_site/docs/misc/microsoft-azure-security/index.html`. All anchors used in cross-links exist:
- `12-host-attestation-fleet-internal-verification` ✓
- `microsoft-azure-attestation-maa-the-customer-facing-service` ✓
- `16-choosing-a-key-manager-the-five-way-decision` ✓
- `customer-lockbox` ✓

## Schema-level change

The per-cloud pattern moved from **five slots** to **two slots**:

- *Before:* Overview + Platform Trust Chain + Infrastructure & Tenant Isolation + Hardware Trust, Encryption & Keys + Security Model.
- *After:* Overview + Security Architecture (five-Part internal structure).

`SCHEMA.md` was rewritten to reflect:
- New domain-ownership table row for `microsoft-azure-security.md` describing the five-Part contents.
- Updated decision tree and "Where new content should go" sections.
- New "Per-cloud structure: two slots" replacement for the old "Five per-cloud slots" table.
- Updated "How to add a new cloud provider" template (two files instead of five).
- Anti-rule added: "Do not split the Security Architecture into multiple files. The whole point of the consolidation is that trust → isolation → encryption → responsibility is one narrative."
- Updated verification grep targets (e.g., CVM SKU substrings should now land in `microsoft-azure-security.md`, not `-hardware-trust-and-keys.md`).

## Verification

- `bundle exec jekyll build` runs cleanly.
- `grep -rn "microsoft-azure-platform-trust-chain\|microsoft-azure-hardware-trust-and-keys\|microsoft-azure-security-model" docs/ index.md SCHEMA.md README.md` returns no live references (only historical mentions in this CHANGELOG).
- All four anchor links into the merged file resolve to existing IDs in the rendered HTML.
- `docs/misc/` now contains exactly five files (one less than before): `index.md`, `microsoft-azure-overview.md`, `microsoft-azure-security.md`, `cloud-fundamentals.md`, `scratch.md`. nav_orders are contiguous 1–4 across the four content files.

## Merge resolution against `main` (PR #46 absorbed)

While this PR was open, `main` advanced with PR #46 ("Add Azure attestation/SKR artifact map and cross-links"). Three modify/delete or content conflicts were resolved as follows:

- **`microsoft-azure-hardware-trust-and-keys.md`** (modify/delete): re-deleted. The new "Artifact Map" section that PR #46 added to the original file was absorbed into the merged note as a `### Artifact map` subsection at the end of §17 (Secure Key Release).
- **`microsoft-azure-platform-trust-chain.md`** (modify/delete): re-deleted. The "On verifying the chain" practical takeaway PR #46 introduced is now part of the merged note's §22 Practical Takeaways.
- **`microsoft-azure-security.md`** (content conflict): kept the merged-note content for the structurally-conflicting hunk (the origin/main version was a stale paragraph from the pre-merge "Infrastructure & Tenant Isolation" file). The artifact-map cross-link references PR #46 added in §13 (THIM consumers) and at the top of §17 (SKR intro) were preserved by adding equivalent in-file anchor links (`#artifact-map`).

Net effect: PR #46's artifact-map content lives intact inside the merged note. No facts lost.
