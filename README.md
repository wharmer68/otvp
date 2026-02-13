# OTVP — Open Trust Verification Protocol

**Real-time, programmable, queryable trust. Not another PDF.**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Status](https://img.shields.io/badge/Status-v1.0--DRAFT-orange.svg)]()

---

## The Problem

The way companies prove their security posture is broken.

SOC 2, ISO 27001, and similar frameworks rely on **human auditors** producing **static reports** that are **months stale** by the time anyone reads them. The company being audited pays the auditor — creating a structural conflict of interest. The resulting 80-page PDF is a snapshot of a moment that no longer exists, written for a generic audience, with no way to verify the underlying evidence.

Meanwhile, the relying party — the customer trying to decide if a vendor is trustworthy — checks the box and moves on. Nobody reads the report. The certificate becomes compliance theater.

**This isn't trust. It's tradition.**

## What OTVP Does

OTVP replaces static audit reports with a system of **continuously operating, certified verification agents** that produce **on-demand, cryptographically verifiable evidence** about a company's security controls, posture, and behavior.

Instead of asking *"Did an auditor check a box six months ago?"*, a customer asks:

> *"Show me — right now — cryptographic proof that this company encrypts my data at rest with AES-256, rotates keys every 90 days, and has functioning access controls on the systems that touch my PHI."*

And gets an answer in seconds. Signed. Verifiable. Scoped to exactly the data and risk context that matters to them.

## Core Principles

| # | Principle | What It Means |
|---|-----------|---------------|
| 1 | **Evidence over attestation** | Raw, verifiable evidence is the primitive. Opinions are layered on top with clear provenance. |
| 2 | **Continuous over periodic** | Controls are verified on an ongoing basis, not during audit windows. |
| 3 | **Queryable over monolithic** | Relying parties ask specific questions and get specific answers — not a 100-page report. |
| 4 | **Cryptographic over contractual** | Trust is established through verifiable proofs, not legal agreements about honesty. |
| 5 | **Open over proprietary** | The standard is open. Agents, evidence formats, and query interfaces are interoperable. |
| 6 | **Contextual over universal** | Trust assessments are scoped to the specific relationship and data flow between parties. |
| 7 | **Adversarial by design** | The protocol assumes the subject company may attempt to game it. |
| 8 | **Composable over monolithic** | Small, verifiable claims compose into larger trust assessments. |

## How It Works

```
┌─────────────────────────────────────────────────┐
│              RELYING PARTY (Customer)            │
│                                                  │
│  "Does $VENDOR encrypt my PHI at rest            │
│   with AES-256 and rotate keys ≤ 90 days?"      │
│                                                  │
│           ┌──────────────────────┐               │
│           │    TRUST QUERY (TQ)  │               │
│           └──────────┬───────────┘               │
└──────────────────────┼──────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│              OTVP TRUST BROKER                    │
│                                                  │
│  Routes queries → Collects evidence →            │
│  Returns signed Trust Envelopes                  │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│           SUBJECT COMPANY (Vendor)               │
│                                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐      │
│  │ Agent:IAM │ │ Agent:Enc │ │ Agent:Net │ ...  │
│  │(Certified)│ │(Certified)│ │(Certified)│      │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘      │
│        │              │              │            │
│        ▼              ▼              ▼            │
│  ┌────────────────────────────────────────┐      │
│  │     EVIDENCE COLLECTION LAYER          │      │
│  │  Cloud APIs · Configs · Logs · Policies│      │
│  └────────────────────────────────────────┘      │
│                                                  │
│  ┌────────────────────────────────────────┐      │
│  │     EVIDENCE STORE (Immutable)         │      │
│  │  Merkle tree · Append-only · Signed    │      │
│  └────────────────────────────────────────┘      │
└──────────────────────────────────────────────────┘
```

The response comes back as a **Trust Envelope** — a signed, timestamped, machine-readable JSON object containing the evidence, the agent's assessment, confidence scores, and a cryptographic proof chain. Not a PDF. Not an opinion letter. Verifiable math.

## Key Concepts

- **Trust Envelope** — Replaces audit reports. JSON, signed, with TTLs measured in minutes not months.
- **Verification Agents** — Domain-specific, certified software agents that run inside the subject's environment and produce evidence. The company doesn't grade its own homework.
- **OTVP-QL** — A query language that lets relying parties ask exactly the questions they care about, scoped to their specific data and risk context.
- **Control Domains** — Modular security domains (encryption, IAM, network, incident response, etc.) each with defined evidence requirements.
- **Framework Bridge** — Maps OTVP domains to SOC 2, ISO 27001, HIPAA, PCI-DSS, and other existing frameworks for migration compatibility.
- **Zero-Knowledge Proofs** — Companies can prove compliance without revealing proprietary architecture details.

## What's in This Repo

```
├── README.md                          ← You are here
├── LICENSE                            ← Apache 2.0
├── CONTRIBUTING.md                    ← How to contribute
├── GOVERNANCE.md                      ← How decisions are made
│
├── spec/
│   └── OTVP-Specification-v1.0.md    ← The full specification
│
├── schemas/
│   ├── trust-envelope.schema.json     ← Trust Envelope JSON schema (planned)
│   ├── trust-query.schema.json        ← Trust Query JSON schema (planned)
│   └── evidence.schema.json           ← Evidence object schema (planned)
│
├── examples/
│   ├── queries/                       ← Example OTVP-QL queries (planned)
│   └── envelopes/                     ← Example Trust Envelope responses (planned)
│
├── mappings/
│   ├── soc2-mapping.md                ← SOC 2 → OTVP domain mapping (planned)
│   ├── iso27001-mapping.md            ← ISO 27001 → OTVP mapping (planned)
│   └── hipaa-mapping.md               ← HIPAA → OTVP mapping (planned)
│
└── agents/
    └── reference/                     ← Reference agent implementations (planned)
```

## How OTVP Compares

| Dimension | SOC 2 / ISO 27001 | OTVP |
|-----------|-------------------|------|
| **Evidence** | Auditor samples | Continuous, automated, multi-source |
| **Freshness** | 3–12 months stale | Real-time (configurable TTL) |
| **Verifiability** | Trust the auditor | Cryptographically verifiable |
| **Queryability** | Read the report | Programmable queries, contextual results |
| **Machine-readable** | PDF | JSON, API, webhooks |
| **Contextual** | One-size-fits-all | Scoped to relying party's data and risk |
| **Incentive alignment** | Subject pays auditor | Relying party selects agents; subject deploys |
| **Time to answer** | Days to weeks | Seconds to minutes |

## Current Status

**v1.0-DRAFT** — The specification is complete in draft form and open for community review. This includes:

- Full architecture and protocol definition
- Trust Envelope and Evidence specifications
- OTVP-QL query language grammar
- Control domain definitions (12 domains)
- Agent certification model
- Framework mapping tables (SOC 2, ISO 27001, HIPAA, PCI-DSS, NIST CSF)
- Governance model

**What's next:**
- Community review and feedback on the draft spec
- JSON schemas for Trust Envelopes, Queries, and Evidence objects
- Reference agent implementation (starting with AWS encryption domain)
- `otvp-agent-sdk` for building verification agents

## Who This Is For

- **Security leaders** tired of paying $50K–$500K for stale audit reports
- **Procurement teams** who want real answers, not checkbox compliance
- **Founders and CTOs** who want trust to be a feature, not a tax
- **Auditors and GRC professionals** looking at where the industry is heading
- **Platform and infrastructure companies** who want to prove trust programmatically
- **Anyone** who thinks the current model is broken and wants to help fix it

## Contributing

We welcome contributions. See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

The fastest ways to contribute right now:
1. **Read the spec** and open issues with feedback
2. **Propose framework mappings** — help map OTVP domains to additional compliance frameworks
3. **Build a reference agent** — pick a control domain and build a proof-of-concept
4. **Socialize the idea** — share this with security teams, auditors, and procurement professionals

## License

This specification and all reference implementations are released under the [Apache License 2.0](LICENSE). No patents. No royalties. No vendor lock-in.

Agent vendors can build proprietary agents on top of the open standard. The standard itself remains free and open.

---

*"The best audit report is one that never goes stale."*

**Created by [Bil Harmer](https://github.com/bilharmer)** · Contributions welcome
