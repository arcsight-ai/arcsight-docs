# Phase 1 Folder Scaffold (FINAL UPDATED VERSION)

## 1. Purpose

This document defines the physical file structure and package boundaries for ArcSight Phase 1.

It establishes:

- where code lives
- what each package is responsible for
- allowed and forbidden imports
- deterministic boundaries
- snapshot builder location rules (Phase 1 → Phase 2 migration requirement)
- the canonical directory tree for each package

This is the day-to-day developer reference for implementing ArcSight Phase 1.

Phase 2 will introduce a monorepo (see [Strategic Architecture](./01-decision-phase1-vs-phase2.md)), but Phase 1 intentionally uses isolated packages within a single repository.

---

## 2. Phase 1 Architecture Overview

Phase 1 contains three isolated packages, living side-by-side in the same repository:

```
/arcsight-wedge      ← deterministic engine
/arcsight-github-app ← runtime orchestrator (GitHub App)
/arcsight-cli        ← developer CLI + golden tests
```

These packages:

- MUST NOT import each other except via the engine's public API surface
- MUST NOT share types, utilities, or modules
- MUST NOT introduce any cross-package coupling

They operate as logically independent packages even though they live in the same repo.

This structure enables:

- determinism
- clarity
- fast iteration
- safe monorepo migration later
- simple drift regression baselines

No monorepo tooling exists in Phase 1 (no workspaces, no shared libs).

---

## 3. Global Phase 1 Boundaries & Rules

### 3.1 No Shared Types (Phase 1 Hard Rule)

To prevent accidental coupling:

- No shared type packages MAY exist in Phase 1.
- No `@arcsight/shared`
- No shared utility folder
- No cross-package type imports

Each package MUST define its own internal types except where types are part of the engine's public API (RepoSnapshot, Envelope).

This rule enables a clean and risk-free monorepo migration in Phase 2.

### 3.2 Snapshot Builder Phase-1 Exception (Temporary)

The RepoSnapshot builder lives in `arcsight-wedge` during Phase 1, but:

- MUST remain pure
- MUST NOT import any graph, rulepack, or analysis logic
- MUST NOT inspect environment variables
- MUST NOT differ between CLI and runtime execution

**In Phase 2, the snapshot builder MUST be moved to `arc-runtime`.**  
(Required for shadow-mode analyzers and drift detection validity.)

This is a mandatory migration step before enabling multi-version engines.

### 3.3 Engine Purity

The engine:

- MUST be deterministic
- MUST contain no filesystem reads (except in tests)
- MUST contain no timestamps, randomness, environment access
- MUST not use GitHub APIs
- MUST only accept a RepoSnapshot as input
- MUST output a deterministic Envelope

Engine purity is foundational to:

- golden tests
- drift detection
- shadow analyzers
- canary rollout
- reproducibility

### 3.4 Runtime Must Not Contain Analysis Logic

The runtime (GitHub App):

- owns webhook handling
- owns tarball fetching
- owns RepoSnapshot construction (Phase 2)
- owns CheckRun publishing
- MUST NOT perform graph building, cycle detection, or rulepack execution

Runtime is a host, not a co-analyzer.

### 3.5 CLI Is a Consumer, Not a Host

CLI:

- drives the engine locally
- runs golden tests
- produces snapshots for local repos
- MUST NOT contain GitHub runtime logic

---

## 4. Package Layouts (Authoritative Canonical Structure)

Below are the exact, canonical structures to follow.

### 📁 arcsight-wedge

**THE ENGINE — deterministic, pure, isolated**

This package contains all core analysis logic.

The engine MUST remain:

- pure
- deterministic
- GitHub-agnostic
- network-free
- environment-free
- filesystem-free during `analyze()`

```
arcsight-wedge/
  package.json
  tsconfig.json
  README.md

  src/
    index.ts                         # analyze(snapshot): Envelope

    types/
      RepoSnapshot.ts                # ONLY input type
      Envelope.ts
      IdentityChain.ts
      GraphTypes.ts
      ErrorCodes.ts
      SandboxPolicy.ts
      Limits.ts

    canonicalization/
      normalizePaths.ts
      normalizeNewlines.ts
      orderFiles.ts
      canonicalHash.ts
      canonicalRepoSnapshot.ts

    snapshot/
      buildSnapshot.ts               # Phase 1 only (must migrate in Phase 2)

    graph/
      buildGraph.ts
      detectCycles.ts
      graphStats.ts

    rulepacks/
      cycles/
        index.ts
        cycleClassifier.ts

    invariants/
      validateEnvelope.ts
      validateGraph.ts

    envelopes/
      buildCoreEnvelope.ts
      attachExtensions.ts
      signEnvelope.ts

    versions/
      analyzerVersion.ts
      schemaVersion.ts
      rulepackVersions.ts

    adapters/
      schemaAdapters.ts              # v1 → v2 migrations

    sandbox/
      sandboxPolicy.ts
      enforceSandboxLimits.ts

    degraded/
      degradeForLimits.ts
      degradeForComplexity.ts

    fingerprint/
      repoFingerprint.ts

    utils/
      deepSort.ts
      stableStringify.ts

  tests/
    unit/
    integration/
    golden/
      repos/
      expected/
      fixtures/
```

**Engine Import Rules**

**MAY import:**

- its own modules
- Node stdlib (non-network, deterministic)

**MAY NOT import:**

- `arcsight-cli`
- `arcsight-github-app`
- GitHub API libraries
- network modules
- environment variables
- filesystem APIs (outside tests)

Engine determinism MUST be absolute.

---

### 📁 arcsight-github-app

**THE RUNTIME — webhook server + orchestrator**

**Responsibilities:**

- receives GitHub webhooks
- dedupes events
- fetches tarballs
- (Phase 2) builds RepoSnapshots
- calls `analyze(snapshot)`
- publishes CheckRuns
- manages rate limits
- manages DLQ
- logs & telemetry

**Runtime MUST NOT contain:**

- graph logic
- cycle logic
- rulepack logic
- deterministic canonicalization logic

```
arcsight-github-app/
  package.json
  tsconfig.json
  README.md

  src/
    server.ts

    webhook/
      routeHandler.ts
      eventRouter.ts
      dedupe.ts
      shaGuard.ts
      prStateStore.ts
      mutex.ts

    github/
      fetchTarball.ts
      getRepoMetadata.ts
      publishCheckRun.ts
      tokenManager.ts
      capabilityDetector.ts

    analysis/
      runAnalysis.ts
      convertTarballToSnapshot.ts     # Phase 1 logic: uses engine snapshot builder
      handleDegraded.ts
      handleAnalysisError.ts

    dlq/
      enqueueDLQ.ts
      processDLQ.ts

    rateLimit/
      coordinator.ts
      retryStrategy.ts

    logs/
      logger.ts
      correlator.ts

    telemetry/
      metrics.ts
      events.ts

    identity/
      identityChain.ts

    config/
      loadConfig.ts
      tierEnforcement.ts
      repoGuardrails.ts

    runbooks/
      githubOutage.ts
      rateLimitEscalation.ts

  tests/
    webhook-fixtures/
    integration/
```

**Runtime Import Rules**

**MAY import:**

- `arcsight-wedge` (public API only)

**MUST NOT import:**

- CLI
- engine internals
- rulepack internals
- test modules

Runtime is a host—not an analyzer.

---

### 📁 arcsight-cli

**THE CLI — golden tests, debugging, local analysis**

The CLI provides:

- local engine execution
- golden test comparison
- envelope inspection
- snapshot generation (Phase 1)
- debugging utilities

```
arcsight-cli/
  package.json
  tsconfig.json
  README.md

  src/
    index.ts

    commands/
      analyze.ts
      compare.ts
      replay.ts
      dumpEnvelope.ts

    utils/
      loadRepo.ts
      printEnvelope.ts
      diffEnvelopes.ts
      snapshotRepo.ts
      projectEnvelope.ts

  fixtures/
    webhook/
      pr_open.json
      pr_sync.json
      pr_reopen.json

  tests/
    integration/
    cli-golden-tests.ts
```

**CLI Import Rules**

**MAY import:**

- `arcsight-wedge` API (`analyze` + types)
- filesystem

**MUST NOT import:**

- runtime
- GitHub APIs
- engine internals

CLI is offline-only.

---

## 5. Import Relationship Diagram (Must Always Hold)

```
arcsight-github-app → arcsight-wedge
arcsight-cli        → arcsight-wedge

arcsight-wedge  ✖ imports nothing external
arcsight-github-app ✖ cannot import arcsight-cli
arcsight-cli        ✖ cannot import arcsight-github-app
```

Violating any of these edges breaks the architecture.

---

## 6. Purpose of This Document

Use this document:

- when creating new files
- when deciding where code belongs
- during PR review
- to enforce deterministic boundaries
- to prevent cross-package drift
- to maintain Phase 1 purity

Do not modify this structure until the Phase 2 monorepo migration.

---

## 7. Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Monorepo Migration](./5B-monorepo-migration.md)
- [Dependency Contract](./5A-dependency-contract.md)
