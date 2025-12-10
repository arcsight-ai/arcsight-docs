# ArcSight Strategic Architecture

## Phase 1 vs Phase 2 Structure — Final Decision Document (v1.2.0)

This document defines ArcSight's high-level structural strategy:

**Phase 1 uses isolated packages inside a single repository.  
Phase 2 migrates to a unified monorepo.**

It answers:

- What structure we use now
- When we switch to a monorepo
- Why not start with a monorepo
- Criteria for migration
- What Phase 2 unlocks

This document is your **architectural constitution** for repository and directory-level decisions.

---

# ✅ FINAL DECISION

**Phase 1 → Isolated packages (within one repository)**  
**Phase 2 → Unified monorepo**

This is the optimal strategy for:

- speed
- clarity
- strict determinism
- minimized tooling overhead
- long-term scalability

---

# 🟦 WHY ISOLATED PACKAGES (WITHIN ONE REPO) ARE IDEAL FOR PHASE 1

Phase 1 builds the three foundational components:

- ✔ deterministic engine core (`arcsight-wedge`)
- ✔ webhook runtime (`arcsight-github-app`)
- ✔ CLI for golden testing (`arcsight-cli`)

During this phase:

- clarity beats structure
- boundaries enforce purity
- tooling overhead is wasteful
- each package evolves independently
- cross-package sharing is forbidden
- determinism is paramount

## 📌 Phase-1 Structural Rule

**All Phase-1 packages live in a single repository but MUST remain fully isolated.**

No imports between packages.  
No shared code.  
No shared layer or utilities.

Isolation improves:

- ⚡ iteration speed
- 🎯 conceptual clarity
- 🧪 golden test stability
- 🛡️ deterministic guarantees
- 🚫 prevents architectural coupling

A monorepo adds zero benefit here and introduces unnecessary cognitive cost.

---

# 🟩 WHY A MONOREPO IS REQUIRED IN PHASE 2

Phase 2 introduces major system capabilities:

- Rulepack modularization
- Safety Sentinel
- Shadow-mode analyzers
- Schema evolution + adapters
- Dashboard / SaaS
- Drift detection + analytics
- Enterprise rulepacks
- Telemetry ingestion + indexing

These require:

### 1️⃣ Shared canonical types and schemas

Without shared definitions, adapters and rulepacks cannot evolve safely.

### 2️⃣ Shared golden fixtures

CLI and engine must share one source of truth.

### 3️⃣ Unified build + test pipeline

Required once multiple services and analyzers exist.

### 4️⃣ Multi-engine builds (LIVE + SHADOW)

Shadow rollout requires building:

- `analyzerVersion` X (live)
- `analyzerVersion` X+1 (shadow)

simultaneously, using shared types and schema definitions.

This is only possible in a monorepo.

### 5️⃣ Sentinel lifecycle (Phase 2)

Sentinel relies on:

- synchronized analyzer versions
- deterministic drift baselines
- shared adapter logic
- stable schema evolution

None of this can be coordinated across scattered packages.

---

# 📅 THE PERFECT MIGRATION MOMENT

Migration MUST occur:

**after Phase-1 engine output is drift-stable across all golden repos,  
and before Phase 2 development begins.**

## 📌 Zero-Drift Requirement (NEW RULE)

Before monorepo migration:

**The Phase-1 analyzer MUST produce a zero-drift, golden-stable baseline on all golden test repositories.**  
Sentinel drift detection and shadow promotion rules (Docs #16 & #17) are invalid until this baseline is fully stable.

Once the engine is stable and deterministic:

- runtime end-to-end path works
- schema adapters exist
- degraded mode is correct
- DLQ is functional
- no rulepack explosion yet

→ migration is safe, low-cost, and high-value.

---

# 🧠 WHAT PHASE 2'S MONOREPO LOOKS LIKE

```
/arcsight
  /packages
    /arc-engine
    /arc-runtime
    /arc-cli
    /arc-shared
    /arc-dashboard
    /arc-sentinel
```

Supported by:

- `pnpm workspaces`
- Turborepo
- or Yarn workspaces

Migration time: **~2–3 hours, zero rewrite.**

---

# 🟦 TL;DR — STRATEGIC DECISION

**Phase 1 → Isolated packages (no shared code)**

Fastest, clearest, safest for determinism and purity.

**Phase 2 → Unified monorepo**

Required for rulepacks, Sentinel, schema evolution, dashboard, multi-engine builds, and long-term scalability.

This is the same evolutionary path used by Stripe, Google (Tricorder), Meta (Infer), and Snyk.

---

## References

- [Phase-1 Folder Scaffold](./02-phase1-folder-scaffold.md)
- [Full System Roadmap](./03-full-system-roadmap.md)
- [Monorepo Migration](./5B-monorepo-migration.md)
