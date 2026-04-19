---
title: TEEs in Practice
parent: Core Concepts
nav_order: 9
---

# TEEs in Practice

## The Big Picture

TEEs are emerging as a practical trust layer across crypto infrastructure and AI agent systems. They're not trustless, but they meaningfully shrink exposure by providing hardware-backed isolation, remote attestation, and constrained key handling. Think of them as a bridge — useful now while stronger cryptographic guarantees (ZK, PIR, MPC) mature.

## What TEEs Actually Provide

- **Isolated execution** where even a privileged host operator can't read plaintext memory
- **Remote attestation** so clients can verify what code is running before releasing secrets
- **Code integrity via measurement** to detect tampering
- **Verifiable runtime identity** — especially valuable for services like MCP servers that currently have no strong way to prove what's running behind an endpoint
- **Coordination without full consensus overhead** — they don't replace consensus, they change *when* it's needed

## Real-World Examples

- **Coinbase (Nitro Enclaves)**: Enforces signing policies *inside* the enclave rather than at the API perimeter. This matters especially as autonomous agents begin initiating transactions — the policy and the key live in the same trust boundary.
- **Flashbots (SGX)**: Uses SGX for confidential block building in MEV, with commendable transparency about the tradeoffs (side-channel risks, reproducible build complexity).
- **Vitalik's framing**: TEEs as a near-term privacy fix for RPC and read-path privacy, while stronger cryptographic methods like PIR mature.

## The Agent Commerce Angle

When paired with protocols like **x402** (machine-to-machine HTTP-native payments), TEEs form an emerging stack:

- **x402** handles *how* agents pay
- **TEEs** constrain *how keys authorize* those payments
- **Attestation** verifies *where* that logic runs

## AI Agents and MCP Servers

Remote MCP servers are an urgent and underappreciated case. A single MCP process typically aggregates multiple high-value credentials — GitHub tokens, database connection strings, AWS keys, Slack tokens — all in one address space. Existing controls don't cover this:

- **IAM** authorizes API calls, not memory reads
- **Secrets vaults** protect data at rest but not in use
- **Container isolation** doesn't defend against privileged host operators
- **Network policies** are irrelevant when credentials can be read directly from process memory

TEEs close this infrastructure gap, but in practice they call for a **two-layer credential model**:

1. **Baseline service credentials** are provisioned at deployment through attestation-to-key-vault flows.
2. **Agent-sent scoped tokens** (user OAuth, STS credentials) are only released after the agent verifies the server's attestation.

The TEE protects both layers from host-level extraction.

## Limits

TEEs are not a silver bullet. A reasonable rule across both crypto and AI: use TEEs to reduce exposure windows and protect confidentiality, never as the sole trust guarantee.

- **In crypto**: side-channel attacks, implementation bugs, supply-chain risks, and vendor trust remain real.
- **In AI**: prompt injection, sampling attacks, and tool poisoning are application-layer threats that hardware isolation can't touch.

## Where It's Heading

The architecture is evolving toward hybrid models — TEE execution verified or audited with ZK techniques, gradually migrating trust from hardware to cryptography over time without waiting for perfect solutions.

## Open Problems

Common across crypto and AI:

- **Multi-party attestation** across chained services (MCP servers, MEV pipelines)
- **Dynamic capability delegation** where LLMs or agents grant scoped access
- **Attestation-aware transport protocols**
- **Confidential observability** for SOC teams
- **Long-term migration** from hardware trust roots to cryptographic verifiability

## The Builder's Mindset That Wins

The right sequencing differs by domain, but the throughline is the same: ship useful systems now, document trust assumptions honestly, and design for eventual compromise.

- **In crypto**: ship with TEEs now, expose assumptions clearly, and build migration paths toward stronger cryptographic guarantees (hybrid TEE + ZK).
- **In AI**: ship the capability model first (default-deny permissions, scoped tokens, confirmation gates for writes), then layer TEEs on top to close the host-trust gap.
- **In both**: plan for key rotation, revocation, and forensic observability from day one.

Ideological purity — whether "fully trustless" in crypto or "we have IAM" in AI — loses to operational accountability.
