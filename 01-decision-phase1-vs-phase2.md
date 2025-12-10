# ArcSight Strategic Architecture  

## Phase 1 vs Phase 2 Structure — Final Decision Document

This document defines ArcSight's *high-level structural strategy*:  

**Separate package folders in Phase 1, monorepo in Phase 2.**

It answers:

- What structure we use now  
- When to switch to a monorepo  
- The criteria for migration  
- Why not start with a monorepo  
- What Phase 2 will introduce

This document is your **architectural constitution** for directory-level decisions.

---

# ✅ FINAL DECISION

### **Stay separate folders for Phase 1.  
Migrate to a monorepo in Phase 2.**

This is the optimal approach for:

- speed  
- clarity  
- determinism  
- future scalability  
- avoiding early tooling overhead  

---

# 🟦 WHY SEPARATE FOLDERS ARE IDEAL FOR PHASE 1

Phase 1 builds three core components:

- ✔ deterministic engine core (`arcsight-wedge`)  
- ✔ webhook runtime (`arcsight-github-app`)  
- ✔ CLI for testing and golden suites (`arcsight-cli`)  

During this phase:

- clarity beats structure  
- boundaries enforce correctness  
- tooling overhead slows development  
- each package evolves independently  

Separate folders give you:

- ⚡ fast iteration  
- 🎯 clean mental model  
- 🧪 easy golden testing  
- 🛡️ strict determinism guarantees  
- 🚫 zero cross-package entanglement  

Most Phase 1 work happens entirely within:

- `arcsight-wedge`  
- `arcsight-github-app`  
- `arcsight-cli`

A monorepo adds no benefit yet — only cost.

---

# 🟩 WHY A MONOREPO BECOMES NECESSARY IN PHASE 2

Phase 2 introduces major capabilities:

- 🔥 Rulepack modularization  
- 🔥 Safety Sentinel service  
- 🔥 Shadow-mode analyzer rollouts  
- 🔥 Schema evolution + adapters  
- 🔥 Dashboard (SaaS)  
- 🔥 Drift detection + hotspot analysis  
- 🔥 Enterprise rulepacks  
- 🔥 Telemetry indexing + analytics  

These require:

### 1️⃣ Shared types across packages  

Without a monorepo, duplication becomes dangerous.

### 2️⃣ Versioned analyzer builds  

Shadow engine rollout demands workspace versioning.

### 3️⃣ Shared golden fixtures  

CLI and engine must share a single source of truth.

### 4️⃣ Unified build + test pipeline  

Needed once multiple services exist.

### 5️⃣ Sentinel running multiple versions of the engine  

Clean imports only possible in a monorepo.

### 6️⃣ Dashboard needs structural access to telemetry + shared types  

### 7️⃣ DX tooling becomes important  

(For rapid cross-package changes)

---

# 📅 THE PERFECT MIGRATION MOMENT

You migrate to a monorepo **after Phase 1** and **before Phase 2**.

Migration conditions:

- engine deterministic  
- golden tests stable  
- runtime end-to-end path working  
- envelopes versioned  
- schema adapters defined  
- degraded mode working  
- DLQ functioning  
- no rulepack explosion yet  

This is when:

- cost is lowest  
- benefit is highest  
- no refactor is required  

---

# 🧠 WHAT THE PHASE 2 MONOREPO LOOKS LIKE

Structure:

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

Tooling:

- `pnpm workspaces` or  
- `turborepo` or  
- `yarn workspaces`

Migration time: **2–3 hours.**  
No rewrite required.

---

# 🎯 TL;DR — STRATEGIC DECISION

### **Phase 1 → Keep separate folders**  

Fastest, clearest, safest for correctness.

### **Phase 2 → Move to a monorepo**  

Required for scalability, rulepacks, sentinel, dashboard, and multi-version engines.

You are making the exact tradeoff used by Stripe, Google, Meta, and Snyk.

This document governs all future directory and structural decisions.

---

## References

- [Full System Roadmap](./03-full-system-roadmap.md)
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md)
- [Monorepo Migration](./05-monorepo-migration.md)
