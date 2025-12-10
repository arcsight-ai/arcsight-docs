# ArcSight — Cross-Package Dependency Contract  

### (Phase 1 & Phase 2 Rules)

This document defines the **official, enforced dependency rules** between all ArcSight packages.

Breaking these rules will:

- undermine determinism  
- create circular dependencies  
- make golden tests unreliable  
- blur engine/runtime boundaries  
- prevent monorepo migration  
- break shadow rollouts and schema evolution  

This is the **architectural safety contract**.

---

# 🟦 PHASE 1 — Allowed Dependencies (Three Separate Repos)

During Phase 1, ArcSight consists of:

```
arcsight-wedge      ← deterministic engine
arcsight-cli        ← developer tooling
arcsight-github-app ← runtime
```

Below are the **only** allowed directions.

---

## ✅ 1. arcsight-wedge (ENGINE)

### MAY import:

- internal engine modules  
- its own types  
- Node.js standard library (non-network, deterministic parts only)  

### MAY NOT import:

- arcsight-cli  
- arcsight-github-app  
- GitHub API clients  
- any network libraries  
- filesystem (`fs`) — except *inside tests only*  
- environment variables  
- timestamps or randomness  
- global state  

### WHY  
The engine must remain **100% deterministic, pure, sandboxable, cacheable, testable**.

This is the core invariant of ArcSight.

---

## ✅ 2. arcsight-cli (DEVELOPER TOOLING)

### MAY import:

- arcsight-wedge  
- local filesystem (`fs`)  
- local fixture files  
- snapshot builder from engine  

### MAY NOT import:

- arcsight-github-app  
- GitHub types  
- network modules  
- Check Run schemas or runtime-specific logic  

### WHY  
CLI is a **consumer of the engine**, not a host layer.

It must remain offline, fast, and strictly local.

---

## ✅ 3. arcsight-github-app (RUNTIME)

### MAY import:

- arcsight-wedge (analyze)  
- GitHub API client libraries (`@octokit/*`)  
- filesystem (temp tarball handling only)  
- queueing / retry logic  
- telemetry modules  

### MAY NOT import:

- arcsight-cli  
- test fixtures  
- golden outputs  
- engine test helpers or internal-only modules  

### WHY  
Runtime is the orchestrator.  
It must remain thin, stateless, and bounded.

It may call `analyze()` but **never influence or mutate the engine**.

---

# 🚫 ABSOLUTELY FORBIDDEN IN PHASE 1

### ❌ runtime → cli  
### ❌ engine → cli  
### ❌ engine → runtime  
### ❌ cli → runtime  
### ❌ shared state between packages  
### ❌ importing test fixtures across repos  
### ❌ duplicating types in multiple packages  

These rules protect the architecture from accidental coupling.

---

# 🟩 PHASE 2 — Allowed Dependencies (Monorepo)

When migrating to the monorepo, packages become:

```
packages/
  arc-engine
  arc-runtime
  arc-cli
  arc-shared
  arc-sentinel
  arc-dashboard
```

The **allowed dependency graph** becomes:

```
       arc-engine
        /      \
       /        \
arc-cli /        \ arc-sentinel
      /            \
arc-runtime ---> arc-dashboard
      \
       arc-shared
```

---

## 📦 arc-engine (deterministic analyzer)

### MAY depend on:

- `arc-shared` only

### MUST NOT depend on:

- arc-runtime  
- arc-cli  
- arc-sentinel  
- arc-dashboard  

Why:  
Engine remains pure and versioned independently.

---

## 📦 arc-cli (local developer tooling)

### MAY depend on:

- arc-engine  
- arc-shared  

### MUST NOT depend on:

- arc-runtime  
- arc-dashboard  

Why:  
CLI is always offline and must never interact with runtime concerns.

---

## 📦 arc-runtime (GitHub App host layer)

### MAY depend on:

- arc-engine  
- arc-shared  

### MUST NOT depend on:

- arc-cli  

### MAY communicate with:

- arc-sentinel via queues/events (not imports)

---

## 📦 arc-dashboard (SaaS UI)

### MAY depend on:

- arc-shared  
- telemetry outputs from runtime  

### MUST NOT depend on:

- arc-engine directly  
- arc-cli  
- arc-sentinel  

Why:  
Dashboard is an observer, not an analyzer.

---

## 📦 arc-sentinel (shadow analyzers + safety system)

### MAY depend on:

- arc-engine (multiple versions at once)  
- arc-shared  

### MUST NOT depend on:

- arc-runtime  

Why:  
Sentinel evaluates analyzer correctness independently of runtime.

---

## 📦 arc-shared (types & schemas)

### MAY NOT depend on any other package.

`arc-shared` is the **root of truth**, imported but never importing.

---

# 🚫 PHASE 2 FORBIDDEN IMPORTS

```
arc-runtime ❌→ arc-cli
arc-dashboard ❌→ arc-engine
arc-engine ❌→ arc-runtime
arc-sentinel ❌→ arc-runtime
arc-cli ❌→ arc-runtime
```

If any of these edges appear, architecture integrity is broken.

---

# 🧠 WHY THIS CONTRACT MATTERS

This prevents:

- circular dependencies  
- schema inconsistencies  
- engine impurity  
- nondeterministic behavior  
- accidental GitHub logic inside the engine  
- tangled monorepo migrations  
- broken shadow analyzers  
- version skew issues  
- test instability  

This contract guarantees:

- clean separation of concerns  
- deterministic analysis  
- predictable deployments  
- safe schema migrations  
- stable shadow-mode rollouts  
- enterprise-grade architecture  

---

# 🎯 FINAL CHECKLIST — MUST ALWAYS HOLD TRUE

- Engine is always pure.  
- CLI is always offline.  
- Runtime is always a thin host.  
- Shared lives in one place.  
- No circular dependencies ever.  
- No cross-test imports.  
- No engine → runtime imports.  
- No dashboard → engine imports.  
- Sentinel can read engine versions but not runtime.  
- Monorepo remains clean and scalable.

This is the architectural boundary that makes ArcSight **scalable, safe, deterministic, and future-proof.**

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)
- [Usage Model](./04-usage-model.md)

