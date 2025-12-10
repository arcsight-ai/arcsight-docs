# ArcSight Physical Architecture  

## Phase 1 Folder Scaffold — The Developer Blueprint

This document defines the **physical file structure** for Phase 1:

- where code lives  
- what each package is responsible for  
- allowed and forbidden imports  
- the canonical directory tree for each repo  

This document is the **day-to-day reference during development**.

---

# 🧱 Phase 1 Structure (Separate Folders)

ArcSight consists of three independent codebases:

```
/arcsight-wedge      ← deterministic engine
/arcsight-github-app ← runtime orchestrator (GitHub App)
/arcsight-cli        ← local tooling + golden tests
```

Each one is completely isolated during Phase 1.  
**No monorepo. No workspace setup. No shared types yet.**

---

# 📁 1. arcsight-wedge  

### **THE ENGINE — deterministic, pure, isolated**

This package contains *all core analysis logic* and must remain:

- pure  
- deterministic  
- side-effect-free  
- GitHub-agnostic  
- network-free  
- environment-variable-free  

```
arcsight-wedge/
  package.json
  tsconfig.json
  README.md

  src/
    index.ts                        # analyze(snapshot) entrypoint

    types/
      RepoSnapshot.ts               # ONLY input to engine
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
      buildSnapshot.ts              # used by CLI + runtime

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
      schemaAdapters.ts             # v1 -> v2 migrations

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
        small-repo/
        medium-repo/
        weird-repo/
      expected/
        small-repo.json
        medium-repo.json
        weird-repo.json
      fixtures/
```

### ❗ Engine Import Rules

- **MAY import:** its own files, Node stdlib  
- **MAY NOT import:** `arcsight-cli`, `arcsight-github-app`, GitHub APIs, network libs  
- **MUST NOT** read filesystem except in tests  
- **NO timestamps, randomness, or env access**

Engine remains completely deterministic.

---

# 📁 2. arcsight-github-app  

### **THE RUNTIME — webhook receiver + orchestrator**

The runtime is a thin host:

- receives webhooks  
- dedupes  
- manages PR state  
- fetches tarballs  
- builds snapshots  
- calls `analyze()`  
- publishes Check Runs  

It does **not** do analysis or graph logic.  
It does **not** mutate envelopes.

```
arcsight-github-app/
  package.json
  tsconfig.json
  README.md

  src/
    server.ts                       # webhook server entrypoint

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
      runAnalysis.ts                # calls analyze(snapshot)
      convertTarballToSnapshot.ts
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

### ❗ Runtime Import Rules

- **MAY import:** `arcsight-wedge` (only the public `analyze` + types)  
- **MUST NOT import:** CLI  
- **MUST NOT** import engine internals, rulepacks, or test utilities  

Runtime is a host, not a co-owner of analysis logic.

---

# 📁 3. arcsight-cli  

### **THE CLI — golden tests, local analysis, debugging**

The CLI exists to:

- run local analysis  
- compare results to golden files  
- replay webhook fixtures  
- inspect envelopes for debugging  
- generate snapshots offline  

```
arcsight-cli/
  package.json
  tsconfig.json
  README.md

  src/
    index.ts                        # main CLI entrypoint

    commands/
      analyze.ts                    # arcsight analyze <path>
      compare.ts                    # arcsight compare --golden
      replay.ts                     # replay webhook fixtures
      dumpEnvelope.ts               # debugging

    utils/
      loadRepo.ts
      printEnvelope.ts
      diffEnvelopes.ts
      snapshotRepo.ts               # uses engine snapshot builder
      projectEnvelope.ts            # pretty printing

  fixtures/
    webhook/
      pr_open.json
      pr_sync.json
      pr_reopen.json

  tests/
    integration/
    cli-golden-tests.ts
```

### ❗ CLI Import Rules

- **MAY import:** `arcsight-wedge`  
- **MUST NOT import:** runtime  
- **MAY** use filesystem and local fixtures  

CLI is the developer's interface to the engine.

---

# 🧩 IMPORT RELATIONSHIPS (Phase 1)

```
arcsight-github-app → arcsight-wedge
arcsight-cli → arcsight-wedge

arcsight-wedge ✖ imports nothing external
arcsight-github-app ✖ cannot import CLI
arcsight-cli ✖ cannot import runtime
```

This prevents nondeterminism, dependency loops, and architectural rot.

---

# 🧠 Purpose of This Document

Use this document:

- when creating new files  
- when deciding where a module belongs  
- during PR reviews  
- to enforce boundaries  
- to keep Phase 1 deterministic and clean  

This is the **canonical folder architecture** for Phase 1.

Do **not** alter structure until the Phase 2 monorepo migration.

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Monorepo Migration](./05-monorepo-migration.md)
