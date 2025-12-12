# ArcSight Wedge — Production Guide (V1)

This guide governs runtime behavior, determinism constraints, and operational safety.

---

# 1. Runtime SLOs & SLA Enforcement

### SLOs:

- p95 runtime < **3 seconds**

- p99 runtime < **7 seconds**

### SLA:

If runtime exceeds **7 seconds**, ArcSight MUST treat analysis as **uncertain** and:

- produce **NO warning**

- enter **silent mode**

- still write a snapshot

This is a hard guarantee.

---

# 2. Deterministic Runtime

ArcSight MUST:

- run deterministically across machines  

- avoid all nondeterministic FS operations  

- explicitly sort all FS traversal results  

- never produce console output  

- never emit debug logs  

Any nondeterministic behavior = silent mode.

---

# 3. Error Handling (Fail Closed)

ArcSight MUST:

- treat all unexpected states as uncertainty  

- never emit partial output  

- suppress warnings when unsure  

Triggers include:

- canonicalization mismatch  

- alias ambiguity  

- partially resolved import graph  

- FS errors  

- diff inconsistencies  

---

# 4. No Logging

ArcSight MUST NOT:

- log anything  

- print debug data  

- print errors except via silent mode  

Logging MAY be added in future versions ONLY behind deterministic flags.

---

# 5. Snapshot Requirements

Snapshots MUST include:

- canonicalCycles  

- confidence  

- timestamp (UTC, deterministic second-level precision)  

Snapshots MUST NOT include:

- file paths  

- import graphs  

- user content  

---

# 6. Confidence & Segmentation

ArcSight MUST:

- require confidence ≥0.8  

- default to silence for monorepos  

- require ≥10 files  

- enforce alias resolution certainty  

Confidence MUST remain isolated from cycle detection.

---

# 7. Security Requirements

ArcSight MUST:

- not execute code  

- not transmit data  

- not mutate repo  

- remain purely local  

---

# 8. Release Gates

A version is releasable ONLY if:

- determinism suite passes  

- PR replay harness shows 0 false positives  

- runtime SLO/SLA satisfied  

- no forbidden dependency introduced  

