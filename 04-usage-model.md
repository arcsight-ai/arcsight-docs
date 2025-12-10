# ArcSight Documentation Usage Model

## How to Navigate, Interpret, and Apply the Architecture Documents

**FINAL VERSION — Phase 1 & Phase 2 Compliant**

---

## 1. Purpose

This document defines how ArcSight's documentation is structured, which document is authoritative in each context, and how engineers must use the documentation across Phases 1 and 2.

This is the "meta-layer" that prevents:

- ambiguity
- conflicting interpretations
- accidental reference to outdated documents
- architectural drift
- incorrect implementation order
- incorrect dependency assumptions

It explains how Documents 01 → 20 relate to each other and when each one governs decisions.

---

## 2. Scope

This document describes:

✔ The four-layer architecture documentation stack  
✔ Which document to follow during development  
✔ How to interpret contracts vs policies vs roadmaps  
✔ How documents evolve across Phase 1 → Phase 2  
✔ Enforcement rules for dependencies, determinism, schema, and config  

**Not included:**

✘ API design details  
✘ Schema evolution specifics ([Schema Evolution Contract](./11-schema-evolution-contract.md))  
✘ Drift detection internals ([Drift Detection Contract](./19-drift-detection-contract.md))  
✘ Analyzer promotion rules ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))  

---

## 3. Definitions & Terms

**Strategic Layer**  
Long-term, slow-changing architectural decisions (Docs 01, 05).

**Physical Layer**  
Directory structure & dependency boundaries (Doc 02).

**Operational Layer**  
Executable system plan and build-order (Doc 03).

**Contract Layer**  
Formal behavioral rules governing determinism, drift, schema, runtime, config, limits, promotion, and extensions (Docs 06–20).

**Authoritative Document**  
The document that overrides all others within its domain.

---

## 4. The Four-Layer Documentation Model

ArcSight documentation is structured as:

```
┌────────────────────────────────────────────────────────┐
│  Layer 4 — Meta Layer (Doc 04)                         │
│  How to use the documentation                          │
└────────────────────────────────────────────────────────┘
                ▲
┌────────────────────────────────────────────────────────┐
│  Layer 1 — Strategic Architecture (Docs 01 & 05)       │
│  Long-term, high-level decisions                       │
└────────────────────────────────────────────────────────┘
                ▲
┌────────────────────────────────────────────────────────┐
│  Layer 2 — Physical Architecture (Doc 02)               │
│  Folder layout, package boundaries, allowed imports    │
└────────────────────────────────────────────────────────┘
                ▲
┌────────────────────────────────────────────────────────┐
│  Layer 3 — Operational Architecture (Doc 03)            │
│  What to build, in what order, with which invariants   │
└────────────────────────────────────────────────────────┘
                ▲
┌────────────────────────────────────────────────────────┐
│  Layer 5 — Contract Layer (Docs 06–20)                 │
│  Deterministic rules governing engine, runtime, etc.    │
└────────────────────────────────────────────────────────┘
```

Each layer narrows the scope and increases precision.

They never conflict because each layer governs a different dimension.

---

## 5. Authority Hierarchy (Which Doc Wins?)

This is the most important section.

If two documents appear to overlap, the following hierarchy resolves it:

### 1️⃣ Contract Layer (Docs 06–20)

**Always wins.**

Contracts define deterministic, enforceable rules.

**Examples:**

- drift classification is defined in [Drift Detection Contract](./19-drift-detection-contract.md)
- schema rules in [Schema Evolution Contract](./11-schema-evolution-contract.md)
- promotion rules in [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- config rules in [Deterministic Config Contract](./20-deterministic-config-contract.md)
- extension namespacing in [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)
- limits in [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)

Contracts override everything.

### 2️⃣ Operational Roadmap (Doc 03)

Defines what to build, in what order, and how the system operates.

If a contract does not specify ordering or build sequence, [Full System Roadmap](./03-full-system-roadmap.md) does.

### 3️⃣ Physical Architecture (Doc 02)

Defines folder structure and boundaries.

If a contract describes behavior but not where it belongs, [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) wins.

### 4️⃣ Strategic Architecture (Docs 01 & 05)

Governs:

- Phase-1 vs Phase-2 structure
- Monorepo migration window
- High-level package rules

If a lower-level doc tries to contradict a strategic decision (rare), strategic wins.

### 5️⃣ This Document (Doc 04)

Never overrides anything.

Its job is to explain how to interpret the other documents.

---

## 6. Correct Usage During Phase 1 (Now)

During Phase 1, developers MUST:

✔ Follow [Full System Roadmap](./03-full-system-roadmap.md) (Operational Roadmap) for implementation order  
✔ Follow [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) (Physical Architecture) for file placement + boundaries  
✔ Follow Docs 06–20 for behavior and determinism  
⚠️ NOT use monorepo rules yet  

Do not read Docs 01 or 05 when writing code (unless planning migration).

**Phase-1 Authority Model:**

```
Contracts → Doc 03 → Doc 02 → Doc 01 → Doc 04
```

---

## 7. Correct Usage During Phase 2

Once the monorepo exists and sentinel is active, the authority model changes:

```
Contracts → Sentinel & Promotion (16,17) → Doc 03 → Doc 02 → Doc 01 → Doc 04
```

**New dominant documents:**

- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md) (Doc 16)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md) (Doc 17)

These now govern analyzer lifecycle decisions.

---

## 8. How to Read the Documentation (Per Role)

### Engine Developers

Read in this order:

1. [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) — folder boundaries
2. [Full System Roadmap](./03-full-system-roadmap.md) — build order
3. [Invariants Contract](./06-invariants-contract.md), [Determinism Contract](./07-determinism-contract.md), [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md), [Schema Evolution Contract](./11-schema-evolution-contract.md), [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md), [Envelope Format Spec](./15-envelope-format-spec.md) — engine core behavior
4. [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md) (Phase 2) — promotion rules
5. [Drift Detection Contract](./19-drift-detection-contract.md) — drift detection

### Runtime Developers

Read:

- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

### Sentinel Developers

Read:

- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)

### Enterprise Rulepack Authors

Read:

- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)

---

## 9. How Documents Evolve Over Time

### Phase 1:

- Docs 01–05 are mostly static
- Docs 06–15 evolve as engine stabilizes
- Docs 16–20 remain unused but frozen for Phase 2

### Phase 2:

- Docs 01 and 02 are locked
- Docs 03, 11, 12, 15, 16, 17, 18, 19, 20 evolve
- New contracts rarely added

---

## 10. Change Log (Append-Only)

**v1.0.0** — Full rewrite for Phase-1 + Phase-2 architecture. Defines authority hierarchy, role-based reading model, documentation layers, and evolution rules.

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Monorepo Migration](./5B-monorepo-migration.md)
- [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md)
