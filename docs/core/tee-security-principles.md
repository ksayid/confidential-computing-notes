---
title: TEE Security Principles
parent: Core Concepts
nav_order: 8
---

# TEE Security Principles

A TEE is only as trustworthy as its attestation boundary is complete. If critical inputs fall outside the measurement, the entire trust model breaks — regardless of how strong the hardware isolation is.

## Core Principles

### Measure every input, not just code
Environment variables, configs, feature flags, and runtime parameters loaded after attestation create a gap between what the client verified and what actually executes. Anything consumed at runtime must be measured or treated as hostile.

### The attack surface includes everything your code trusts
Hardware configuration (like ACPI tables) sits below application code but still shapes behavior. A malicious hypervisor can exploit unmeasured hardware definitions to access protected memory while attestation appears valid. Most teams measure their binaries but forget the layers beneath them.

### Never trust self-reported trustworthiness
If firmware or any component reports its own patch level, a compromised version can simply lie. Verification must use cryptographically signed external sources. Self-reporting reintroduces the exact trust assumption attestation was designed to eliminate.

### Attestation must prove freshness, not just identity
Without a session-specific nonce or equivalent, valid attestation reports can be replayed indefinitely, turning a one-time compromise into persistent impersonation.

## Complementary Techniques Worth Considering

Attestation and measurement are the foundation, but a full security solution usually layers in additional techniques:

- **RA-TLS (Remote Attestation TLS)**: Combines attestation with encrypted communication so a client can verify what code is running on the server and establish a secure channel in one step.
- **Anonymous routing**: Decouples requests from user identity so even if someone observes traffic patterns, they can't tie a specific request back to a specific user.
- **Transparency logs**: Append-only records (often hosted by a third party) of every software artifact running in the system. The client won't connect unless the server proves it's running code that matches what's been logged. Changes can't be made silently.
- **Ephemeral, stateless processing**: No data is retained after a request is handled. There's no history to leak, subpoena, or compromise later.
- **In-app transparency reports**: Letting users see what happened to their data is a design pattern worth noting — it gives users visibility without requiring technical sophistication.

## For Builders

Treat these as a threat-model checklist:

- What sits outside your measured boundary?
- Are any inputs loaded post-measurement?
- Is firmware and platform state verified cryptographically?
- Is attestation bound to a specific session?

If you can't answer clearly, your TEE story is weaker than you think.

## For Enterprise Buyers

Go beyond "do you use TEEs?" and ask:

- What exactly is measured?
- What sits outside the attestation boundary?
- How are firmware and patch state verified?
- How is replayed attestation prevented?
