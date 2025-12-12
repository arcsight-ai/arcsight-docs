# ArcSight Wedge — Module Boundaries (V1)

**Status:** Frozen after V1 Design Doc  

**Authority:** Foundation Contract + Safety Wedge Contract + API Contract  

---

# 1. High-Level Module Graph

```
analysis ─→ diff ─→ pr
│         │      │
├─ alias  │      │
├─ cycles │      │
└─ canonicalize
└→ confidence
└→ safety
└→ snapshot
```

Modules MUST remain isolated and pure.

---

# 2. Module Definitions and Restrictions

## 2.1 analysis/

Responsible for:

- import graph creation  

- alias resolution  

- raw cycle detection  

- canonicalization utilities  

Restrictions:

- MUST NOT inspect PR  

- MUST NOT rely on safety/snapshot modules  

- MUST NOT write data

- `aliasResolver.ts` MUST NOT import safety modules

- Safety modules may read alias results, but MUST NOT call back into analysis modules  

---

## 2.2 diff/

Responsible for:

- comparing base vs head cycles  

- detecting new cycles  

- root-cause edge detection  

- PR-diff file filtering  

Restrictions:

- MUST NOT depend on pr/, safety/, snapshot/  

- MUST NOT modify canonical cycles  

---

## 2.3 confidence/

Responsible for:

- evaluating repo quality  

- computing confidence (0–1)  

- assigning High/Low bucket  

### **Hard Isolation Rule (NEW):**

The confidence module MUST NOT:

- inspect cycles  

- inspect imports  

- inspect root-cause edges  

- reference diff outputs  

- depend on canonicalization  

Confidence MAY ONLY depend on:

- segmentation quality  

- file-count statistics  

- alias resolution success/failure  

---

## 2.4 safety/

Responsible for:

- enforcing silent mode  

- determinism checks  

- safety switch logic  

Restrictions:

- MUST NOT inspect cycle structure  

- MUST NOT access PR logic  

- MUST NOT modify outputs  

---

## 2.5 pr/

Responsible for:

- rendering one PR comment  

- updating comment only if content changes  

- handling "ignore" replies  

Restrictions:

- MUST NOT detect cycles  

- MUST NOT perform diffing  

- MUST NOT compute confidence  

---

## 2.6 snapshot/

Responsible for:

- writing cycle snapshots  

- storing confidence values  

Restrictions:

- MUST NOT compute deltas  

- MUST NOT read historical snapshots

- MUST be append-only (no read API in V1)

- MUST NOT export FS helpers usable elsewhere

- Snapshot writing has no read API in V1  

---

# 3. Forbidden Cross-Boundary Interactions

Forbidden:

- pr → analysis  

- pr → diff  

- analysis → pr  

- diff → safety  

- confidence → cycles  

- pr → confidence  

Allowed interactions ONLY as listed in the design doc.

---

# 4. Determinism Boundary

Only these modules MAY affect final output:

- canonicalize.ts  

- cycles.ts  

- cycleDiff.ts  

- rootCause.ts  

Each MUST produce stable, reproducible results.

---

# 5. Enforcement

Violations require:

- rollback  

- ADR  

- SSOT update  

Boundaries are mandatory and binding.

