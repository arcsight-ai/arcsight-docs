# ArcSight Documentation Index & Authoring Protocol

## FINAL VERSION — COMPLETE SYSTEM GOVERNANCE DOCUMENT

This document defines:

- The complete ArcSight documentation index
- The required structure for all technical contracts
- The rules for adding, editing, and versioning docs
- The cross-reference governance
- The Cursor development authoring protocol

It is the root of truth for all ArcSight documentation.

---

## 1. Purpose

The purpose of this document is to:

- Establish the canonical documentation structure
- Define authoring constraints
- Prevent documentation drift
- Maintain determinism across all specs
- Ensure contract-level documents remain stable and mechanically checkable
- Guide Cursor-automated writing safely

This document MUST be applied to every future document unless explicitly exempt.

---

## 2. Scope

This protocol governs:

✔ All contracts (runtime, engine, schema, invariants, rulepacks, drift, promotion)  
✔ All architectural docs (Phase 1 vs Phase 2 decisions, roadmaps, usage model)  
✔ All invariant subsystem docs (06, 06A, 06B, 06C, 06X)  
✔ All future enterprise extensions  
✔ All documentation residing under `/ARSIGHT-DOCS/`  

**Not governed:**

✘ Source code  
✘ Inline comments  
✘ Generated documentation  

---

## 3. Document Categories

ArcSight documentation is divided into three tiers:

### 3.1 Tier A — Architectural Guides (EXEMPT from 10-section rule)

These documents define high-level philosophy and should not follow the contract structure:

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md) (01)
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) (02)
- [Full System Roadmap](./03-full-system-roadmap.md) (03)
- [Usage Model](./04-usage-model.md) (04)
- [Monorepo Migration Guide](./5B-monorepo-migration.md) (05B)

These documents are narrative, conceptual, and directional.

**CRITICAL EXEMPTION RULES:**

✔ **Docs 01–05 are Tier A Strategic/Architectural documents**  
✔ **They are EXEMPT from the 10-section Contract template**  
✔ **They MUST NOT be rewritten to match the contract structure**  
✔ **They may have flexible structure, narrative sections, diagrams, etc.**  
✔ **Cursor MUST NOT attempt to "fix" or restructure these documents**  
✔ **CI validation MUST NOT flag these documents as invalid**  

**They are exempt from "Purpose / Scope / Definitions / Determinism Impact / Boundary Impact / Versioning / Testing" requirements.**

This exemption prevents:
- validation issues
- refactoring confusion
- Cursor misinterpretation
- future rewrite attempts

### 3.2 Tier B — Enforceable Technical Contracts (MUST follow 10-section rule)

These govern engine, runtime, deterministic guarantees, schema, rulepacks, limits, invariants, drift, etc.

**Examples:**

- [Dependency Contract](./5A-dependency-contract.md) (5A)
- [Invariants Contract](./06-invariants-contract.md) (06)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md) (06A)
- [Error Codes Catalog](./06B-error-codes-catalog.md) (06B)
- [Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md) (06C)
- [Invariant System Index](./06X-invariant-system-index.md) (06X)
- [Determinism Contract](./07-determinism-contract.md) (07)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md) (08)
- [Golden Test Governance](./09-golden-test-governance.md) (09)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md) (10)
- [Schema Evolution Contract](./11-schema-evolution-contract.md) (11)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md) (12)
- [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md) (13)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) (14)
- [Envelope Format Spec](./15-envelope-format-spec.md) (15)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md) (16)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md) (17)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md) (18)
- [Drift Detection Contract](./19-drift-detection-contract.md) (19)
- [Deterministic Config Contract](./20-deterministic-config-contract.md) (20)

**All Tier B documents must comply with the 10-section authoring protocol below.**

### 3.3 Tier C — Operational Docs, Readmes, Protocol Helpers

Includes:

- Cursor Development Protocol (10C) ← to be created
- repo README files
- developer onboarding
- runbooks
- diagrams

These should be structured but are not enforceable contracts.

---

## 4. Required Structure for All Tier B Contracts

**IMPORTANT:** This section applies ONLY to Tier B documents (5A–20).  
**Tier A documents (01–05) are EXEMPT and MUST NOT be restructured.**

Every Tier B document MUST contain these 10 sections in order:

1. **Purpose**
2. **Scope**
3. **Definitions & Terms**
4. **Rules / Contract**
5. **Determinism Impact**
6. **Runtime / Engine Boundary Impact**
7. **Versioning Impact**
8. **Testing Requirements**
9. **Cross-References**
10. **Change Log (Append-Only)**

If any section is missing, CI (and Cursor validation tools) MUST treat it as an error.

**Document Tier Reference:**

| Doc Range | Tier | Required Structure? | Purpose |
|-----------|------|---------------------|---------|
| 00 | Meta | N/A | Foundation & definitions |
| 01–05 | A | ❌ **Exempt** | Architecture & roadmap |
| 5A–20 | B | ✔ **Mandatory** | Contracts defining behavior |
| 10C | C | Flexible | Operational protocols |

---

## 5. Document Numbering Rules

### 5.1 Canonical numbering

Documents use fixed identifiers:

- **Tier A:** 01–05
- **Tier B:** 5A–20
- **Invariant subsystem:** 06, 06A, 06B, 06C, 06X

### 5.2 Never renumber

Once assigned, numbers MUST NOT change.

### 5.3 Use suffixes for expansions

- New invariant docs → 06Y, 06Z
- New runtime docs → 10A, 10B, 10C, etc.

### 5.4 Cursor-authoring rule

Cursor MUST NOT generate a document with a reused number.

---

## 6. Cross-Reference Governance

**Canonical rules:**

✔ Always reference docs via relative paths  
✔ Always reference by ID and title  
✔ Never reference a document that does not exist  
✔ Never reference deprecated documents  

**Automatic validation:**

- A link to a missing file MUST fail CI
- A link to an outdated filename MUST fail CI
- Circular document references MUST fail CI

**CI Enforcement:**

The **ArcSight Documentation Consistency Monitor** MUST run automatically after every push via GitHub Actions (`.github/workflows/docs-consistency-monitor.yml`).

The monitor validates:
- Cross-reference integrity (no broken links)
- Structure compliance (Tier B documents have required sections)
- File existence (all required documents present)
- Link validation (all internal links point to existing files)

This ensures permanent link integrity and structural compliance.

---

## 7. Authoring Protocol for Cursor

Cursor MUST adhere to the following:

### 7.1 Cursor may generate contracts only when explicitly asked

Cursor must not autonomously create:

- new invariants
- new rulepacks
- new envelope fields
- new schema versions
- new contracts

### 7.2 Cursor must enforce determinism rules in all generated examples

All examples MUST avoid:

- time
- randomness
- OS-dependent pathing
- non-canonical JSON ordering

### 7.3 Cursor must maintain section order exactly

Reordering sections is forbidden.

### 7.4 Cursor must update cross-references when rewriting a document

All references MUST remain correct.

### 7.5 Cursor must refuse to generate non-deterministic content that violates [Determinism Contract](./07-determinism-contract.md)

**Examples:**

- "generate a random ID" → refused
- "use new Date()" → refused

---

## 8. Document Lifecycle Rules

**States:**

- **Draft** — allowed to change freely
- **Active** — locked; changes require version bump
- **Deprecated** — replaced by new doc; still referenced by adapters
- **Archived** — no longer referenced

A document's state MUST be listed in its Change Log.

**Modification constraints:**

- Active documents may only change in additive ways
- Breaking changes require:
  - `analyzerVersion` bump
  - `schemaVersion` bump
  - invariant suite update
  - golden test update

---

## 9. Full Documentation Index

### Architectural Docs (Tier A — Exempt)

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md) (01)
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) (02)
- [Full System Roadmap](./03-full-system-roadmap.md) (03)
- [Usage Model](./04-usage-model.md) (04)
- [Monorepo Migration Guide](./5B-monorepo-migration.md) (05B)

### Technical Contracts (Tier B — Must follow structure)

- [Dependency Contract](./5A-dependency-contract.md) (5A)
- [Invariants Contract](./06-invariants-contract.md) (06)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md) (06A)
- [Error Codes Catalog](./06B-error-codes-catalog.md) (06B)
- [Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md) (06C)
- [Invariant System Index](./06X-invariant-system-index.md) (06X)
- [Determinism Contract](./07-determinism-contract.md) (07)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md) (08)
- [Golden Test Governance](./09-golden-test-governance.md) (09)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md) (10)
- [Schema Evolution Contract](./11-schema-evolution-contract.md) (11)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md) (12)
- [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md) (13)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) (14)
- [Envelope Format Spec](./15-envelope-format-spec.md) (15)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md) (16)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md) (17)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md) (18)
- [Drift Detection Contract](./19-drift-detection-contract.md) (19)
- [Deterministic Config Contract](./20-deterministic-config-contract.md) (20)

### Operational Docs (Tier C)

- [README](./README.md)
- Cursor Development Protocol (10C) ← to be generated

**All future documents must be added to this index.**

---

## 10. Change Log (Append-Only)

**v1.3.0**

- Added 06A/06B/06C/06X to index
- Added Tier A exemption rule for Docs 01–05
- Updated all cross-references
- Added formal Cursor authoring protocol
- Split Dependency/Monorepo docs into 5A/5B
- Cleaned dead references (invariants-contract, cursor-protocol)

**v1.2.0**

- Added contract structure enforcement

**v1.1.0**

- Initial index + rules

---

## 11. Glossary

Essential terms used throughout ArcSight documentation:

**Envelope**  
The deterministic output structure produced by the ArcSight engine. Contains `version`, `identity`, `core`, `extensions`, and `meta` sections. Must be byte-for-byte identical for identical inputs.

**RepoSnapshot**  
The canonical, normalized representation of a repository used as input to the engine. Must be stripped of all filesystem metadata, timestamps, and OS-specific information.

**SchemaVersion**  
A structural version number (integer) that increments only when the Envelope structure changes. Governs envelope shape, not semantic behavior.

**AnalyzerVersion**  
A semantic version that increments when analyzer behavior or logic changes. Governs what the analyzer does, not the structure of its output.

**RulepackVersion**  
A version number for individual rulepack modules. Governs rulepack-specific analysis logic and outputs.

**Determinism**  
The guarantee that identical inputs (RepoSnapshot + config + versions) produce identical outputs (Envelope) byte-for-byte, across all machines, OSes, and time.

**Extensions.***  
The growth zone within the Envelope where optional fields may be added without schema version bumps. Preserves backward compatibility while allowing feature expansion.

**Adapter Chain**  
A series of pure, deterministic functions that upgrade historical envelope schemas to the current canonical schema. Ensures all envelopes normalize to a single shape for consumers.
