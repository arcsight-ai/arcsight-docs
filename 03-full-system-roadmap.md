# ArcSight Full System Roadmap

## FINAL CTO SIGN-OFF

### Phase 1 → Phase 2 Complete Execution Blueprint

This document defines the entire correct sequence for building ArcSight:

- Architecture Blueprint
- Determinism Requirements
- Canonicalization
- Snapshot → Envelope Pipeline
- Graph Construction
- Cycle Detection
- Rulepacks
- Invariants
- Degraded Mode
- Schema Evolution
- Runtime Orchestration
- Golden Tests
- DLQ + Retries
- Phase 2 Migration Plan
- Shadow Analyzer Architecture
- Enterprise Roadmap

This is the master execution document for ArcSight.  
Every subsystem, every file, every invariant derives from this.

---

# 🟦 SECTION 1 — HIGH-LEVEL DECISION (Phase Structure)

### ✅ Phase 1 = Isolated Packages (Same Repository, No Shared Types)

**Packages:**

- `arcsight-wedge` (deterministic engine)
- `arcsight-cli` (developer tooling + golden tests)
- `arcsight-github-app` (runtime orchestrator)

**Rules:**

- no shared utilities
- no shared types
- no workspaces
- strict import boundaries

### ✅ Phase 2 = Monorepo Migration

**Adds:**

- shared types
- analyzer version matrices
- rulepack modularization
- sentinel
- dashboard
- shadow analyzer infrastructure

This two-phase structure is locked.

---

# 🟦 SECTION 2 — WHY SEPARATE PACKAGES (Phase 1)

Phase 1 must produce:

- a deterministic, pure analyzer
- a minimal runtime
- a golden-test-verified CLI

Separate packages guarantee:

- zero cross-dependencies
- zero type drift
- no accidental impurity
- deterministic golden tests
- easy migration to monorepo

This is the only correct architecture for Phase 1.

---

# 🟩 SECTION 3 — WHY MONOREPO (Phase 2)

Phase 2 introduces features that cannot be built safely without a monorepo:

🔥 Shared types/schemas  
🔥 Multi-engine shadow builds  
🔥 Stable rulepack versioning  
🔥 Schema adapters  
🔥 Shared golden fixtures  
🔥 Drift detection  
🔥 Sentinel  
🔥 Dashboard  
🔥 Enterprise rulepacks  

**Additional constraints:**

1. **Multi-engine builds require workspace tooling**

   Shadow analyzers must import shared types from a single canonical location.

2. **Drift detection requires a stable baseline**

   Migration cannot occur until zero drift across all golden repos.

3. **Schema evolution must be centralized**

   Adapters and rules must be co-located.

4. **Sentinel must load multiple engine versions**

   Impossible with separate package folders.

Thus, Phase 2 requires a monorepo.

---

# 🟧 SECTION 4 — PHASE 1 PACKAGE STRUCTURE

**Authoritative structure (from [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)):**

```
/arcsight-wedge      ← deterministic engine
/arcsight-cli        ← developer tooling + golden tests
/arcsight-github-app ← runtime orchestrator
```

**Rules:**

- no shared types
- no shared utils
- only CLI/runtime may import engine's public API
- snapshot builder temporarily lives in engine (must migrate to runtime in Phase 2)

Engine remains pure.  
Runtime remains a host.  
CLI remains a consumer.

---

# 🟣 SECTION 5 — PHASE 1 IMPLEMENTATION ORDER

**This order is binding. Changing it introduces nondeterminism or invalid baselines.**

## STEP 1 — Build the Deterministic Engine (arcsight-wedge)

The engine MUST reach:

✔ determinism  
✔ purity  
✔ zero nondeterministic behavior  
✔ zero drift across golden repos  

before Phase 2.

**Build subsystems in this order:**

### 1️⃣ Canonicalization Subsystem

Implement:

- path normalization
- newline normalization
- deterministic file ordering
- canonical hashing
- `canonicalRepoSnapshot`

Canonicalization must be:

- pure
- stable
- locale-invariant
- encoding-invariant
- environment-invariant

Locale-dependent logic (e.g., Turkish lowercase) is forbidden.

### 2️⃣ RepoSnapshot Format + Temporary Snapshot Builder

Snapshot builder lives temporarily inside the engine.

Snapshot must:

- be a pure structural view
- depend only on archive contents
- never depend on PR metadata, commit messages, timestamps, or runtime context
- be deterministic across machines

**Phase-2 rule**

Snapshot builder must migrate out of the engine into `arc-runtime`.

### 3️⃣ Graph Builder

Implement:

- `buildGraph`
- `graphStats`

Graph must be deterministic, stable, and free of internal heuristics.

### 4️⃣ Cycle Detection (Builtin Core Rulepack)

Implement:

- `detectCycles`
- `cycleClassifier`

Classification must be deterministic, pure, and ordering-invariant.

### 🔥 Phase-1 Rulepack Clarification (New)

**Phase 1 includes ONLY built-in core rulepacks.**  
External or enterprise rulepacks are forbidden until Phase 2.

This enforces purity, reproducibility, and stable `analyzerVersion` behavior.

### 5️⃣ Invariants

Implement:

- `validateGraph`
- `validateEnvelope`

If invariants fail → deterministic error envelope.

### 6️⃣ Envelope Builder

Implement:

- `buildCoreEnvelope`
- `attachExtensions`
- `signEnvelope`

Envelope must be identical for identical RepoSnapshots.

### 7️⃣ Degraded Mode

Two deterministic degradation classes:

- `degradeForLimits`
- `degradeForComplexity`

Limit enforcement must be deterministic and prefix-based.

### 8️⃣ Schema Adapters

Implement:

- v1 → v2 forward adapter

Adapters must:

- maintain backwards compatibility
- never drop fields
- preserve `identityChain`
- be reversible for testing

### 9️⃣ Fingerprinting

Implement `repoFingerprint` using:

- canonical ordering
- stable hashing
- pure structural signature

Fingerprints must be identical across machines, OSes, and locales.

### 🔟 Golden Tests (Engine)

Golden repos:

- small
- medium
- weird

Outputs must be:

- byte-for-byte stable
- stable across machines
- stable across environments

### 🔥 New Refinement: Cycle-Complete Before Golden Tests

**Before generating golden tests:**

Engine MUST reach cycle-complete deterministic behavior.  
Golden baselines MUST NOT be captured while cycle logic is still changing.

This aligns with [Golden Test Governance](./09-golden-test-governance.md) and prevents locking in incorrect baselines.

**Required for Phase-2 migration:**

✔ zero drift across all golden repos  
✔ `analyzerVersion` frozen

---

## STEP 2 — Implement arcsight-cli

CLI provides:

- local analysis
- golden comparison
- fixture replay
- envelope inspection

**Commands:**

- `analyze <path>`
- `compare --golden`
- `replay <fixture>`
- `dump-envelope`

**CLI uses:**

- engine's snapshot builder (Phase 1 only)
- engine's `analyze()`
- `stableStringify`-based inspect tooling

CLI golden tests ensure full determinism.

---

## STEP 3 — Implement arcsight-github-app (Runtime)

**Runtime responsibilities:**

- Webhook server
- Event dedupe
- SHA guard
- PR-level mutex
- Token manager
- Tarball fetcher
- Tarball → RepoSnapshot
- `analyze(snapshot)`
- CheckRun publishing
- DLQ
- Rate-limit coordinator
- Telemetry + logging

**Runtime must:**

- never perform graph logic
- never run rulepacks
- never mutate envelopes
- never apply schema adapters

Runtime treats the engine as a pure black box.

---

# 🟩 SECTION 6 — PHASE 2 MONOREPO MIGRATION

**Final monorepo structure:**

```
arcsight/
  package.json
  pnpm-workspace.yaml
  turbo.json

  packages/
    arc-engine
    arc-runtime
    arc-cli
    arc-shared
    arc-sentinel
    arc-dashboard
```

**Migration requirements:**

**MUST be true before migration:**

✔ engine golden tests are stable  
✔ zero drift across golden repos  
✔ `analyzerVersion` frozen  
✔ snapshot builder ready to move into `arc-runtime`  
✔ deterministic config contract enforced  
✔ `repoFingerprint` stable  
✔ no external rulepacks yet  

Migration cost: 2–3 hours, zero rewrites.

---

# 🟥 SECTION 7 — SHADOW ANALYZER PIPELINE (Phase 2)

Shadow engines enable safe evolution:

```
arc-engine/dist/v1.0.0        ← live
arc-engine/dist/v1.1.0-shadow ← shadow
arc-engine/dist/v1.2.0-shadow ← next shadow
```

Runtime invokes:

- live engine
- all shadow engines

Sentinel compares:

- envelopes
- drift regions
- invariants
- fingerprints
- rulepack outputs

If drift is unacceptable → shadow engine not promoted.

This is the same technique used by Google Tricorder, Meta Infer, and Stripe Sorbet.

---

# 🟨 SECTION 8 — SCHEMA LIFECYCLE

Phase-1 `schemaVersion` = v1 (frozen).

**Schema evolution rules:**

- adapters required for all future versions
- never breaking
- preserve `identityChain`
- preserve core envelope structure
- rulepacks must consume upgraded schema deterministically

Schema adapters guarantee backwards compatibility indefinitely.

---

# 🟦 SECTION 9 — ENTERPRISE & PHASE 3 ROADMAP

Post-Phase-2 features:

- Dashboard
- Org-level insights
- Hotspot analysis
- Historical drift timelines
- Rulepack marketplace
- Enterprise namespaces
- Tiered limits
- Slack/Teams notifications
- Long-term archival
- Multi-repo correlation
- Boundary rules
- Forbidden imports checks

All of these rely on:

- stable envelopes
- drift-safe schema evolution
- deterministic engine behavior

---

# 🎯 FINAL SUMMARY

This roadmap defines:

- What to build
- In what order
- With what invariants
- Under what deterministic rules
- How to migrate to a monorepo
- How to introduce shadow analyzers safely
- How schema and rulepacks evolve
- How ArcSight scales to enterprise-grade static analysis

This is the authoritative execution document for ArcSight.

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)
- [Monorepo Migration](./5B-monorepo-migration.md)
- [Dependency Contract](./5A-dependency-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)
