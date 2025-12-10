# Cross-Package Dependency Contract

## Deterministic Architecture Rules for Engine, Runtime, CLI, Sentinel, Dashboard & Rulepacks

**FINAL VERSION — Phase 1 & Phase 2**

---

## 1. Purpose

This document defines ArcSight's formal, enforceable dependency rules across all packages and systems.

It ensures:

- determinism
- purity of the engine
- clean architectural boundaries
- safety during monorepo migration
- rulepack isolation
- multi-analyzer compatibility
- future enterprise extensibility

This contract MUST hold in both Phase 1 and Phase 2.

Violating this document breaks:

- determinism
- drift detection
- analyzer promotion
- schema evolution
- runtime correctness
- snapshot consistency

---

## 2. Scope

This contract governs:

✔ Allowed / forbidden imports  
✔ Cross-package access rules  
✔ Engine purity requirements  
✔ CLI & runtime boundaries  
✔ Rulepack isolation  
✔ Monorepo dependency graph (Phase 2)  
✔ File system, environment, and network access rules  
✔ Deterministic configuration loading  

**Not included:**

✘ Schema evolution details ([Schema Evolution Contract](./11-schema-evolution-contract.md))  
✘ Drift detection rules ([Drift Detection Contract](./19-drift-detection-contract.md))  
✘ Analyzer promotion lifecycle ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))  

---

## 3. Definitions

**Engine**  
Pure deterministic analyzer (`arcsight-wedge` / `arc-engine`).

**Runtime**  
GitHub App orchestrator (`arcsight-github-app` / `arc-runtime`).

**CLI**  
Local tooling & golden test runner (`arcsight-cli` / `arc-cli`).

**Rulepack**  
Namespace-specific extensions running inside the engine.

**Sentinel**  
Shadow-mode analyzer comparator (`arc-sentinel`).

**Dashboard**  
SaaS analytics layer (`arc-dashboard`).

---

## 4. Phase 1 Dependency Rules (Separate Package Folders)

Phase-1 consists of three isolated packages:

```
arcsight-wedge      ← deterministic engine
arcsight-github-app ← runtime orchestrator
arcsight-cli        ← local developer tooling
```

These MUST remain fully independent.

### 4.1 arcsight-wedge (Engine) Rules

**Engine MAY import:**

- its own internal modules
- Node.js standard library (Filesystem only inside tests)
- pure deterministic utilities

**Engine MUST NOT import:**

- runtime
- CLI
- GitHub APIs
- network libraries
- timers, randomness, or environment variables
- filesystem (except inside test files only)
- any PR metadata, commit metadata, branch names
- any config that depends on request context

**Engine MUST be:**

- pure
- deterministic
- stateless
- side-effect free

RepoSnapshot is the only input.  
Envelope is the only output.

### 4.2 arcsight-github-app (Runtime) Rules

**Runtime MAY import:**

- engine public API (`analyze`)
- Node.js fs/network
- GitHub API clients
- Redis/queue libs (optional)

**Runtime MUST NOT import:**

- engine internals
- CLI
- golden tests
- rulepacks
- schema adapters
- cycle detection internals

**Runtime MUST NOT:**

- mutate envelopes
- inject nondeterministic values (except timestamps into `generated_at`)
- retry analysis with altered settings

### 4.3 arcsight-cli (CLI) Rules

**CLI MAY import:**

- engine public API
- filesystem
- test fixtures
- stable deterministic libs

**CLI MUST NOT import:**

- runtime
- GitHub API
- queue systems
- rulepacks directly

CLI is purely a developer tool, not part of production runtime.

### 4.4 Forbidden Phase-1 Edges (Absolute)

```
runtime → cli        ❌ forbidden
engine → runtime     ❌ forbidden
engine → cli         ❌ forbidden
cli → runtime        ❌ forbidden
```

**No cross-package imports except:**

- `runtime → engine` (allowed)
- `cli → engine` (allowed)

These rules prevent:

- dependency cycles
- nondeterminism
- config drift
- accidental API leaks
- early monorepo entanglement

---

## 5. Phase 2 Dependency Rules (Monorepo)

Phase-2 packages:

```
packages/
  arc-engine
  arc-runtime
  arc-cli
  arc-shared
  arc-sentinel
  arc-dashboard
```

The dependency graph becomes:

```
          arc-shared
         /     |     \
        /      |      \
  arc-engine  arc-cli  arc-runtime
       |         |           \
       |         |            \
   arc-sentinel  |         arc-dashboard
```

### 5.1 arc-engine Rules (Most Restrictive)

**Engine MAY depend on:**

- `arc-shared` ONLY
- deterministic pure libraries

**Engine MUST NOT depend on:**

- `arc-runtime`
- `arc-cli`
- `arc-dashboard`
- `arc-sentinel`
- external rulepacks (enterprise packs load via contract only)

**Engine MUST run:**

- without network
- without filesystem
- without environment variables
- without timestamps
- without nondeterministic data structures

**Engine MUST NOT:**

- modify RepoSnapshot
- modify configuration at runtime
- derive defaults based on repo content
- load multiple configs based on request context

Engine is pure-zero.

### 5.2 arc-cli Rules

**CLI MAY depend on:**

- `arc-engine`
- `arc-shared`
- filesystem
- local fixtures

**CLI MUST NOT depend on:**

- `arc-runtime`
- `arc-dashboard`
- `arc-sentinel`

**CLI MUST NOT:**

- modify engine state
- access network except GitHub download fixtures for tests

### 5.3 arc-runtime Rules

**Runtime MAY depend on:**

- `arc-engine`
- `arc-shared`
- network libraries
- filesystem tarball extraction
- GitHub APIs

**Runtime MUST NOT depend on:**

- `arc-cli`
- `arc-sentinel`
- `arc-dashboard`
- rulepacks directly

**Runtime MUST NOT:**

- retry analysis with altered limits
- modify envelopes
- mutate snapshots
- produce nondeterministic config

### 5.4 arc-sentinel Rules

**Sentinel MAY depend on:**

- `arc-engine` (multiple versions)
- `arc-shared`

**Sentinel MUST NOT:**

- import `arc-runtime`
- import `arc-cli`
- modify envelopes
- modify snapshots
- alter analyzer behavior

Sentinel observes, compares, and classifies ONLY.

### 5.5 arc-dashboard Rules

**Dashboard MAY depend on:**

- `arc-shared`
- telemetry stored by runtime/sentinel

**Dashboard MUST NOT depend on:**

- `arc-engine` directly
- `arc-runtime`
- `arc-cli`
- internal analyzer logic

Dashboard is an observer, not an analyzer.

### 5.6 arc-shared Rules

`arc-shared` is the root package.

**`arc-shared` MUST NOT import anything.**

All other packages may import `arc-shared`.

It contains:

- stable types
- schema definitions
- utility functions permitted for all layers

`arc-shared` must remain extremely stable.

---

## 6. Rulepacks Dependency Rules

Rulepacks MUST:

- run inside engine sandbox
- import only allowed deterministic utilities
- NEVER access network
- NEVER use filesystem
- NEVER depend on runtime or PR context
- NEVER mutate or delete historical extension fields
- NEVER introduce nondeterministic data structures

Rulepacks MUST operate ONLY on:

- canonical RepoSnapshot
- canonical Graph
- canonical Envelope extension namespace

---

## 7. Deterministic Dependency Rules (Global)

Across all packages:

- No environment-variable-dependent behavior
- No locale-dependent behavior
- No randomness
- No `Date.now` inside engine
- No cross-layer retries with different parameters
- No dynamic config merging
- No partially applied configs
- Config MUST be identical for live vs shadow analyzers
- Defaults MUST be static and global

Non-deterministic dependencies break:

- drift detection
- analyzer promotion
- golden tests
- shadow-mode behavior

---

## 8. Enforcement Requirements

These rules MUST be enforced via:

- TypeScript project references
- ESLint import boundaries
- CI dependency graph validation
- No-implicit-dependencies rules
- Build-time enforceable barriers

Violations MUST fail CI deterministically.

---

## 9. Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [System Roadmap](./03-full-system-roadmap.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 10. Change Log (Append-Only)

**v1.0.0** — Full rewrite. Defines Phase-1 and Phase-2 dependency graphs, engine purity rules, sentinel boundaries, dashboard isolation, rulepack restrictions, and deterministic dependency guarantees.
