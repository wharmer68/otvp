# OTVP — Open Trust Verification Protocol

## Specification v1.0-DRAFT

**Status:** Public Draft  
**License:** Apache 2.0  
**Authors:** Bil Harmer, [Contributors Welcome]  
**Last Updated:** 2026-02-11  
**Repository:** github.com/otvp-standard/otvp-spec (proposed)

---

## Abstract

The Open Trust Verification Protocol (OTVP) replaces static, periodic audit reports (SOC 2, ISO 27001, etc.) with a system of **continuously operating, certified verification agents** that produce **on-demand, cryptographically verifiable evidence** about a company's security controls, posture, and behavior.

Trust becomes **real-time, programmable, and queryable.**

Instead of asking "Did an auditor check a box six months ago?", a customer, partner, or regulator asks: *"Show me — right now — cryptographic proof that this control is operating, and an expert agent's opinion on its effectiveness."*

---

## Table of Contents

1. [The Problem with Current Trust Models](#1-the-problem-with-current-trust-models)
2. [Design Principles](#2-design-principles)
3. [Architecture Overview](#3-architecture-overview)
4. [Core Concepts](#4-core-concepts)
5. [The Trust Envelope](#5-the-trust-envelope)
6. [Control Domains](#6-control-domains)
7. [Verification Agents](#7-verification-agents)
8. [Evidence Specification](#8-evidence-specification)
9. [Trust Queries — The Query Interface](#9-trust-queries--the-query-interface)
10. [Cryptographic Verification Layer](#10-cryptographic-verification-layer)
11. [Agent Certification & Governance](#11-agent-certification--governance)
12. [Trust Levels & Scoring](#12-trust-levels--scoring)
13. [Interoperability & Migration](#13-interoperability--migration)
14. [Threat Model of OTVP Itself](#14-threat-model-of-otvp-itself)
15. [Implementation Guide](#15-implementation-guide)
16. [Governance & Standards Body](#16-governance--standards-body)
17. [Appendices](#17-appendices)

---

## 1. The Problem with Current Trust Models

### 1.1 What's Broken

The dominant trust frameworks — SOC 2, ISO 27001, PCI DSS, HITRUST — share a fundamental architectural flaw: **they produce static attestations about dynamic systems.**

**Point-in-time evidence.** An auditor visits (or reviews screenshots) over a 2-8 week window. The resulting report describes the state of controls during that window. By the time a customer reads it, the evidence is 3-12 months stale.

**Human-speed verification.** Auditors manually sample controls. Sample sizes are small. The audit itself is a performance — companies prepare for audits the way students cram for exams. The steady-state posture between audits is unknown.

**Binary trust decisions from analog evidence.** A SOC 2 Type II report is either "clean" or has "exceptions." There is no gradient. A company with one minor exception in password policy looks the same as a company barely holding things together. Customers cannot query the specific controls they care about.

**Misaligned incentives.** The audited company pays the auditor. The auditor's incentive is to keep the client happy while maintaining enough rigor to avoid regulatory trouble. This is not corruption — it is structural. The party relying on the report (the customer) has no seat at the table.

**Compliance theater.** The existence of a certificate becomes a substitute for actual security. Procurement teams check the box. Nobody reads the 80-page SOC 2 report. The certificate creates a false sense of due diligence that dissolves the moment an actual breach occurs.

**No context for the relying party.** A SaaS company storing anonymized analytics data and a SaaS company storing patient health records both present the same SOC 2 Type II. The relying party cannot ask: "Show me the controls that matter for *my* data, in *my* risk context."

### 1.2 What Should Replace It

A trust system fit for modern software supply chains must be:

- **Continuous** — not point-in-time
- **Evidence-based** — not opinion-based
- **Cryptographically verifiable** — not trust-me-bro
- **Queryable** — not one-size-fits-all
- **Machine-readable** — not PDF
- **Contextual** — scoped to the relying party's actual risk
- **Incentive-aligned** — the verifier serves the relying party, not the subject

---

## 2. Design Principles

### 2.1 Core Principles

| # | Principle | Implication |
|---|-----------|-------------|
| 1 | **Evidence over attestation** | Raw, verifiable evidence is the primitive. Opinions are layered on top with clear provenance. |
| 2 | **Continuous over periodic** | Controls are verified on an ongoing basis, not during audit windows. |
| 3 | **Queryable over monolithic** | Relying parties ask specific questions and get specific answers, not a 100-page report. |
| 4 | **Cryptographic over contractual** | Trust is established through verifiable proofs, not legal agreements about honesty. |
| 5 | **Open over proprietary** | The standard is open. Agents, evidence formats, and query interfaces are interoperable. |
| 6 | **Contextual over universal** | Trust assessments are scoped to the specific relationship and data flow between parties. |
| 7 | **Adversarial by design** | The protocol assumes the subject company may attempt to game it. Agents are designed to resist manipulation. |
| 8 | **Composable over monolithic** | Small, verifiable claims compose into larger trust assessments. No single authority owns the full picture. |

### 2.2 Non-Goals

OTVP does not:

- Replace the need for security engineering. It verifies; it does not remediate.
- Guarantee security. It provides evidence about controls. Controls can still fail.
- Eliminate human judgment. Expert opinion is a first-class object in the protocol, but it is always accompanied by evidence and always signed.
- Mandate specific security controls. It provides a framework for verifying whatever controls exist. Control frameworks (what *should* exist) remain separate.

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        RELYING PARTY                             │
│                                                                  │
│   "Show me that $COMPANY encrypts data at rest                   │
│    using AES-256 and rotates keys every 90 days"                 │
│                                                                  │
│   ┌──────────────────────────────────────────┐                   │
│   │         TRUST QUERY (TQ)                 │                   │
│   │   domain: encryption.at_rest             │                   │
│   │   assertions:                            │                   │
│   │     - algorithm: AES-256                 │                   │
│   │     - key_rotation: ≤ 90 days            │                   │
│   │   context:                               │                   │
│   │     - data_classification: PHI           │                   │
│   │     - relationship: processor            │                   │
│   └──────────────┬───────────────────────────┘                   │
└──────────────────┼───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                     OTVP TRUST BROKER                             │
│                                                                  │
│   Routes queries → Collects evidence → Returns Trust Envelopes   │
│   Validates agent signatures │ Caches evidence within TTL        │
│                                                                  │
└──────────────────┬───────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                   SUBJECT COMPANY                                 │
│                                                                  │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│   │  Agent: IAM  │  │ Agent: Enc  │  │ Agent: Net  │  ...        │
│   │  (Certified) │  │ (Certified) │  │ (Certified) │             │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│          │                │                │                     │
│          ▼                ▼                ▼                     │
│   ┌─────────────────────────────────────────────┐               │
│   │            EVIDENCE COLLECTION LAYER         │               │
│   │                                              │               │
│   │  Cloud APIs │ Config files │ Logs │ Policies │               │
│   │  Runtime state │ Network flows │ IAM state   │               │
│   └──────────────────────────────────────────────┘               │
│                                                                  │
│   ┌─────────────────────────────────────────────┐               │
│   │            EVIDENCE STORE (Immutable)        │               │
│   │                                              │               │
│   │  Merkle tree of all collected evidence       │               │
│   │  Append-only │ Signed │ Timestamped          │               │
│   └──────────────────────────────────────────────┘               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                     TRUST ENVELOPE (Response)                     │
│                                                                  │
│   {                                                              │
│     "query_ref": "tq-2026-02-11-enc-001",                       │
│     "subject": "acme-corp",                                     │
│     "timestamp": "2026-02-11T14:30:00Z",                        │
│     "ttl": "PT1H",                                              │
│     "claims": [                                                  │
│       {                                                          │
│         "domain": "encryption.at_rest",                          │
│         "assertion": "algorithm",                                │
│         "result": "SATISFIED",                                   │
│         "evidence_ref": "ev-abc123",                             │
│         "confidence": 0.97,                                      │
│         "agent_id": "enc-agent-v2.1",                            │
│         "agent_cert": "cert-xyz789",                             │
│         "opinion": "AES-256-GCM verified across all             │
│                     production RDS instances and S3 buckets.     │
│                     EBS volumes use AWS-managed keys with        │
│                     AES-256. No unencrypted storage detected.",  │
│         "caveats": [                                             │
│           "Dev environment uses AES-128. Not in scope for PHI." │
│         ]                                                        │
│       },                                                         │
│       {                                                          │
│         "domain": "encryption.at_rest",                          │
│         "assertion": "key_rotation",                             │
│         "result": "SATISFIED",                                   │
│         "evidence_ref": "ev-def456",                             │
│         "confidence": 0.94,                                      │
│         "agent_id": "enc-agent-v2.1",                            │
│         "opinion": "KMS key rotation enabled. Last rotation     │
│                     23 days ago. Policy enforced via SCP.",       │
│         "caveats": [                                             │
│           "Customer-managed keys in 2 accounts rotate on         │
│            120-day cycle. Remediation ticket open."               │
│         ]                                                        │
│       }                                                          │
│     ],                                                           │
│     "signature": "0x...",                                        │
│     "evidence_merkle_root": "0x..."                              │
│   }                                                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.1 Key Actors

**Subject Company** — The organization whose controls are being verified. Deploys certified OTVP agents within its environment.

**Relying Party** — The customer, partner, regulator, or insurer that wants to verify the subject's controls. Issues Trust Queries.

**Verification Agent** — A certified software agent that continuously collects evidence, evaluates controls, and produces signed claims. Agents are domain-specific (IAM, encryption, network, etc.).

**Trust Broker** — An intermediary that routes queries, validates signatures, and manages access control between relying parties and subject companies. Optional — relying parties can also query agents directly.

**Certification Authority** — The body that certifies agents meet OTVP standards for evidence collection, tamper resistance, and accuracy. Not a single entity — federated.

**Evidence Store** — An immutable, append-only store of collected evidence within the subject company's environment. Merkle-tree-backed for verifiable integrity.

---

## 4. Core Concepts

### 4.1 Evidence

Evidence is the atomic unit of OTVP. It is a cryptographically signed record of an observed fact.

```json
{
  "evidence_id": "ev-2026-02-11-001",
  "type": "OBSERVATION",
  "domain": "encryption.at_rest",
  "source": {
    "type": "cloud_api",
    "provider": "aws",
    "api": "rds:DescribeDBInstances",
    "region": "us-east-1"
  },
  "observation": {
    "resource": "arn:aws:rds:us-east-1:123456789:db/prod-primary",
    "property": "StorageEncrypted",
    "value": true,
    "metadata": {
      "kms_key_id": "arn:aws:kms:us-east-1:123456789:key/abc-def",
      "engine": "postgres",
      "engine_version": "15.4"
    }
  },
  "collected_at": "2026-02-11T14:28:00Z",
  "agent_id": "enc-agent-v2.1",
  "agent_signature": "0x...",
  "previous_evidence_hash": "0x..."
}
```

**Evidence types:**

| Type | Description | Example |
|------|-------------|---------|
| `OBSERVATION` | A directly observed configuration or state | S3 bucket encryption setting |
| `LOG_SAMPLE` | A sample from audit/system logs | CloudTrail entry showing key rotation |
| `POLICY_SNAPSHOT` | A captured policy document or configuration | IAM policy JSON, SCP definition |
| `RUNTIME_CHECK` | An active test performed by the agent | TLS handshake verification |
| `BEHAVIORAL` | A pattern observed over time | Access frequency, anomaly detection |
| `ATTESTATION` | A signed statement from a human or external system | Pen test report hash, training completion |
| `DERIVED` | A conclusion drawn from multiple evidence items | "All production databases are encrypted" |

### 4.2 Claims

A claim is an agent's assertion about a control, backed by evidence and accompanied by an opinion.

```json
{
  "claim_id": "cl-2026-02-11-001",
  "domain": "access_control.mfa",
  "assertion": "All human users with production access have MFA enabled",
  "result": "SATISFIED | NOT_SATISFIED | PARTIAL | INDETERMINATE | NOT_APPLICABLE",
  "confidence": 0.0-1.0,
  "evidence_refs": ["ev-001", "ev-002", "ev-003"],
  "opinion": "Free-form expert analysis from the agent...",
  "caveats": ["Known exceptions or limitations..."],
  "scope": {
    "environment": "production",
    "accounts": ["123456789", "987654321"],
    "exclusions": ["break-glass account (documented)"]
  },
  "valid_from": "2026-02-11T14:30:00Z",
  "ttl": "PT1H",
  "agent_id": "iam-agent-v3.0",
  "signature": "0x..."
}
```

**Result values:**

| Result | Meaning |
|--------|---------|
| `SATISFIED` | The assertion holds based on available evidence |
| `NOT_SATISFIED` | The assertion does not hold; evidence contradicts it |
| `PARTIAL` | The assertion holds for some scope but not all |
| `INDETERMINATE` | Insufficient evidence to determine; agent cannot verify |
| `NOT_APPLICABLE` | The assertion does not apply to the subject's architecture |

### 4.3 Opinion

Opinions are first-class objects in OTVP. Unlike current audit frameworks that bury nuance in footnotes, OTVP makes expert judgment explicit, signed, and queryable.

An opinion includes:

- **Assessment** — The agent's analysis of what the evidence means
- **Context** — Why this matters (or doesn't) for the relying party's specific risk
- **Caveats** — What the agent cannot verify, what assumptions it made, what exceptions exist
- **Recommendations** — What would improve the posture (optional, can be suppressed by subject)

Opinions are generated by the agent's reasoning engine (which may be AI-powered, rule-based, or hybrid) and are always traceable to the underlying evidence.

### 4.4 Trust Envelope

The Trust Envelope is the response object returned to a relying party. It bundles claims, evidence references, opinions, and cryptographic proofs into a single verifiable package. See Section 5 for the full specification.

---

## 5. The Trust Envelope

The Trust Envelope is the core deliverable of OTVP. It is the replacement for the SOC 2 report — but it is machine-readable, cryptographically signed, continuously refreshed, and contextual to the asker.

### 5.1 Envelope Structure

```json
{
  "$schema": "https://otvp.org/schema/trust-envelope/v1",
  "envelope_id": "te-2026-02-11-acme-reliant-001",
  "version": "1.0",
  
  "metadata": {
    "generated_at": "2026-02-11T14:30:00Z",
    "valid_until": "2026-02-11T15:30:00Z",
    "query_ref": "tq-2026-02-11-enc-001",
    "subject": {
      "organization": "Acme Corp",
      "otvp_id": "otvp:org:acme-corp",
      "environment": "production"
    },
    "relying_party": {
      "organization": "Reliant Health",
      "otvp_id": "otvp:org:reliant-health",
      "context": {
        "data_classification": "PHI",
        "relationship": "data_processor",
        "regulatory_frameworks": ["HIPAA", "SOC2"]
      }
    }
  },

  "claims": [
    {
      "claim_id": "cl-001",
      "domain": "encryption.at_rest",
      "assertion": "AES-256 encryption on all storage containing relying party data",
      "result": "SATISFIED",
      "confidence": 0.97,
      "evidence_refs": ["ev-001", "ev-002", "ev-003"],
      "opinion": {
        "assessment": "All production storage services (RDS, S3, EBS) verified as AES-256 encrypted. KMS keys are AWS-managed with automatic rotation.",
        "context": "For PHI data processing, this meets HIPAA §164.312(a)(2)(iv) encryption requirements.",
        "caveats": [
          "Development environment uses AES-128; confirmed no PHI data present via data classification agent.",
          "Two legacy S3 buckets use SSE-S3 rather than SSE-KMS. Both are empty and scheduled for deletion (Jira SEC-1234)."
        ],
        "recommendations": [
          "Migrate remaining SSE-S3 buckets to SSE-KMS for consistent key management."
        ]
      },
      "scope": {
        "services": ["RDS", "S3", "EBS"],
        "regions": ["us-east-1", "us-west-2"],
        "accounts": ["prod-main", "prod-data"]
      },
      "agent": {
        "agent_id": "enc-agent-v2.1",
        "certification": "otvp:cert:enc-agent:2026-001",
        "vendor": "TrustForge Inc.",
        "last_calibration": "2026-01-15T00:00:00Z"
      },
      "signature": "0x..."
    }
  ],

  "evidence_summary": {
    "total_evidence_items": 147,
    "evidence_merkle_root": "0x...",
    "collection_window": {
      "start": "2026-02-11T14:00:00Z",
      "end": "2026-02-11T14:28:00Z"
    },
    "evidence_access": {
      "method": "API",
      "endpoint": "https://trust.acme-corp.com/otvp/evidence",
      "authentication": "mutual_tls",
      "note": "Full evidence available to relying parties with executed DPA"
    }
  },

  "composite_score": {
    "overall": "HIGH",
    "by_domain": {
      "encryption.at_rest": { "level": "HIGH", "confidence": 0.97 },
      "encryption.in_transit": { "level": "HIGH", "confidence": 0.99 },
      "access_control.mfa": { "level": "HIGH", "confidence": 0.95 },
      "access_control.least_privilege": { "level": "MEDIUM", "confidence": 0.82 },
      "logging.completeness": { "level": "HIGH", "confidence": 0.91 }
    }
  },

  "envelope_signature": {
    "algorithm": "Ed25519",
    "signer": "otvp:broker:trustnet-primary",
    "signature": "0x...",
    "certificate_chain": ["0x...", "0x..."]
  }
}
```

### 5.2 Envelope Freshness

Every Trust Envelope has a `valid_until` timestamp (TTL). After expiry, the envelope must be re-generated. TTL is determined by:

| Factor | Shorter TTL | Longer TTL |
|--------|------------|------------|
| Data sensitivity | PHI, PCI → 1 hour | Public data → 24 hours |
| Control volatility | IAM changes → 15 min | Encryption config → 4 hours |
| Regulatory requirement | Real-time monitoring → 5 min | Annual audit → 7 days |
| Relying party preference | Paranoid → minimum | Pragmatic → reasonable |

Subjects can configure minimum TTLs per domain. Relying parties can request shorter TTLs up to the agent's collection frequency.

### 5.3 Envelope Privacy

Not all evidence should be visible to all relying parties. OTVP supports scoped disclosure:

**Full disclosure** — The relying party can drill into all evidence supporting a claim. Appropriate for regulated relationships with executed DPAs.

**Claim-only disclosure** — The relying party sees claims, opinions, and scores but cannot access raw evidence. Appropriate for general vendor assessments.

**Zero-knowledge disclosure** — The relying party receives a cryptographic proof that a claim is SATISFIED without seeing any evidence or implementation details. Appropriate for competitive or sensitive environments.

The subject company controls disclosure levels per relying party and per domain. This is configured in the Subject's OTVP policy.

---

## 6. Control Domains

OTVP organizes controls into domains. Domains are hierarchical, extensible, and mapped to existing frameworks for interoperability.

### 6.1 Core Domains

```
otvp:domain:
├── identity_and_access
│   ├── authentication
│   │   ├── mfa_enforcement
│   │   ├── password_policy
│   │   ├── sso_configuration
│   │   └── session_management
│   ├── authorization
│   │   ├── least_privilege
│   │   ├── role_based_access
│   │   ├── privilege_escalation_controls
│   │   └── separation_of_duties
│   └── lifecycle
│       ├── provisioning
│       ├── deprovisioning
│       └── access_reviews
│
├── data_protection
│   ├── encryption
│   │   ├── at_rest
│   │   ├── in_transit
│   │   ├── key_management
│   │   └── key_rotation
│   ├── classification
│   │   ├── labeling
│   │   ├── handling_rules
│   │   └── data_flow_mapping
│   └── retention
│       ├── policy_enforcement
│       └── deletion_verification
│
├── network_security
│   ├── segmentation
│   ├── ingress_controls
│   ├── egress_controls
│   ├── dns_security
│   └── tls_configuration
│
├── infrastructure
│   ├── compute
│   │   ├── hardening
│   │   ├── patch_management
│   │   └── container_security
│   ├── cloud_configuration
│   │   ├── account_isolation
│   │   ├── service_control_policies
│   │   └── resource_tagging
│   └── supply_chain
│       ├── dependency_management
│       ├── sbom_verification
│       └── build_integrity
│
├── detection_and_response
│   ├── logging
│   │   ├── completeness
│   │   ├── integrity
│   │   └── retention
│   ├── monitoring
│   │   ├── alerting_coverage
│   │   ├── response_time
│   │   └── false_positive_rate
│   └── incident_response
│       ├── plan_existence
│       ├── last_test_date
│       └── mean_time_to_respond
│
├── application_security
│   ├── sdlc
│   │   ├── code_review
│   │   ├── static_analysis
│   │   ├── dynamic_testing
│   │   └── dependency_scanning
│   ├── api_security
│   │   ├── authentication
│   │   ├── rate_limiting
│   │   └── input_validation
│   └── multi_tenancy
│       ├── isolation_model
│       ├── data_segregation
│       └── noisy_neighbor_controls
│
├── governance
│   ├── policies
│   │   ├── existence
│   │   ├── review_cadence
│   │   └── acknowledgment
│   ├── risk_management
│   │   ├── assessment_cadence
│   │   ├── remediation_tracking
│   │   └── risk_acceptance_process
│   └── third_party
│       ├── vendor_assessment
│       ├── subprocessor_controls
│       └── concentration_risk
│
└── operational_resilience
    ├── availability
    │   ├── redundancy
    │   ├── failover
    │   └── capacity_planning
    ├── backup
    │   ├── coverage
    │   ├── testing
    │   └── recovery_time
    └── disaster_recovery
        ├── plan_existence
        ├── rto_rpo_verification
        └── last_test_date
```

### 6.2 Framework Mapping

Every OTVP domain maps to existing framework controls for interoperability:

```json
{
  "otvp_domain": "data_protection.encryption.at_rest",
  "framework_mappings": {
    "SOC2_CC": ["CC6.1", "CC6.7"],
    "ISO27001": ["A.10.1.1", "A.10.1.2"],
    "NIST_CSF": ["PR.DS-1"],
    "PCI_DSS": ["3.4", "3.5"],
    "HIPAA": ["§164.312(a)(2)(iv)"],
    "CIS_Controls": ["3.11"]
  }
}
```

This means a relying party can issue a Trust Query using OTVP domain language, SOC 2 control language, or any mapped framework — and receive the same evidence.

### 6.3 Custom Domains

Organizations can register custom domains for industry-specific or proprietary controls:

```
otvp:domain:custom:fintech:
├── pci_scope_isolation
├── cardholder_data_environment
└── tokenization_verification

otvp:domain:custom:healthcare:
├── phi_access_controls
├── minimum_necessary_enforcement
└── patient_consent_verification
```

Custom domains must specify their evidence requirements and can optionally map to core domains.

---

## 7. Verification Agents

### 7.1 What is an Agent?

A Verification Agent is a software system that:

1. **Collects evidence** from the subject company's environment continuously
2. **Evaluates controls** against defined assertions
3. **Produces signed claims** with evidence references and expert opinions
4. **Responds to Trust Queries** with fresh, verifiable Trust Envelopes

Agents are domain-specific. An IAM agent understands identity providers. An encryption agent understands KMS and storage services. A network agent understands firewalls and segmentation.

### 7.2 Agent Architecture

```
┌────────────────────────────────────────────────┐
│              VERIFICATION AGENT                 │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │         COLLECTION ENGINE                 │  │
│  │                                           │  │
│  │  Cloud API Collectors                     │  │
│  │  Configuration Scanners                   │  │
│  │  Log Ingestion                            │  │
│  │  Runtime Probes                           │  │
│  │  Policy Analyzers                         │  │
│  └──────────────┬────────────────────────────┘  │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐  │
│  │         EVIDENCE PROCESSOR                │  │
│  │                                           │  │
│  │  Normalize → Validate → Sign → Store      │  │
│  │  Deduplication │ Anomaly Detection        │  │
│  └──────────────┬────────────────────────────┘  │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐  │
│  │         REASONING ENGINE                  │  │
│  │                                           │  │
│  │  Rule-based evaluation                    │  │
│  │  ML-based pattern analysis                │  │
│  │  LLM-based opinion generation             │  │
│  │  Confidence calibration                   │  │
│  └──────────────┬────────────────────────────┘  │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐  │
│  │         CLAIM GENERATOR                   │  │
│  │                                           │  │
│  │  Assertion evaluation                     │  │
│  │  Evidence linking                         │  │
│  │  Opinion composition                      │  │
│  │  Cryptographic signing                    │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │         INTEGRITY MODULE                  │  │
│  │                                           │  │
│  │  Tamper detection (self-monitoring)       │  │
│  │  Heartbeat to certification authority     │  │
│  │  Code signing verification                │  │
│  │  Isolated execution environment           │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
└────────────────────────────────────────────────┘
```

### 7.3 Agent Requirements

For certification, an agent MUST:

| Requirement | Description |
|------------|-------------|
| **Signed binaries** | Agent code is signed by the vendor and verifiable by the certification authority |
| **Tamper resistance** | Agent detects modification to its own code, configuration, or evidence store |
| **Heartbeat** | Agent sends regular heartbeat to the certification authority proving it is running and unmodified |
| **Evidence integrity** | All evidence is signed at collection time and chained (hash-linked) |
| **Scope limitation** | Agent only accesses the data sources necessary for its domain |
| **Audit logging** | Agent logs its own actions in a tamper-evident log |
| **Failure transparency** | Agent reports when it cannot collect evidence or evaluate a control |
| **Version pinning** | Agent version is recorded in all claims; certification is version-specific |
| **Isolation** | Agent runs in an isolated execution environment (container, TEE, or equivalent) |
| **No data exfiltration** | Agent does not transmit raw customer data outside the subject's environment |

### 7.4 Agent Types

**Tier 1: Automated Agents** — Fully automated evidence collection and evaluation. No human in the loop. Suitable for technical controls with deterministic verification (encryption enabled, MFA enforced, patches applied).

**Tier 2: Augmented Agents** — Automated evidence collection with human-reviewed opinions. Suitable for controls requiring judgment (policy adequacy, risk acceptance appropriateness, incident response effectiveness).

**Tier 3: Expert Agents** — Human experts using agent tooling to collect evidence and produce opinions. Suitable for governance controls, cultural assessments, and complex architectural reviews. These are effectively "continuous auditors" using OTVP tooling instead of spreadsheets.

### 7.5 Agent Marketplace

OTVP is designed for a competitive marketplace of agents:

- Multiple vendors can build agents for the same domain
- Relying parties can express preferences for specific agents or vendors
- Agent certification ensures a baseline of quality; market competition drives excellence
- Agents are scored on accuracy, coverage, freshness, and reliability
- Subject companies choose which certified agents to deploy

This breaks the current auditor oligopoly. Anyone can build a verification agent. If it passes certification, it can compete.

---

## 8. Evidence Specification

### 8.1 Evidence Schema

All evidence conforms to the OTVP Evidence Schema:

```json
{
  "$schema": "https://otvp.org/schema/evidence/v1",
  "evidence_id": "string (UUID)",
  "type": "OBSERVATION | LOG_SAMPLE | POLICY_SNAPSHOT | RUNTIME_CHECK | BEHAVIORAL | ATTESTATION | DERIVED",
  "domain": "string (OTVP domain path)",
  "source": {
    "type": "cloud_api | config_file | log_system | network_probe | human | external_system",
    "provider": "string",
    "identifier": "string (API name, file path, etc.)",
    "region": "string (optional)"
  },
  "observation": {
    "resource": "string (resource identifier)",
    "property": "string (what was checked)",
    "value": "any (what was observed)",
    "expected": "any (optional — what was expected)",
    "metadata": "object (additional context)"
  },
  "collected_at": "ISO 8601 timestamp",
  "collection_duration_ms": "integer",
  "agent_id": "string",
  "agent_version": "string",
  "signature": "string (agent's Ed25519 signature over canonical evidence)",
  "chain": {
    "previous_hash": "string (hash of previous evidence in chain)",
    "sequence_number": "integer"
  }
}
```

### 8.2 Evidence Chain

Evidence items are hash-chained into a Merkle tree. This provides:

- **Tamper evidence** — Any modification to historical evidence breaks the chain
- **Efficient verification** — Relying parties can verify specific evidence items without downloading the entire history
- **Append-only guarantees** — New evidence can only be added, never removed or modified

```
                    Root Hash
                   /         \
              Hash(A+B)    Hash(C+D)
              /     \       /     \
          Hash(A)  Hash(B) Hash(C) Hash(D)
            |        |       |        |
         Ev-001   Ev-002  Ev-003   Ev-004
```

### 8.3 Evidence Retention

| Category | Minimum Retention | Rationale |
|----------|------------------|-----------|
| Active claims | Duration of claim validity + 30 days | Support verification and disputes |
| Historical evidence | 1 year | Trend analysis and regression detection |
| Compliance-relevant | Per regulatory requirement | HIPAA: 6 years, PCI: 1 year, etc. |
| Merkle tree roots | Indefinite | Chain integrity verification |

---

## 9. Trust Queries — The Query Interface

### 9.1 Query Structure

A Trust Query is how a relying party asks a question. It is the replacement for "Send me your SOC 2 report."

```json
{
  "$schema": "https://otvp.org/schema/trust-query/v1",
  "query_id": "tq-2026-02-11-001",
  "relying_party": {
    "otvp_id": "otvp:org:reliant-health",
    "contact": "security@reliant-health.com"
  },
  "subject": {
    "otvp_id": "otvp:org:acme-corp"
  },
  "context": {
    "data_classification": ["PHI", "PII"],
    "relationship": "data_processor",
    "regulatory_frameworks": ["HIPAA"],
    "data_residency": ["us-east-1", "us-west-2"]
  },
  "assertions": [
    {
      "domain": "data_protection.encryption.at_rest",
      "requirement": "algorithm >= AES-256"
    },
    {
      "domain": "data_protection.encryption.key_management.key_rotation",
      "requirement": "interval <= 90 days"
    },
    {
      "domain": "identity_and_access.authentication.mfa_enforcement",
      "requirement": "coverage = all_human_users"
    },
    {
      "domain": "detection_and_response.logging.completeness",
      "requirement": "coverage >= 95%"
    }
  ],
  "disclosure_level": "full | claims_only | zero_knowledge",
  "freshness_requirement": "PT1H",
  "response_format": "trust_envelope | summary | badge"
}
```

### 9.2 Query Types

**Specific Query** — "Is AES-256 encryption enabled on all storage containing my data?" Returns a targeted Trust Envelope.

**Framework Query** — "Show me your HIPAA posture." Returns claims for all OTVP domains mapped to HIPAA controls.

**Comparative Query** — "How has your encryption posture changed in the last 90 days?" Returns trend data with historical evidence references.

**Continuous Query (Subscription)** — "Alert me if MFA coverage drops below 100%." Creates a standing query with push notifications.

**Badge Query** — "Give me an embeddable trust badge for your website." Returns a dynamically-refreshed badge backed by live verification.

### 9.3 Query Language

OTVP defines a query language (OTVP-QL) for expressing complex assertions:

```
# Simple assertion
ASSERT encryption.at_rest.algorithm >= "AES-256"
  SCOPE environment = "production"
  CONFIDENCE >= 0.90

# Composite assertion
ASSERT ALL(
  encryption.at_rest.algorithm >= "AES-256",
  encryption.key_management.rotation_interval <= 90d,
  encryption.in_transit.tls_version >= "1.2"
)
  SCOPE data_classification IN ("PHI", "PCI")
  WITH opinion

# Temporal assertion
ASSERT access_control.mfa_enforcement.coverage = 100%
  CONTINUOUSLY SINCE 2026-01-01
  ALERT ON violation

# Framework assertion
ASSERT FRAMEWORK("HIPAA")
  SCOPE relationship = "business_associate"
  DISCLOSURE full
```

### 9.4 Pre-built Query Templates

To reduce adoption friction, OTVP provides pre-built query templates:

- `otvp:template:hipaa-baa` — All controls relevant for a HIPAA Business Associate Agreement
- `otvp:template:soc2-type2` — All controls that map to SOC 2 Type II criteria
- `otvp:template:pci-service-provider` — Controls for PCI DSS service provider assessment
- `otvp:template:vendor-basic` — Minimum viable vendor security assessment
- `otvp:template:saas-customer` — Standard SaaS customer security diligence

These templates are maintained by the OTVP community and versioned.

---

## 10. Cryptographic Verification Layer

### 10.1 Signing Hierarchy

```
OTVP Root of Trust
    │
    ├── Certification Authority Keys
    │       │
    │       ├── Agent Vendor Signing Keys
    │       │       │
    │       │       └── Individual Agent Instance Keys
    │       │               │
    │       │               └── Evidence Signatures
    │       │
    │       └── Broker Signing Keys
    │               │
    │               └── Trust Envelope Signatures
    │
    └── Domain Registry Keys
            │
            └── Domain Definition Signatures
```

### 10.2 Algorithms

| Purpose | Algorithm | Key Size | Rationale |
|---------|-----------|----------|-----------|
| Evidence signing | Ed25519 | 256-bit | Fast, compact, no RNG dependency |
| Envelope signing | Ed25519 | 256-bit | Consistency with evidence layer |
| Evidence chaining | SHA-256 | 256-bit | Merkle tree hashing |
| Agent attestation | ECDSA P-384 | 384-bit | Hardware security module compatibility |
| Zero-knowledge proofs | zk-SNARKs (Groth16) | - | Claim verification without evidence disclosure |
| Timestamping | RFC 3161 + Ed25519 | - | Non-repudiation of collection time |

### 10.3 Verification Flow

A relying party verifying a Trust Envelope:

```
1. Receive Trust Envelope
2. Verify envelope_signature against broker's public key
3. Verify broker's key chains to OTVP root of trust
4. For each claim:
   a. Verify claim signature against agent's public key
   b. Verify agent's key chains to certified vendor key
   c. Verify vendor key chains to certification authority
   d. Verify agent certification is current (not revoked)
5. For each evidence_ref (if disclosure permits):
   a. Retrieve evidence item
   b. Verify evidence signature against agent's instance key
   c. Verify evidence hash matches Merkle tree path to root
6. Verify evidence_merkle_root in envelope matches Evidence Store
7. Verify timestamps are within acceptable freshness window
8. Verify no certificate revocations in any chain
```

### 10.4 Zero-Knowledge Proofs

For `zero_knowledge` disclosure, the subject company can prove a claim is SATISFIED without revealing any evidence:

```json
{
  "claim_id": "cl-zk-001",
  "domain": "encryption.at_rest",
  "assertion": "algorithm >= AES-256",
  "result": "SATISFIED",
  "proof": {
    "type": "groth16",
    "verification_key": "0x...",
    "proof": "0x...",
    "public_inputs": [
      "SATISFIED",
      "0.97",
      "2026-02-11T14:30:00Z"
    ]
  }
}
```

The relying party can verify the proof is valid without seeing encryption configurations, KMS keys, resource identifiers, or any other implementation details.

---

## 11. Agent Certification & Governance

### 11.1 Certification Process

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Agent      │     │   Submit to  │     │  Technical   │     │  Certified   │
│   Development│────▶│   Cert Body  │────▶│  Evaluation  │────▶│  & Listed    │
└─────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                │
                                                ▼
                                         ┌──────────────┐
                                         │  Continuous   │
                                         │  Monitoring   │
                                         │  (Post-Cert)  │
                                         └──────────────┘
```

### 11.2 Certification Requirements

**Functional Requirements:**

- Agent correctly identifies control state for its domain (tested against reference environments)
- Agent produces valid OTVP evidence and claims
- Agent opinions are accurate and calibrated (confidence scores match actual accuracy)
- Agent handles edge cases (missing data, partial access, conflicting evidence)

**Security Requirements:**

- Agent code passes security review (SAST, DAST, manual review)
- Agent uses signed binaries with verifiable build provenance (SLSA Level 3+)
- Agent implements tamper detection for its own code and data
- Agent runs in isolated execution environment
- Agent does not exfiltrate data beyond signed claims and evidence metadata

**Operational Requirements:**

- Agent maintains heartbeat with certification authority
- Agent reports failures transparently (INDETERMINATE, not false SATISFIED)
- Agent handles version upgrades without evidence chain breaks
- Agent provides reproducible results (same environment = same evidence)

**Calibration Requirements:**

- Agent confidence scores are empirically calibrated
- When an agent says confidence = 0.95, the claim should be correct 95% of the time
- Calibration is tested against labeled datasets and red team exercises
- Mis-calibrated agents lose certification

### 11.3 Certification Levels

| Level | Name | Requirements | Use Case |
|-------|------|-------------|----------|
| L1 | **Basic** | Automated evidence collection, rule-based evaluation, basic tamper detection | Startups, non-regulated industries |
| L2 | **Standard** | L1 + calibrated confidence scoring, Merkle-chained evidence, TEE execution | Most SaaS companies, general compliance |
| L3 | **High Assurance** | L2 + formal verification of agent logic, hardware-backed attestation, continuous red teaming | Financial services, healthcare, government |
| L4 | **Critical** | L3 + multi-agent consensus (2-of-3 agents must agree), air-gapped evidence store, national lab certification | Critical infrastructure, defense, nuclear |

### 11.4 Revocation

Agent certifications can be revoked for:

- Discovery of evidence fabrication or manipulation
- Failure to maintain heartbeat for > 24 hours without explanation
- Discovery of data exfiltration
- Failure to report known vulnerabilities in evidence collection
- Systematic mis-calibration of confidence scores

Revocation is published to the OTVP Certificate Transparency Log (inspired by web PKI CT logs). Relying parties can monitor revocations in real-time.

---

## 12. Trust Levels & Scoring

### 12.1 Scoring Model

OTVP does not produce a single "pass/fail" like current audits. Instead, it produces a multi-dimensional trust assessment:

```json
{
  "trust_assessment": {
    "subject": "otvp:org:acme-corp",
    "assessed_at": "2026-02-11T14:30:00Z",
    "context": "otvp:template:hipaa-baa",
    
    "composite_level": "HIGH",
    
    "dimensions": {
      "control_effectiveness": {
        "level": "HIGH",
        "score": 0.93,
        "description": "Controls are implemented and operating effectively across assessed domains"
      },
      "evidence_freshness": {
        "level": "HIGH",
        "score": 0.98,
        "description": "97% of evidence items collected within the last hour"
      },
      "coverage_completeness": {
        "level": "MEDIUM",
        "score": 0.84,
        "description": "16 of 19 HIPAA-relevant domains fully assessed; 3 domains have INDETERMINATE claims"
      },
      "agent_confidence": {
        "level": "HIGH",
        "score": 0.91,
        "description": "Mean agent confidence across all claims"
      },
      "historical_consistency": {
        "level": "HIGH",
        "score": 0.96,
        "description": "Controls have been consistently SATISFIED for 89 of last 90 days"
      }
    },

    "trend": {
      "30_day": "STABLE",
      "90_day": "IMPROVING",
      "anomalies": [
        {
          "date": "2026-01-23",
          "domain": "access_control.least_privilege",
          "description": "Temporary regression during org restructure. Resolved in 4 hours.",
          "evidence_ref": "ev-regression-001"
        }
      ]
    }
  }
}
```

### 12.2 Trust Levels

| Level | Meaning | Numeric Range |
|-------|---------|---------------|
| `CRITICAL` | Controls are absent or failing; immediate risk | 0.00 - 0.29 |
| `LOW` | Significant gaps in controls or evidence | 0.30 - 0.54 |
| `MEDIUM` | Controls exist but have notable gaps or low confidence | 0.55 - 0.74 |
| `HIGH` | Controls are operating effectively with minor caveats | 0.75 - 0.94 |
| `VERIFIED` | Controls are operating effectively with high confidence and historical consistency | 0.95 - 1.00 |

### 12.3 What the Score Is NOT

The trust score is NOT a guarantee of security. It is a measure of:

- Whether declared controls exist and are operating
- Whether evidence supports the controls' effectiveness
- How fresh and complete the evidence is
- How confident the agents are in their assessments

A company with a VERIFIED score can still be breached. The score means the controls the agents can verify are in place and working. It does not mean the controls are sufficient for every threat, or that there are no unknown unknowns.

---

## 13. Interoperability & Migration

### 13.1 Framework Bridge

Organizations with existing compliance programs can adopt OTVP incrementally. The Framework Bridge maps existing framework evidence to OTVP evidence:

```
SOC 2 Report → OTVP Framework Bridge → OTVP Evidence (ATTESTATION type)
   │
   ├── Extract control assertions from SOC 2
   ├── Map to OTVP domains
   ├── Import as ATTESTATION evidence (lower confidence than OBSERVATION)
   ├── Layer automated agents on top over time
   └── Gradually replace ATTESTATION with OBSERVATION evidence
```

### 13.2 Migration Path

**Phase 1: Shadow Mode (Month 1-3)**
- Deploy OTVP agents alongside existing audit program
- Agents collect evidence but do not replace existing reports
- Compare agent findings with audit findings to build confidence

**Phase 2: Supplementary Mode (Month 3-6)**
- Share OTVP Trust Envelopes with relying parties as a supplement to traditional reports
- Relying parties can use OTVP data to enhance their vendor risk assessments
- Continue existing audit cadence

**Phase 3: Primary Mode (Month 6-12)**
- OTVP Trust Envelopes become the primary trust mechanism
- Traditional audits reduced to annual spot-checks or eliminated
- Relying parties issue Trust Queries directly

**Phase 4: Native Mode (Month 12+)**
- Full OTVP deployment with continuous verification
- No periodic audits; trust is always current
- Real-time alerting for relying parties on posture changes

### 13.3 API Specification

OTVP defines a RESTful API for all interactions:

```
POST   /v1/queries                    Create a Trust Query
GET    /v1/queries/{query_id}         Get query status
GET    /v1/envelopes/{envelope_id}    Retrieve a Trust Envelope
GET    /v1/evidence/{evidence_id}     Retrieve specific evidence
POST   /v1/subscriptions              Create a continuous query subscription
GET    /v1/agents                     List deployed agents
GET    /v1/agents/{agent_id}/status   Agent health and certification status
GET    /v1/domains                    List supported domains
GET    /v1/verify/{envelope_id}       Verify an envelope's cryptographic chain
```

All endpoints require mutual TLS. Authentication is via OTVP organization credentials.

---

## 14. Threat Model of OTVP Itself

OTVP must be adversarially robust. The subject company has an incentive to appear more secure than it is. The protocol must resist:

### 14.1 Threats and Mitigations

| Threat | Description | Mitigation |
|--------|-------------|------------|
| **Agent tampering** | Subject modifies agent code to produce false evidence | Signed binaries, TEE execution, heartbeat monitoring, code attestation |
| **Evidence fabrication** | Subject generates fake API responses for agents to collect | Cross-referencing multiple evidence sources, behavioral anomaly detection, runtime environment attestation |
| **Evidence suppression** | Subject prevents agent from accessing failing controls | Agent reports INDETERMINATE for unreachable resources; relying party sees coverage gaps |
| **Selective deployment** | Subject only deploys agents in well-controlled areas | Coverage completeness scoring; relying parties can see which domains lack agents |
| **Stale evidence replay** | Subject replays old (passing) evidence instead of current | Timestamp verification, nonce-based freshness challenges, time-bound signatures |
| **Agent shopping** | Subject tries multiple agents and only deploys the most lenient | All agent deployments are logged; historical agent changes visible to relying parties |
| **Collusion** | Agent vendor colludes with subject to produce false claims | Multi-vendor agent verification (L4), certification authority audits, whistleblower incentives |
| **Certification authority compromise** | A CA is compromised or corrupted | Federated CA model (no single point of trust), CT log transparency, community governance |
| **Network manipulation** | Subject intercepts agent communications | Mutual TLS, certificate pinning, signed heartbeats |
| **Inference attacks** | Relying party uses evidence to reverse-engineer subject's architecture | Disclosure levels, zero-knowledge proofs, evidence redaction policies |

### 14.2 Red Team Requirements

L2+ certified agents must undergo annual red team exercises:

- Can the subject company force a SATISFIED result when controls are failing?
- Can the subject company suppress INDETERMINATE or NOT_SATISFIED results?
- Can evidence be fabricated without detection?
- Can the agent be modified without heartbeat failure?

Red team results are published (in aggregate) by the certification authority.

---

## 15. Implementation Guide

### 15.1 For Subject Companies

**Step 1: Inventory** — Map your existing controls to OTVP domains. Identify which domains you can support with automated evidence collection.

**Step 2: Select agents** — Choose certified agents for your priority domains. Start with high-value, easy-to-automate domains: encryption, IAM, logging.

**Step 3: Deploy** — Install agents in your environment. Grant them read-only access to the data sources they need. Configure evidence retention.

**Step 4: Configure disclosure** — Set disclosure levels per domain and per relying party category. Start with claims-only; expand to full disclosure as you build confidence.

**Step 5: Publish** — Register your OTVP endpoint. Begin accepting Trust Queries.

**Step 6: Iterate** — Expand agent coverage. Add domains. Reduce TTLs. Move from shadow mode to native mode.

### 15.2 For Relying Parties

**Step 1: Define your risk context** — What data do you share with this vendor? What regulatory frameworks apply? What controls matter most?

**Step 2: Build or select a query template** — Use a pre-built template or create a custom Trust Query.

**Step 3: Query** — Send the query. Receive a Trust Envelope. Verify the cryptographic chain.

**Step 4: Evaluate** — Review claims, opinions, and scores. Drill into evidence where disclosure permits. Compare against your risk appetite.

**Step 5: Subscribe** — Set up continuous queries for critical vendors. Get alerted when posture changes.

### 15.3 For Agent Developers

**Step 1: Choose a domain** — Pick a control domain you have deep expertise in.

**Step 2: Build** — Implement the OTVP Agent SDK. Build collection, evaluation, and claim generation for your domain.

**Step 3: Test** — Validate against OTVP reference environments. Calibrate confidence scores against labeled datasets.

**Step 4: Certify** — Submit to a certification authority. Pass functional, security, and calibration reviews.

**Step 5: Publish** — List on the OTVP Agent Registry. Compete on accuracy, coverage, and reliability.

### 15.4 Reference Implementation

The OTVP project maintains open-source reference implementations:

- `otvp-agent-sdk` — SDK for building verification agents (Python, Go, Rust)
- `otvp-evidence-store` — Reference implementation of the Merkle-chained evidence store
- `otvp-broker` — Reference Trust Broker implementation
- `otvp-verifier` — Client library for verifying Trust Envelopes
- `otvp-ref-agents` — Reference agents for common domains (AWS IAM, encryption, logging)
- `otvp-query-builder` — Interactive Trust Query builder
- `otvp-dashboard` — Web dashboard for subjects and relying parties

### 15.5 Endpoint Discovery & DNS Verification

OTVP endpoints are published using the `.well-known` URI convention ([RFC 8615](https://www.rfc-editor.org/rfc/rfc8615)). DNS TXT records provide an out-of-band trust anchor that binds a domain to a specific public key, independent of the web server.

#### 15.5.1 Discovery Endpoint

Subject companies MUST publish a configuration document at:

```
https://{domain}/.well-known/otvp/otvp-config.json
```

The configuration document has the following structure:

```json
{
  "otvp_version": "1.0",
  "otvp_id": "otvp:org:acme-corp",
  "organization": "Acme Corp",
  "contact_email": "security@acme-corp.com",
  "public_keys": [
    {
      "kid": "acme-2026-primary",
      "algorithm": "Ed25519",
      "public_key": "<base64-encoded raw Ed25519 public key>",
      "valid_from": "2026-01-01T00:00:00Z",
      "valid_until": "2027-01-01T00:00:00Z",
      "revoked": false
    }
  ],
  "endpoints": {
    "envelopes": "/.well-known/otvp/envelopes/index.json",
    "latest": "/.well-known/otvp/envelopes/latest.json"
  },
  "domains_covered": [
    "data_protection.encryption.at_rest",
    "identity_and_access.authentication.mfa_enforcement"
  ],
  "retention_days": 365,
  "refresh_interval_seconds": 3600
}
```

**Required fields:** `otvp_version`, `otvp_id`, `organization`, `public_keys` (at least one non-revoked key), `endpoints`.

**Envelope endpoints:** The configuration references two retrieval patterns:
- `envelopes` — An index document listing individual envelope files (preferred for granular access)
- `latest` — A single document containing the most recent envelopes (convenience endpoint)

#### 15.5.2 DNS TXT Record Format

Subject companies SHOULD publish a DNS TXT record at:

```
_otvp.{domain}. IN TXT "v=otvp1; fp={sha256-hex}; kid={key_id}; org={otvp_id}"
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `v=otvp1` | MUST | Protocol version identifier. Follows the convention established by SPF (`v=spf1`), DKIM (`v=DKIM1`), and DMARC (`v=DMARC1`). |
| `fp=` | MUST | SHA-256 hex digest (64 characters) of the raw Ed25519 public key bytes (32 bytes). |
| `kid=` | MUST | Key ID matching the `kid` field in the `public_keys` array of the `otvp-config.json` document. |
| `org=` | SHOULD | OTVP organization ID. Provides a cross-check against the config's `otvp_id`. |

**Example:**

```
_otvp.acme-corp.com. 3600 IN TXT "v=otvp1; fp=9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08; kid=acme-2026-primary; org=otvp:org:acme-corp"
```

**Size constraint:** A single DNS TXT string is limited to 255 bytes. The OTVP record format uses approximately 148 characters, well within this limit.

**Generating the fingerprint:** The fingerprint is computed as follows:

```
fingerprint = hex(SHA-256(raw_ed25519_public_key_bytes))
```

Where `raw_ed25519_public_key_bytes` is the 32-byte raw public key (the same bytes that are base64-encoded in the `public_key` field of `otvp-config.json`).

#### 15.5.3 Verification Procedure

Relying parties SHOULD verify DNS records when discovering a new vendor. The verification procedure is:

1. Fetch and validate `otvp-config.json` from the vendor's `.well-known/otvp/` endpoint.
2. Query DNS for TXT records at `_otvp.{domain}`.
3. For each TXT record beginning with `v=otvp1`:
   a. Parse the semicolon-delimited fields (`fp`, `kid`, `org`).
   b. Locate the key in `otvp-config.json` where `public_keys[].kid` matches the record's `kid`.
   c. Compute `SHA-256` of that key's raw public key bytes.
   d. Compare the hex digest to the record's `fp` value.
4. If any active (non-revoked) key matches any TXT record: **DNS verified**.
5. If the `org` field is present, verify it matches `otvp-config.json`'s `otvp_id`.

**Verification states:**

| State | Meaning | Recommended UI |
|-------|---------|----------------|
| **Verified** | At least one active key fingerprint matches a TXT record | Green indicator |
| **Not configured** | No `_otvp` TXT records exist for this domain | Yellow warning — trust relies solely on HTTPS |
| **Mismatch** | TXT records exist but no fingerprint matches any active key | Red warning — possible key compromise or stale DNS |

**Browser-based implementations:** Web applications cannot perform raw DNS queries. Implementations SHOULD use DNS-over-HTTPS (DoH) providers such as:
- Cloudflare: `https://cloudflare-dns.com/dns-query?name=_otvp.{domain}&type=TXT` (header: `Accept: application/dns-json`)
- Google: `https://dns.google/resolve?name=_otvp.{domain}&type=TXT`

Implementations SHOULD use at least two DoH providers as fallbacks to mitigate provider outages.

#### 15.5.4 Key Rotation

Key rotation requires coordination between the config endpoint and DNS:

1. **Add the new key** to `otvp-config.json`'s `public_keys` array (both old and new keys active).
2. **Add a second TXT record** at `_otvp.{domain}` with the new key's fingerprint and `kid`.
3. **Wait for DNS propagation** (at least 2× the DNS TTL).
4. **Revoke the old key** in `otvp-config.json` (set `revoked: true`).
5. **Remove the old TXT record** after a grace period.

Multiple TXT records at `_otvp.{domain}` are valid during rotation. Verification passes if ANY active key matches ANY record.

A recommended DNS TTL for `_otvp` records is 3600 seconds (1 hour). Lower TTLs speed up rotation but increase DNS query volume.

#### 15.5.5 Trust Model Layers

DNS verification is one layer in a defense-in-depth trust model:

| Layer | Mechanism | Question Answered |
|-------|-----------|-------------------|
| **0 — Domain Control** | DNS TXT record at `_otvp.{domain}` | Does the domain admin endorse this public key? |
| **1 — Endpoint Discovery** | `.well-known/otvp/otvp-config.json` over HTTPS | What keys, endpoints, and domains does this vendor support? |
| **2 — Envelope Integrity** | Ed25519 signatures, Merkle trees, PKI chain | Is this envelope authentic and untampered? |

Missing DNS (Layer 0) degrades trust but does not prevent verification. Relying parties MAY accept envelopes from vendors without DNS verification, but SHOULD display the reduced trust level to users.

#### 15.5.6 DNSSEC

Implementations SHOULD support DNSSEC validation but MUST NOT require it. When using DoH providers, the JSON response includes an `AD` (Authenticated Data) flag indicating whether the DNS response was DNSSEC-validated. Implementations MAY display DNSSEC status as an additional trust signal.

#### 15.5.7 Subdomain Delegation

If a vendor serves OTVP from a subdomain (e.g., `security.acme-corp.com`), the TXT record MUST be placed at `_otvp.security.acme-corp.com`, matching the domain from which `otvp-config.json` is served.

---

## 16. Governance & Standards Body

### 16.1 The OTVP Foundation

OTVP is governed by an open foundation with the following structure:

**Technical Steering Committee (TSC)** — Owns the specification. Merges changes. Manages versioning. Composed of elected members from contributing organizations.

**Certification Board** — Approves and revokes agent certifications. Approves new certification authorities. Manages red team programs.

**Domain Working Groups** — Maintain domain definitions, evidence requirements, and framework mappings. Open to anyone with domain expertise.

**Community** — Anyone can propose changes, build agents, or contribute to reference implementations. Governance is transparent and meritocratic.

### 16.2 Specification Versioning

OTVP uses semantic versioning:

- **Major versions** (v2.0) — Breaking changes to evidence schema, query format, or verification protocol. 18-month migration window.
- **Minor versions** (v1.1) — New domains, new evidence types, new query features. Backward compatible.
- **Patch versions** (v1.0.1) — Clarifications, typo fixes, reference implementation updates.

### 16.3 Intellectual Property

All OTVP specifications, schemas, and reference implementations are released under Apache 2.0. No patents. No royalties. No vendor lock-in.

Agent vendors can build proprietary agents on top of the open standard. The standard itself remains free and open.

---

## 17. Appendices

### Appendix A: Comparison with Existing Frameworks

| Dimension | SOC 2 | ISO 27001 | OTVP |
|-----------|-------|-----------|------|
| Evidence type | Auditor samples | Auditor samples | Continuous, automated, multi-source |
| Freshness | 3-12 months stale | Annual snapshot | Real-time (configurable TTL) |
| Verifiability | Trust the auditor | Trust the auditor | Cryptographically verifiable |
| Queryability | Read the report | Read the report | Programmable queries, contextual results |
| Granularity | Pass/fail with exceptions | Certified/not certified | Multi-dimensional scoring with confidence intervals |
| Cost structure | $50K-$500K per audit | $20K-$200K per audit | Agent licensing + hosting (projected: lower and falling) |
| Incentive alignment | Subject pays auditor | Subject pays auditor | Relying party selects agents; subject deploys |
| Machine readability | PDF | PDF/Word | JSON, API, webhooks |
| Contextual to asker | No | No | Yes — scoped to relying party's data and risk |
| Continuous monitoring | No | No (unless supplemented) | Yes — native design |
| Time to answer | Days to weeks | Days to weeks | Seconds to minutes |

### Appendix B: OTVP Evidence Types — Detailed

| Type | Source | Automation | Example | Confidence Typical Range |
|------|--------|-----------|---------|-------------------------|
| OBSERVATION | Cloud APIs, config files | Full | S3 encryption setting via AWS API | 0.90 - 0.99 |
| LOG_SAMPLE | Audit logs, SIEM | Full | CloudTrail showing key rotation event | 0.80 - 0.95 |
| POLICY_SNAPSHOT | IAM, SCP, network rules | Full | Captured IAM policy document | 0.85 - 0.98 |
| RUNTIME_CHECK | Active testing | Full | TLS handshake test result | 0.90 - 0.99 |
| BEHAVIORAL | Pattern analysis over time | Full | Anomalous access pattern detection | 0.60 - 0.85 |
| ATTESTATION | Human input, external reports | Manual | Signed pen test summary hash | 0.50 - 0.80 |
| DERIVED | Multi-evidence reasoning | Full/Hybrid | "All prod DBs encrypted" from 47 OBSERVATIONs | 0.70 - 0.95 |

### Appendix C: OTVP-QL Grammar (Simplified)

```ebnf
query          = assertion+
assertion      = "ASSERT" (simple_assert | composite_assert) scope? confidence? options*
simple_assert  = domain_path operator value
composite_assert = ("ALL" | "ANY" | "NONE") "(" simple_assert ("," simple_assert)* ")"
domain_path    = identifier ("." identifier)*
operator       = "=" | "!=" | ">" | ">=" | "<" | "<="
value          = string | number | boolean | percentage | duration
scope          = "SCOPE" scope_expr ("AND" scope_expr)*
scope_expr     = identifier ("=" | "IN" | "NOT IN") value_list
confidence     = "CONFIDENCE" operator number
options        = with_option | temporal_option | alert_option
with_option    = "WITH" ("opinion" | "evidence" | "trend")
temporal_option = "CONTINUOUSLY" "SINCE" datetime
alert_option   = "ALERT" "ON" ("violation" | "degradation" | "any_change")
```

### Appendix D: Glossary

| Term | Definition |
|------|-----------|
| **Agent** | A certified software system that collects evidence and produces signed claims about a specific control domain |
| **Assertion** | A specific, verifiable statement about a control (e.g., "MFA is enforced for all users") |
| **Claim** | An agent's evaluated assertion, including result, evidence references, confidence score, and opinion |
| **Confidence** | A calibrated probability that a claim's result is correct |
| **Domain** | A hierarchical category of security controls (e.g., `encryption.at_rest`) |
| **Evidence** | A cryptographically signed record of an observed fact |
| **Evidence Chain** | A Merkle-tree-linked sequence of evidence items ensuring tamper evidence |
| **Opinion** | An agent's expert analysis of what evidence means in context, including caveats and recommendations |
| **Relying Party** | The organization consuming Trust Envelopes to make trust decisions |
| **Subject** | The organization whose controls are being verified |
| **Trust Broker** | An intermediary that routes queries, validates signatures, and manages access |
| **Trust Envelope** | The complete response to a Trust Query, containing claims, evidence references, and cryptographic proofs |
| **Trust Query** | A structured question from a relying party about a subject's controls |
| **TTL** | Time-to-live; how long a Trust Envelope or claim remains valid before re-verification |

### Appendix E: Regulatory Mapping Index

| Regulation | OTVP Template | Key Domains |
|-----------|---------------|-------------|
| HIPAA | `otvp:template:hipaa-baa` | data_protection, identity_and_access, detection_and_response, governance |
| SOC 2 Type II | `otvp:template:soc2-type2` | All core domains |
| PCI DSS v4.0 | `otvp:template:pci-service-provider` | network_security, data_protection, identity_and_access, application_security |
| ISO 27001:2022 | `otvp:template:iso27001` | All core domains |
| GDPR | `otvp:template:gdpr-processor` | data_protection, governance, identity_and_access |
| FedRAMP | `otvp:template:fedramp-moderate` | All core domains + custom:gov domains |
| NIST CSF 2.0 | `otvp:template:nist-csf` | All core domains |
| SOX | `otvp:template:sox-itgc` | identity_and_access, governance, detection_and_response |

---

## Contributing

OTVP is an open standard. Contributions welcome.

- **Specification changes** — Open an RFC in the spec repo. Discuss in the working group. TSC votes on inclusion.
- **New domains** — Propose in the Domain Working Group. Define evidence requirements and framework mappings.
- **Reference implementations** — PRs welcome. Follow the contribution guide.
- **Agent development** — Build agents. Get certified. List them.
- **Feedback** — Open issues. Join the community.

---

## License

This specification is released under the Apache License 2.0.

You are free to implement, extend, and build commercial products on top of this standard without royalty or restriction.

---

*"The best audit report is one that never goes stale."*

— OTVP Project
