# ArcSight Documentation

Welcome to the ArcSight project documentation. This repository contains comprehensive documentation for the ArcSight system architecture, development phases, and migration strategies.

**ArcSight Constitution:** see [`docs/CONSTITUTION.md`](./docs/CONSTITUTION.md).  
**ArcSight v1 Scope:** see [`docs/V1_SCOPE.md`](./docs/V1_SCOPE.md).

> **📋 Start here:** [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md) — The canonical root document that defines the complete documentation structure, authoring rules, and governance.

---

## Document Tiers

ArcSight documentation is organized into three tiers:

- **Tier A (Architectural)** — Strategic guides exempt from contract structure (Docs 01–05)
- **Tier B (Contracts)** — Enforceable technical contracts with required 10-section structure (Docs 5A–20)
- **Tier C (Operational)** — Workflows, protocols, and operational guides (Doc 10C, README)

**Important:** Docs 01–05 are **exempt** from the 10-section contract template. They use flexible, narrative structures and MUST NOT be rewritten to match contract format.

---

## Complete Documentation Index

### Tier A — Architectural Guides (Exempt from 10-Section Structure)

These documents define high-level philosophy, strategy, and architecture. They use flexible, narrative structures.

- **[00 — Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md)**  
  Foundation document defining documentation structure, authoring rules, and governance

- **[01 — Strategic Architecture](./01-decision-phase1-vs-phase2.md)**  
  Phase 1 vs Phase 2 structure decision — when to stay separate vs migrate to monorepo

- **[02 — Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)**  
  Phase 1 folder scaffold — where code lives and architectural boundaries

- **[03 — Full System Roadmap](./03-full-system-roadmap.md)**  
  Full system roadmap — what to build, when, and in what order (CTO sign-off)

- **[04 — Usage Model](./04-usage-model.md)**  
  How documents 1, 2, and 3 work together — which document to use when

- **[05B — Monorepo Migration](./5B-monorepo-migration.md)**  
  Strategy and guide for migrating to a monorepo structure

### Tier B — Enforceable Technical Contracts (10-Section Structure Required)

These documents define formal, enforceable rules and must follow the 10-section contract structure.

#### Dependency & Architecture Contracts

- **[5A — Dependency Contract](./5A-dependency-contract.md)**  
  Cross-package dependency rules for Phase 1 & Phase 2, engine purity requirements, and forbidden imports

#### Invariant System (Complete Suite)

- **[06 — Invariants Contract](./06-invariants-contract.md)**  
  Deterministic conditions that MUST hold across engine, runtime, Sentinel & drift detection

- **[06A — Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)**  
  Formal test suite required to validate all invariants with unit, integration, and golden tests

- **[06B — Error Codes Catalog](./06B-error-codes-catalog.md)**  
  Stable machine-readable error codes for all invariant violations

- **[06C — Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md)**  
  CI gate specification that guarantees invariants are always enforced

- **[06X — Invariant System Index](./06X-invariant-system-index.md)**  
  Master reference unifying the entire invariant subsystem (06, 06A, 06B, 06C)

#### Core Determinism & Schema Contracts

- **[07 — Determinism Contract](./07-determinism-contract.md)**  
  Engine determinism requirements, canonicalization rules, and forbidden sources of non-determinism

- **[08 — Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)**  
  Schema versioning model, adapter chain rules, and backward compatibility

- **[09 — Golden Test Governance](./09-golden-test-governance.md)**  
  Golden test creation, updates, drift detection, and stability rules

- **[10 — Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)**  
  Formal boundary contract between deterministic engine and runtime orchestrator

- **[11 — Schema Evolution Contract](./11-schema-evolution-contract.md)**  
  Envelope format evolution rules, adapter pipeline, and versioning governance

#### Rulepack & Limits Contracts

- **[12 — Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)**  
  Rulepack semantic versioning, namespace governance, purity guarantees, and deterministic execution

- **[13 — Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)**  
  Snapshot and analysis limits, degraded mode rules, deterministic truncation, and sandbox guarantees

#### Input/Output Contracts

- **[14 — RepoSnapshot Contract](./14-repo-snapshot-contract.md)**  
  Canonical RepoSnapshot format, deterministic construction rules, path/content normalization, and binary handling

- **[15 — Envelope Format Specification](./15-envelope-format-spec.md)**  
  Canonical Envelope format, deterministic output structure, signature rules, and extension namespace governance

#### Phase 2 Contracts

- **[16 — Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)**  
  Shadow analyzer execution, deterministic drift classification, promotion gating, and envelope comparison rules

- **[17 — Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)**  
  Formal promotion lifecycle (shadow → canary → production), manual approval requirements, and deterministic gating rules

- **[18 — Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)**  
  Enterprise/partner rulepack namespace rules, isolation requirements, determinism guarantees, and JSON-serialization constraints

- **[19 — Drift Detection Contract](./19-drift-detection-contract.md)**  
  Deterministic drift classification rules, invariant validation, unknown extension preservation, and promotion gating

- **[20 — Deterministic Config Contract](./20-deterministic-config-contract.md)**  
  AnalyzerConfig normalization, deterministic hashing, shadow/live equivalence, locale independence, and immutable config rules

### Tier C — Operational Documents

- **[10C — Cursor Development Protocol](./10C-cursor-development-protocol.md)**  
  Operational specification for AI-assisted development, defining how Cursor must operate within ArcSight

---

## Documentation Dependency Diagram

```
┌─────────────────────────────────────────────────────────┐
│  00 — Documentation Index & Authoring Protocol          │
│  (Root governance document)                             │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│ Tier A       │  │ Tier B       │  │ Tier C       │
│ (01-05)      │  │ (5A-20)      │  │ (10C)        │
│              │  │              │  │              │
│ Architectural│  │ Contracts    │  │ Operational  │
│ Guides       │  │              │  │ Protocols    │
│              │  │              │  │              │
│ Exempt from  │  │ 10-section   │  │ Flexible     │
│ 10-section   │  │ structure    │  │ structure    │
│ structure    │  │ required     │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│ Invariant    │  │ Core         │  │ Phase 2      │
│ System       │  │ Contracts    │  │ Contracts   │
│ (06-06X)     │  │ (07-15)      │  │ (16-20)     │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Quick Navigation

### For New Engineers

**Start here:** [Usage Model](./04-usage-model.md) → [Strategic Architecture](./01-decision-phase1-vs-phase2.md) → [Physical Architecture](./02-phase1-folder-scaffold.md) → [System Roadmap](./03-full-system-roadmap.md)

### For Daily Development

- **What to build**: [System Roadmap](./03-full-system-roadmap.md)
- **Where code goes**: [Physical Architecture](./02-phase1-folder-scaffold.md)
- **Which doc to use**: [Usage Model](./04-usage-model.md)
- **Determinism rules**: [Determinism Contract](./07-determinism-contract.md)
- **Invariants**: [Invariants Contract](./06-invariants-contract.md), [Invariant System Index](./06X-invariant-system-index.md)
- **Schema changes**: [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md), [Schema Evolution Contract](./11-schema-evolution-contract.md)
- **Rulepacks**: [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- **Limits**: [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)
- **Snapshots**: [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- **Envelopes**: [Envelope Format Specification](./15-envelope-format-spec.md)
- **Golden tests**: [Golden Test Governance](./09-golden-test-governance.md)
- **Engine/Runtime boundary**: [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)

### For Planning

- **Migration strategy**: [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- **Monorepo details**: [Monorepo Migration](./5B-monorepo-migration.md)
- **Phase 2 Sentinel**: [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- **Phase 2 Promotion**: [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- **Phase 2 Enterprise**: [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)
- **Phase 2 Drift**: [Drift Detection Contract](./19-drift-detection-contract.md)
- **Phase 2 Config**: [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## How to Add a New Document

1. **Determine the tier:**
   - **Tier A** — Architectural/strategic guide (exempt from 10-section structure)
   - **Tier B** — Technical contract (MUST follow 10-section structure)
   - **Tier C** — Operational protocol (flexible structure)

2. **Assign a number:**
   - Check [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md) for available numbers
   - Use suffixes for expansions (e.g., 06A, 06B, 06C, 06X)
   - Never reuse existing numbers

3. **Follow the structure:**
   - **Tier A** — Flexible, narrative structure
   - **Tier B** — Must include all 10 required sections (see [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md) §4)
   - **Tier C** — Flexible but structured

4. **Update cross-references:**
   - Add the new document to Doc #00 index
   - Update this README
   - Add appropriate cross-references in related documents

5. **Add to change log:**
   - Document the addition in Doc #00 change log
   - Add change log entry to the new document

---

## How Cursor Should Use These Docs

**Cursor MUST follow:** [Cursor Development Protocol](./10C-cursor-development-protocol.md)

**Key rules:**

- **Tier A documents (01–05)** — Cursor MUST NOT attempt to restructure these documents to match contract format
- **Tier B documents (5A–20)** — Cursor MUST maintain the 10-section structure
- **Before generating code** — Cursor MUST validate against all relevant contracts
- **Cross-references** — Cursor MUST cite exact document sections when making architectural changes
- **Determinism** — Cursor MUST never introduce non-deterministic behavior

See [Cursor Development Protocol](./10C-cursor-development-protocol.md) for complete rules.

---

## Document Categories Summary

| Category | Documents | Tier | Structure | Purpose |
|----------|-----------|------|-----------|---------|
| **Meta** | 00 | Meta | N/A | Foundation & definitions |
| **Architecture** | 01–05 | A | Flexible | Strategic guides |
| **Dependency** | 5A | B | 10-section | Cross-package rules |
| **Invariants** | 06, 06A–06X | B | 10-section | Correctness guarantees |
| **Core Contracts** | 07–15 | B | 10-section | Engine, schema, I/O |
| **Phase 2** | 16–20 | B | 10-section | Shadow, promotion, drift |
| **Operational** | 10C | C | Flexible | Development protocols |

---

## Important Notes

- **Docs 01–05 are exempt** from the 10-section contract structure. They use narrative, flexible formats.
- **All Tier B documents (5A–20)** must follow the 10-section structure defined in [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md).
- **Cross-references** must use relative paths and exact document numbers.
- **No external links** are permitted in documentation.
- **Change logs** are append-only and mandatory for all documents.
- **Documentation Consistency Monitor** runs automatically after every push via GitHub Actions to validate cross-references, structure compliance, and file existence.

For complete authoring rules, see [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md).

---

## CI/CD Integration

### Documentation Consistency Monitor

The **ArcSight Documentation Consistency Monitor** runs automatically after every push to validate:

- ✅ Cross-reference integrity (no broken links)
- ✅ Structure compliance (Tier B documents have required sections)
- ✅ File existence (all required documents present)
- ✅ Link validation (all internal links point to existing files)

**Workflow:** `.github/workflows/docs-consistency-monitor.yml`

**Status:** The monitor will fail CI if any documentation inconsistencies are detected, preventing broken cross-references or structural violations from being merged.
