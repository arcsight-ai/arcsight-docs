# Phase 1 Contract Authority Index

## Constitution Table of Contents

This document serves as the **constitution table of contents** for ArcSight Phase 1 contracts. It classifies all contract documents into their implementation phases and establishes the authoritative list of contracts that the Phase 1 wedge MUST implement.

**This index is the single source of truth for:**
- Which contracts are mandatory for Phase 1
- Which contracts are preserved for Phase 2+
- Which rulepack contracts are included in Phase 1

---

## 1. Purpose

This document provides a clear, authoritative classification of all ArcSight contract documents by implementation phase. It ensures:

- The Phase 1 wedge implements exactly the required contracts
- Phase 2+ contracts are preserved but not prematurely implemented
- Contract dependencies are understood and respected
- Development teams have a single reference for contract authority

---

## 2. Scope

This index classifies all Tier B contract documents (per [Documentation Index](./00-docs-index-and-authoring-protocol.md)) into:

1. **Phase 1 Required Contracts** — Contracts the wedge MUST implement
2. **Phase 2+ Contracts** — Contracts that must be preserved but are not yet implemented
3. **Rulepack v1 Contracts** — Rulepack contract files being added to Phase 1

---

## 3. Phase 1 Required Contracts

**The wedge MUST implement these contracts.** These are foundational to Phase 1 functionality and determinism guarantees.

### 3.1 Core Determinism & Engine Contracts

- **[06 — Invariants Contract](./06-invariants-contract.md)**  
  Defines the invariant system that enforces architectural guarantees. Must be implemented to ensure engine purity and correctness.

- **[06A — Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)**  
  Specifies how invariant tests must be structured and executed. Required for validating invariant compliance.

- **[07 — Determinism Contract](./07-determinism-contract.md)**  
  The foundational contract that guarantees identical inputs produce identical outputs. Absolutely required for Phase 1.

- **[09 — Golden Test Governance](./09-golden-test-governance.md)**  
  Governs how golden tests must be structured and maintained. Essential for Phase 1 validation and regression prevention.

### 3.2 Runtime & Data Format Contracts

- **[10 — Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)**  
  Defines the boundary between runtime and engine. Must be implemented to enable clean separation of concerns.

- **[14 — RepoSnapshot Contract](./14-repo-snapshot-contract.md)**  
  Specifies the canonical, normalized repository representation used as engine input. Required for determinism.

- **[15 — Envelope Format Spec](./15-envelope-format-spec.md)**  
  Defines the deterministic output structure produced by the engine. Must be implemented for Phase 1 output compliance.

### 3.3 Configuration & Drift Contracts

- **[19 — Drift Detection Contract](./19-drift-detection-contract.md)**  
  Defines how drift detection must work. Required for Phase 1 stability validation.

- **[20 — Deterministic Config Contract](./20-deterministic-config-contract.md)**  
  Specifies how configuration must be deterministic and normalized. Required for Phase 1 determinism guarantees.

### 3.4 Phase 1 Specification Contracts (NEW)

**These specifications MUST be finalized before wedge implementation begins.**

- **[21 — Contract Layer Spec v1](./21-contract-layer-spec-v1.md)**  
  Defines all data schemas (Violation Schema v1, Analyzer Interface v1, Rulepack Schema v1, Snapshot Schema v1, PR Summary JSON Schema). The single source of truth for all data moving through ArcSight.

- **[22 — Wedge Architecture Spec v1](./22-wedge-architecture-spec-v1.md)**  
  Defines the engine architecture, including pipeline (snapshot → analyzers → canonicalize → envelope), module layering, analyzer plugin system, internal type signatures, error handling, and contract enforcement.

- **[23 — Phase 1 Implementation Plan](./23-phase1-implementation-plan.md)**  
  Defines milestone-level implementation plan with build order, dependencies, test harness requirements, freeze criteria, wedge versioning strategy, and acceptance conditions.

---

## 4. Phase 2+ Contracts (Preserved, Not Yet Implemented)

**These contracts must be preserved but are NOT implemented in Phase 1.** They define capabilities that will be built in Phase 2 and beyond.

### 4.1 Rulepack & Versioning Contracts

- **[12 — Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)**  
  Defines how rulepacks are versioned and managed. Phase 2 feature; preserved for future implementation.

### 4.2 Enterprise & Advanced Features

- **[13 — Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)**  
  Defines resource limits and sandboxing policies. Phase 2+ feature for production safety and isolation.

- **[16 — Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)**  
  Defines shadow analyzer architecture and safety validation. Phase 2 feature for safe analyzer rollout.

- **[17 — Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)**  
  Specifies how analyzers are promoted from shadow to live. Phase 2 feature dependent on Sentinel.

- **[18 — Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)**  
  Defines enterprise-level extension mechanisms. Phase 2+ feature for extensibility.

---

## 5. Rulepack v1 Contract Files

**Rulepack contract information is now defined in [Contract Layer Spec v1](./21-contract-layer-spec-v1.md).**

The rulepack schema, analyzer interface, and violation schema are all specified in the Contract Layer Spec v1, which serves as the single source of truth for all contract layer data structures.

Phase 1 includes ONLY built-in core rulepacks (e.g., cycles detection). External rulepacks are forbidden until Phase 2.

---

## 6. Contract Dependency Map

### Phase 1 Dependency Hierarchy

```
Phase 1 Foundation:
├── 07-determinism-contract.md (foundational)
├── 14-repo-snapshot-contract.md (input format)
├── 15-envelope-format-spec.md (output format)
├── 10-runtime-and-engine-contract.md (boundary)
├── 20-deterministic-config-contract.md (config)
│
Phase 1 Validation:
├── 06-invariants-contract.md
├── 06A-invariant-test-suite-specification.md
├── 09-golden-test-governance.md
└── 19-drift-detection-contract.md

Phase 1 Specifications (MUST be finalized before implementation):
├── 21-contract-layer-spec-v1.md (data schemas)
├── 22-wedge-architecture-spec-v1.md (engine architecture)
└── 23-phase1-implementation-plan.md (implementation roadmap)
```

### Phase 2+ Dependencies (Not Implemented in Phase 1)

```
Phase 2 Features (blocked until Phase 1 complete):
├── 12-rulepack-versioning-contract.md (requires Phase 1 stable)
├── 16-sentinel-shadow-analysis-contract.md (requires Phase 1 stable)
├── 17-analyzer-promotion-policy.md (requires Sentinel)
├── 13-limits-and-sandbox-policy.md (requires Phase 1 runtime)
└── 18-enterprise-extension-namespaces.md (requires Phase 2 infrastructure)
```

---

## 7. Implementation Checklist

### Phase 1 Wedge MUST Implement

- [ ] [06-invariants-contract.md](./06-invariants-contract.md)
- [ ] [06A-invariant-test-suite-specification.md](./06A-invariant-test-suite-specification.md)
- [ ] [07-determinism-contract.md](./07-determinism-contract.md)
- [ ] [09-golden-test-governance.md](./09-golden-test-governance.md)
- [ ] [10-runtime-and-engine-contract.md](./10-runtime-and-engine-contract.md)
- [ ] [14-repo-snapshot-contract.md](./14-repo-snapshot-contract.md)
- [ ] [15-envelope-format-spec.md](./15-envelope-format-spec.md)
- [ ] [19-drift-detection-contract.md](./19-drift-detection-contract.md)
- [ ] [20-deterministic-config-contract.md](./20-deterministic-config-contract.md)
- [ ] [21-contract-layer-spec-v1.md](./21-contract-layer-spec-v1.md)
- [ ] [22-wedge-architecture-spec-v1.md](./22-wedge-architecture-spec-v1.md)
- [ ] [23-phase1-implementation-plan.md](./23-phase1-implementation-plan.md)

### Phase 2+ Contracts (Preserved, Not Implemented)

- [ ] [12-rulepack-versioning-contract.md](./12-rulepack-versioning-contract.md) — Preserved
- [ ] [13-limits-and-sandbox-policy.md](./13-limits-and-sandbox-policy.md) — Preserved
- [ ] [16-sentinel-shadow-analysis-contract.md](./16-sentinel-shadow-analysis-contract.md) — Preserved
- [ ] [17-analyzer-promotion-policy.md](./17-analyzer-promotion-policy.md) — Preserved
- [ ] [18-enterprise-extension-namespaces.md](./18-enterprise-extension-namespaces.md) — Preserved

---

## 8. Cross-References

- [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md) — Root documentation governance
- [Strategic Architecture](./01-decision-phase1-vs-phase2.md) — Phase 1 vs Phase 2 decision
- [Full System Roadmap](./03-full-system-roadmap.md) — Complete system execution blueprint

---

## 9. Change Log (Append-Only)

**v1.1.0** — Added Phase 1 specification contracts
- Added 21-contract-layer-spec-v1.md to Phase 1 required contracts
- Added 22-wedge-architecture-spec-v1.md to Phase 1 required contracts
- Added 23-phase1-implementation-plan.md to Phase 1 required contracts
- Updated rulepack v1 section to reference Contract Layer Spec
- Updated dependency map to include specification contracts
- Updated implementation checklist

**v1.0.0** — Initial creation
- Established Phase 1 required contracts list
- Established Phase 2+ preserved contracts list
- Added placeholder for rulepack v1 contracts
- Created contract dependency map
- Added implementation checklist

