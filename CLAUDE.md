# CLAUDE.md — OTVP Specification

## What This Repo Is

The **Open Transparency Verification Platform specification** — this defines the OTVP standard itself. It is not code. It is the authoritative document that describes how trust envelopes work, what controls are verified, how agents should behave, and how results are structured.

## Repository Structure

```
otvp/
├── OTVP-Specification-v1.0.md    # The full spec (65KB) — primary document
├── README.md                      # Project overview and quickstart
├── LICENSE
├── CONTRIBUTING.md                # How to contribute to the spec
├── GOVERNANCE.md                  # Governance model for the standard
├── agents/
│   └── reference/                 # (empty) Placeholder for reference agent implementations
├── mappings/                      # (empty) Placeholder for control-to-framework mappings
├── schemas/                       # (empty) Placeholder for envelope JSON schemas
├── spec/                          # (empty) Placeholder for supplementary spec docs
├── examples/
│   ├── envelopes/                 # (empty) Placeholder for example envelope JSON files
│   └── queries/                   # (empty) Placeholder for example queries
└── CLAUDE.md
```

**Note:** Most subdirectories are empty placeholders for future content. The real substance is in `OTVP-Specification-v1.0.md`.

## Key Concepts

- **Trust Envelope** — a signed, point-in-time attestation of a specific security control's status
- **Control Criteria** — the 11 security checks mapped to SOC 2 Common Criteria (CC6.x, CC7.x, CC9.x)
- **Agent** — an autonomous process that evaluates infrastructure against a control criterion and produces an envelope
- **Killswitch Advisory** — Bil's advisory firm, used as the organization identifier in envelopes

## Relationship to Other Repos

- **otvp-sdk** implements this spec — Python agents that produce trust envelopes
- **otvp-app** displays the envelopes produced by the SDK
- **otvp-dashboard** is deprecated static UI
- See `~/otvp-projects/CLAUDE.md` for the full system overview

## Working on This Repo

This is a **documentation repo**. Changes here are spec changes, not code changes. When editing:
- The spec doc is the source of truth — if the SDK disagrees with the spec, the spec wins
- Placeholder directories should be populated as the project matures (example envelopes, JSON schemas, reference implementations)
- Changes to control criteria or envelope structure here should be reflected in otvp-sdk

## Owner Context

Bil Harmer — CISO at Supabase, founder of Killswitch Advisory. This spec is designed to be an open standard that others could implement, not just an internal tool. Treat it accordingly — clarity and precision matter.
