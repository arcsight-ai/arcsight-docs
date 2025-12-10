# ArcSight — Full System Roadmap  

### FINAL CTO SIGN-OFF  

### Phase 1 → Phase 2 Complete Execution Plan

This document defines:

- Architecture Blueprint  
- Build Order  
- Determinism Rules  
- Canonicalization Rules  
- Snapshot → Envelope Contract  
- Cycle Detection  
- Rulepacks  
- Invariants  
- Degraded Mode  
- Schema Evolution  
- Runtime Orchestration  
- Golden Tests  
- DLQ + retries  
- Phase 2 Migration Plan  
- Shadow Analyzer Rollout  
- Enterprise Roadmap  

This is the **complete execution manual** for implementing ArcSight correctly.

---

# 🟦 SECTION 1 — HIGH-LEVEL DECISION

### ✅ Phase 1 = Separate Folders  

### ✅ Phase 2 = Monorepo Migration  

This structure is the optimal balance of:

- speed  
- clarity  
- determinism  
- future scalability  

No changes needed. This is locked.

---

# 🟦 SECTION 2 — WHY SEPARATE FOLDERS (PHASE 1)

Phase 1 primarily builds:

- deterministic engine core  
- GitHub runtime (webhook orchestrator)  
- CLI (golden tests + developer tooling)

Separate packages give us:

- no workspace/tooling overhead  
- fast iteration  
- minimal cognitive load  
- zero risk of circular imports  
- guaranteed purity inside the engine  

This is the **only correct** approach for Phase 1.

---

# 🟩 SECTION 3 — WHY MONOREPO (PHASE 2)

Phase 2 introduces:

🔥 Rulepack modularization  
🔥 Safety Sentinel  
🔥 Shadow-mode analyzer versions  
🔥 Schema upgrades (v1 → v2 → v3)  
🔥 SaaS Dashboard  
🔥 Drift detection  
🔥 Telemetry aggregation  
🔥 Enterprise rulesets  

These require:

1. Shared types across packages  
2. Shared golden fixtures  
3. Multiple versions of the engine loaded side-by-side  
4. Unified builds  
5. Unified testing  
6. Shared schema definitions  
7. Shared envelope formats  

Therefore, Phase 2 requires a monorepo.

---

# 🟧 SECTION 4 — PHASE 1 FOLDER SCAFFOLD (Code Structure)

This section defines **where every line of code goes in Phase 1**.

## 📁 arcsight-wedge  

**THE ENGINE — pure, deterministic, GitHub-agnostic**

```
src/
  index.ts

  types/
  canonicalization/
  snapshot/
  graph/
  rulepacks/
  invariants/
  envelopes/
  versions/
  adapters/
  sandbox/
  degraded/
  fingerprint/
  utils/

tests/
  unit/
  integration/
  golden/
    repos/
    expected/
    fixtures/
```

Engine Rules:

✔ Deterministic  
✔ Pure functions only  
✔ No network  
✔ No environment variables  
✔ No timestamps/randomness  
✔ No GitHub objects  
✔ Reads NOTHING from disk when analyzing  

RepoSnapshot is the ONLY input.  
Envelope is the ONLY output.

---

## 📁 arcsight-cli  

**Developer Tool — golden tests, diffing, replay**

```
src/
  commands/
  utils/
  index.ts

fixtures/
tests/
```

CLI Rules:

✔ May read filesystem  
✔ May consume engine  
✔ Must not import runtime  
✔ Golden tests live here and engine must satisfy them  

---

## 📁 arcsight-github-app  

**Runtime Orchestrator — the GitHub App**

```
src/
  server.ts

  webhook/
  github/
  analysis/
  dlq/
  rateLimit/
  logs/
  telemetry/
  identity/
  config/
  runbooks/

tests/
```

Runtime Rules:

✔ Only orchestrates  
✔ Never performs graph logic  
✔ Never mutates envelopes  
✔ Uses engine as a black-box  

---

# 🟣 SECTION 5 — PHASE 1 IMPLEMENTATION ORDER  

### **Do not change this order. This prevents chaos and nondeterminism.**

---

## STEP 1 — Implement arc-engine (wedge)  

### **The engine must be 100% complete and stable before anything else.**

Build in this order:

1️⃣ Canonicalization  
- normalizePaths  
- normalizeNewlines  
- deterministic ordering  
- canonicalRepoSnapshot  
- canonical hashing  

2️⃣ RepoSnapshot Format & SnapshotBuilder  
- snapshot is a pure representation  
- no runtime or environment behavior  

3️⃣ Graph Builder  
- buildGraph  
- graphStats  

4️⃣ Cycle Detector  
- detectCycles  
- cycle classification  

5️⃣ Invariants  
- validateGraph  
- validateEnvelope  

6️⃣ Envelope Builder  
- buildCoreEnvelope  
- attachExtensions  
- signEnvelope  

7️⃣ Degraded Mode  
- degradeForLimits  
- degradeForComplexity  

8️⃣ Schema Adapters  
- v1 → v2 migration logic  

9️⃣ Fingerprint  
- repoFingerprint  

🔟 Golden Repos + Golden Tests  
- small  
- medium  
- weird  

Engine must produce stable, byte-for-byte golden envelopes.

💯 Hard-code analyzerVersion & schemaVersion  

These anchor the lifecycle.

---

## STEP 2 — Implement arc-cli  

Commands:

- `arcsight analyze ./repo`  
- `arcsight compare --golden`  
- `arcsight replay <fixture>`  
- `arcsight dump-envelope <sha>`  

CLI must use:

✔ snapshot builder from engine  
✔ analyze() from engine  
✔ projectEnvelope() for pretty printing  

---

## STEP 3 — Implement arc-runtime (GitHub App)

1️⃣ Webhook server  
2️⃣ Dedupe  
3️⃣ SHA guard  
4️⃣ PR-level mutex  
5️⃣ Token manager  
6️⃣ Tarball fetcher  
7️⃣ Tarball → RepoSnapshot  
8️⃣ analyze(snapshot)  
9️⃣ CheckRun publisher  
🔟 DLQ (dead-letter queue)  
1️⃣1️⃣ Rate-limit coordinator  
1️⃣2️⃣ Logging + telemetry  

At this point:

PR → ArcSight → CheckRun works end-to-end.

---

# 🟩 SECTION 6 — PHASE 2 MONOREPO MIGRATION

The future monorepo structure:

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
    arc-dashboard
    arc-sentinel
```

Move each existing folder into `/packages/**`.

Monorepo adds:

- shared types  
- unified builds  
- shared golden tests  
- sentinel  
- dashboard  
- shadow analyzers  

Migration takes **2–3 hours**.  
Zero rewrite.

---

# 🟥 SECTION 7 — FUTURE: SHADOW ANALYZER PIPELINE (Phase 2)

After engine v1.0.0 is stable:

```
/arc-engine/dist/v1.0.0        ← live
/arc-engine/dist/v1.1.0-shadow ← experimental
```

Runtime loads both:

- runs both engines  
- compares envelopes  
- reports deltas  
- only publishes stable version  
- sentinel monitors drift  

This is how Google, Stripe, and Meta roll out static analysis.

---

# 🟨 SECTION 8 — SCHEMA LIFECYCLE

Schema v1 is stable in Phase 1.

Future schema versions require:

- backwards-compatible adapters  
- envelope invariants  
- extension-only evolution  
- stable identity chain  
- stable analyzerVersion  

---

# 🟦 SECTION 9 — ENTERPRISE ROADMAP (Phase 2+)

Later additions include:

- Dashboard  
- Org-level insights  
- Hotspot analysis  
- Drift detection  
- Slack notifications  
- Long-term archival  
- Enterprise rulepacks  
- Boundary rules  
- Forbidden imports  
- Tiered limits  

All follow the same envelope/telemetry foundation.

---

# 🎯 FINAL SUMMARY

This roadmap defines:

- **Exactly what to build**
- **Exactly when to build it**
- **Exact order**
- **Exact invariants**
- **Exact purity rules**
- **Exact folder boundaries**
- **Exact migration plan**
- **Exact future evolution path**

This is the complete execution architecture for ArcSight.

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Monorepo Migration](./05-monorepo-migration.md)
- [Dependency Contract](./06-dependency-contract.md)
