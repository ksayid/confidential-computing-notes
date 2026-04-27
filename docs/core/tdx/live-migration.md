---
title: Live Migration
parent: Intel TDX
grand_parent: Core Concepts
nav_order: 3
---

# TDX Live Migration

Intel TDX v1.5 adds live migration on top of the TDX v1.0 confidential-VM machinery described in the [TDX overview]({{ site.baseurl }}/docs/core/tdx/). The same brownout/blackout shape from the [generic case]({{ site.baseurl }}/docs/core/live-migration/) still applies — what changes is who can see the state in transit.

## Why it's harder under TDX

In legacy VMM migration, the host VMM is trusted and has full access to VM memory and state, so source and destination VMMs handle verification, authentication, and transport directly. Under TDX, the host VMM is **untrusted** and cannot read TD memory or state at all. Every byte of migration data has to flow through the TDX Module, encrypted and integrity-protected. This forces two additional components with no legacy equivalent: a **Migration TD (migTD)** that brokers trust between platforms, and a set of **TDX Module migration APIs** that package and unpackage state.

## Roles

| Role | Description |
|---|---|
| **Target TD (tgtTD)** | The TD being migrated. It is *unaware* migration is happening and performs no special actions. |
| **Source TD (srcTD)** | The tgtTD while running on the source platform. |
| **Destination TD (dstTD)** | A template TD constructed on the destination platform to receive migration data. |
| **Migration TD (migTD)** | A special TD created on *both* platforms to verify attestation and exchange cryptographic keys. Bound to the tgtTD before its measurements are finalized. |
| **Source/Destination VMM** | The host VMMs on each platform. They coordinate the transfer but are untrusted and cannot see TD data. |

Although migration is designed for cross-platform relocation, the operations can technically take place on a single platform.

## Bundles, session keys, and attestation

All TD state transferred during migration is packaged into **migration bundles** — units of private memory or non-memory state that are both encrypted and integrity-protected under AES-GCM-256 using a **Migration Session Key (MSK)**. The TDX Module generates the MSK; the migTDs on each platform exchange MSKs with each other, and the destination migTD installs the key into the dstTD.

Before migration proceeds, the migTDs examine each other's TDX Module capabilities and attestation evidence and check them against a **migration policy** that specifies acceptable TD attributes, allowed SVNs, and supported migration protocol versions. This is what extends TDX's confidentiality and integrity guarantees across platforms: the dstTD only accepts state that comes from a TDX Module the migTD has vouched for.

## Migration Stream Contexts

Some migration steps — decrypting bundles, importing metadata — are time-consuming. The TDX Module checks for pending hardware interrupts during these operations and, if one arrives, saves progress to a **Migration Stream Context (MigSC)**, returns control to the VMM, and later resumes from where it left off — verifying that no conflicting operations occurred in the interim. This keeps the untrusted VMM responsive without giving it a window to tamper with in-flight state.

## Phases

The migration walks the dstTD through a sequence of operation states, paralleling the normal build sequence but sourced from bundles rather than from the VMM directly.

1. **Setup.** The destination VMM creates a template dstTD (allocate → configure encryption → add control structures, the same common setup as a normal build). A migTD is created on both platforms and bound to the target TD. Migration Stream Contexts are created, and the migTDs exchange Migration Session Keys.
2. **Immutable state import.** The dstTD's immutable state — the equivalent of what the normal build's initialization step sets up — is imported from encrypted bundles. Interruption and resumption via MigSCs is supported. On completion, the TD enters the **Memory Import** operation state.
3. **In-order memory import (brownout).** The srcTD *continues running*. Memory pages are exported from the source, encrypted into bundles, and imported into the dstTD where they are decrypted and verified (attributes and location must match between source and destination). Order is critical: if the same page is exported multiple times because the srcTD modified it, imports must happen oldest-first, newest-last so the final state is correct.
4. **Blackout begins.** The Intel TDX-imposed blackout starts: the srcTD is *stopped*. Mutable TD-scope state (registers, control structures, other runtime state that changes while the TD runs) can only be exported during this blackout. That state is imported into the dstTD, which enters the **State Import** operation state.
5. **VP state and dirty-page migration.** With the srcTD stopped, per-VP state is imported and any pages dirtied during brownout are re-migrated. Memory import continues in parallel. **Epoch tracking** manages the transition out of this phase: the source creates migration epoch tokens and the destination consumes them; when the final (start) token arrives, the TD enters **Post-Import**.
6. **Out-of-order memory import (post-copy).** The srcTD is no longer runnable and page ordering no longer matters, so memory can be imported in any order. Once the import is committed, **the blackout ends and the dstTD can start running — even before all memory pages have arrived.** If the dstTD accesses a page that hasn't transferred yet, an EPT violation fires and the destination VMM can prioritize that page. The rest stream in in the background. When all memory is transferred and the migration is finalized, the TD returns to the **Runnable** state.

Shared (non-private) memory is handled separately through traditional VMM workflows, outside the TDX Module.

## Operation-state flow

```
              ┌─────────────────────────────────────────┐
              │          COMMON SETUP                   │
              │  Allocate TD → Configure Encryption     │
              │  → Add Control Structures               │
              └──────────────┬──────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
         BUILD PATH                    IMPORT PATH
              │                             │
     Initialize TD state          Bind migTD, create
              │                   streams, exchange keys
              ▼                             │
        INITIALIZED               Import immutable state
              │                             │
     Create VPs, populate                   ▼
     memory, measure                 MEMORY IMPORT
              │                   (srcTD still running,
     Finalize measurements         in-order page import)
              │                             │
              │                   Stop srcTD (blackout),
              │                   import mutable TD state
              │                             │
              │                             ▼
              │                      STATE IMPORT
              │                   (VP state, dirty pages,
              │                    epoch tracking)
              │                             │
              │                             ▼
              │                      POST-IMPORT
              │                   (out-of-order pages,
              │                    no ordering needed)
              │                             │
              │                   Commit → blackout ends
              │                   → dstTD can run
              │                             │
              │                             ▼
              │                      LIVE IMPORT
              │                   (post-copy, dstTD running,
              │                    remaining pages transfer)
              │                             │
              │                   Finalize migration
              │                             │
              ▼                             ▼
                        RUNNABLE
```
