# Monorepo Migration (FINAL, Phase 2)

## Authoritative Migration Plan for ArcSight

This document defines when, how, and under what deterministic constraints ArcSight migrates from Phase-1 isolated packages to the Phase-2 monorepo.

This is the official and binding process for:

- consolidating packages
- introducing shared types
- enabling multi-version engines
- enabling Sentinel
- preparing for enterprise rulepacks
- eliminating cross-package drift
- stabilizing schema evolution

No other document may override the policies here.

---

## 1. Purpose

The purpose of this migration:

- enable multi-engine shadow analysis
- enable shared types (envelope schema, config schema, identity-chain schema)
- unify golden tests
- enable stable rulepack versioning
- support deterministic schema evolution
- enforce single-source-of-truth for:
  - RepoSnapshot
  - Envelope schema
  - Limits + Sandbox policy
  - RulepackVersion
  - Config schema
  - Invariants

Phase-1 isolated packages cannot support these features safely.

Monorepo migration is therefore a hard requirement for Phase-2 analyzer evolution.

---

## 2. Scope

### Included

- consolidation of:
  - `arcsight-wedge` → `arc-engine`
  - `arcsight-cli` → `arc-cli`
  - `arcsight-github-app` → `arc-runtime`
- introduction of:
  - `arc-shared`
  - `arc-sentinel`
  - `arc-dashboard`
- deterministic workspace build rules
- dependency boundaries
- migration blockers
- migration rollback plan

### Excluded

- rulepack semantics ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- drift detection logic ([Drift Detection Contract](./19-drift-detection-contract.md))
- schema versioning ([Schema Evolution Contract](./11-schema-evolution-contract.md))

These are integrated but governed separately.

---

## 3. Migration Preconditions (Hard Gates)

Migration MUST NOT occur until every condition below is satisfied:

### 3.1 Engine Stability

✔ `analyzerVersion` is frozen  
✔ `schemaVersion` stable at v1  
✔ rulepack behavior frozen  
✔ no planned Phase-1 breaking changes remain  

### 3.2 Golden Baseline Stability (From [Golden Test Governance](./09-golden-test-governance.md))

✔ golden tests pass with zero drift across all golden repos  
✔ golden envelopes locked  
✔ canonicalization is 100% stable  
✔ cycle detection is complete and stable  

**New Requirement:**

Golden tests MUST NOT be generated until cycle logic is final and deterministic.

### 3.3 Snapshot Stability (From [RepoSnapshot Contract](./14-repo-snapshot-contract.md))

✔ RepoSnapshot format frozen  
✔ `canonicalRepoSnapshot` stable  
✔ snapshot builder deterministic  
✔ no pending structural changes  

### 3.4 No Shared Types in Phase-1 (From [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md))

✔ all Phase-1 packages remain isolated  
✔ no shared utilities  
✔ no shared type declarations  
✔ no circular dependencies  

### 3.5 Deterministic Config (From [Deterministic Config Contract](./20-deterministic-config-contract.md))

Config MUST:

- be static
- not depend on PR metadata
- not contain locale-sensitive operations
- not depend on runtime heuristics
- be fully resolved before analysis
- generate identical normalized hashes

### 3.6 Runtime Stability

✔ GitHub App runs full E2E pipeline  
✔ DLQ functioning  
✔ rate limits handled deterministically  
✔ degraded mode logs stable  

### 3.7 Documentation Baseline

Before migration:

- Docs #01 → #20 fully updated
- Definitive `analyzerVersion` + `schemaVersion` documented
- All invariants locked for Phase-1

No migration before this.

---

## 4. Monorepo Target Structure

Upon migration, the structure becomes:

```
arcsight/
  package.json
  pnpm-workspace.yaml
  turbo.json

  packages/
    arc-engine       ← deterministic analyzer
    arc-runtime      ← GitHub App
    arc-cli          ← local tooling + golden tests
    arc-shared       ← types, schema, invariants, limits, adapters
    arc-sentinel     ← drift detection, shadow-mode orchestration
    arc-dashboard    ← SaaS UI (Phase 3)
```

### 4.1 New Internal Dependency Graph

**Allowed:**

```
        arc-engine
       /          \
 arc-cli         arc-sentinel
   |                |
 arc-runtime → arc-shared
                ↓
           arc-dashboard
```

**Forbidden (from [Dependency Contract](./5A-dependency-contract.md)):**

- `arc-engine` → `arc-runtime`
- `arc-engine` → `arc-cli`
- `arc-engine` → `arc-dashboard`
- `arc-sentinel` → `arc-runtime`
- `arc-cli` → `arc-runtime`
- `arc-dashboard` → `arc-engine`

Arc-shared becomes the only leaf package.

---

## 5. Migration Process (Precise Steps)

Migration MUST occur exactly in this sequence.

### Step 1 — Create Monorepo Shell

Create:

- root `package.json`
- `pnpm-workspace.yaml`
- `turbo.json`

Add:

- no prebuild scripts
- no postinstall scripts
- no implicit cross-links

Workspace must be deterministically empty.

### Step 2 — Move Packages into /packages

Move:

- `arcsight-wedge` → `/packages/arc-engine`
- `arcsight-cli` → `/packages/arc-cli`
- `arcsight-github-app` → `/packages/arc-runtime`

**During migration:**

- Zero behavior changes allowed
- Zero version bumps allowed
- Golden tests MUST remain bit-for-bit identical

If any diff occurs → migration is invalid.

### Step 3 — Introduce arc-shared

Move into `arc-shared`:

- stable types
- invariants
- schema definitions
- rulepack interfaces
- limits & sandbox policy
- config schema
- identity chain schema
- RepoFingerprint
- envelope normalization rules

**Rules:**

- `arc-shared` MUST NOT import any other workspace package.
- `arc-engine` MAY import `arc-shared`.
- `arc-runtime` MAY import `arc-shared`.
- `arc-cli` MAY import `arc-shared`.

This unifies schema and invariants.

### Step 4 — Migrate Snapshot Builder Out of Engine

Snapshot builder must move:

- from → `arc-engine`
- to → `arc-runtime` + `arc-cli`

**Reason:**

- Phase-2 requires building multiple engine versions from identical snapshots
- snapshot MUST be runtime-owned ([Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- Engine receives a pure RepoSnapshot, never constructs one.

### Step 5 — Enable Multi-Version Engine Builds

Using `arc-engine/dist/vX.Y.Z`:

- `v1.0.0` (live)
- `v1.1.0` (shadow)
- `v1.2.0` (next shadow)

Runtime loads each version from:

```
packages/arc-engine/dist/vX.Y.Z/analyze.js
```

This is the fundamental enabling step for:

- shadow analysis
- drift detection
- analyzer promotion

### Step 6 — Introduce arc-sentinel

Sentinel now:

- receives envelopes from live and shadow engines
- computes drift
- enforces analyzer promotion rules ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- classifies drift: benign / warning / blocker
- ensures zero-schema-violation guarantee

Sentinel requires monorepo because:

- it must import multiple engine versions
- must share schema types
- must depend on stable invariants

### Step 7 — Validate Workspace Determinism

Validate:

- identical outputs before and after migration
- golden tests unchanged
- live analyzer envelopes identical
- RepoSnapshot unchanged
- checksum of full E2E pipeline unchanged

If any nondeterminism appears → rollback.

---

## 6. Migration Blockers

Migration MUST NOT proceed if any of the following occurs:

### Structural

- snapshot builder unstable
- canonicalization unstable
- `analyzerVersion` not frozen
- `schemaVersion` in transition
- drift across golden repos

### Determinism ([Determinism Contract](./07-determinism-contract.md))

- locale-dependent operations exist
- timestamps, randomness, or system data leak
- config hash mismatch
- sandbox policy mismatch
- noncanonical orderings

### Semantic

- rulepacks still changing
- cycle detection still evolving
- invariants not fully encoded in `arc-shared`

### Operational

- runtime not stable in end-to-end CI
- DLQ unreliable
- rate-limit behavior nondeterministic

Any blocker → migration halts.

---

## 7. Rollback Strategy (Must Be Lossless)

Rollback MUST be:

- deterministic
- complete
- restorable in <30 minutes
- leave no partial state

**Rollback Plan:**

1. Delete monorepo shell
2. Move packages back to Phase-1 layout
3. Restore `package.json` files exactly
4. Re-run golden tests
5. Confirm golden stability
6. Re-freeze `analyzerVersion`

If golden tests diverge → rollback is invalid and must restart.

---

## 8. Post-Migration Requirements

After monorepo migration:

### 8.1 All future `analyzerVersion` bumps must occur in monorepo

Phase-1 structure is permanently frozen.

### 8.2 Shared types must live in `arc-shared`

No exceptions.

### 8.3 Rulepacks become fully modular

Phase-2 loads rulepacks from `/packages/arc-engine/rulepacks/*`.

### 8.4 Sentinel monitors all analyzer promotion

Automatic promotion forbidden ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)).

### 8.5 All drift comparisons use `arc-shared` schemas

Unknown extension fields preserved.

---

## 9. Future Extensions Enabled by Monorepo

- SaaS dashboard (Phase 3)
- org-wide insights
- boundary rulepacks
- enterprise rulepacks
- policy enforcement
- multi-analyzer test grids
- experimental rulepack sandboxes
- long-term envelope archiving
- advanced drift analytics

These require:

- workspace linking
- unified types
- multi-engine runtime
- schema evolution framework

Phase-1 layout cannot support any of these.

---

## 10. Summary

Migration occurs exactly once.  
Migration requires zero drift.  
Migration introduces monorepo with deterministic builds.  
Migration unlocks shadow analysis + sentinel + enterprise features.

This document governs the entire process.

---

## 11. References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Phase-1 Physical Architecture](./02-phase1-folder-scaffold.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Dependency Contract](./5A-dependency-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Sentinel Shadow Analysis](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)
