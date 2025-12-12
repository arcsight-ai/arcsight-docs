# ArcSight Wedge — Determinism Requirements (V1)

**Status:** Frozen  

**Purpose:** Guarantees reproducible, zero-noise behavior.  

**Authority:** Foundation Contract + Safety Wedge Contract  

Determinism is the core requirement of ArcSight Wedge.  

If determinism fails, the wedge fails.

---

# 1. Deterministic Input → Deterministic Output

For any repo state:

- same input MUST produce same output  

- across machines  

- across Node versions  

- across FS orderings  

- across repeated runs  

ArcSight MUST avoid:

- concurrency  

- environment-based logic  

- unordered data structures  

- nondeterministic APIs  

---

# 2. Filesystem Determinism

ArcSight MUST:

- sort all file listings explicitly  

- normalize paths (POSIX)  

- lowercase all paths  

- never assume FS ordering  

Failure → determinism violation → silent mode.

---

# 3. Graph Traversal Determinism

Import graph traversal MUST:

- use lexicographically sorted node lists  

- avoid recursion that depends on object insertion order  

- operate on normalized paths  

Cycle detection MUST be stable across runs.

---

# 4. Canonicalization Determinism

Canonical cycle output MUST follow exact steps:

1. POSIX-normalize  

2. lowercase  

3. rotate cycle to lexicographically smallest node  

4. join nodes with `" → "`  

5. sort final cycle list  

Any deviation → determinism violation.

---

# 5. Alias Resolution Determinism

Alias maps MUST:

- be deterministic  

- resolve to exactly one path  

- be based on static config  

If alias resolution is ambiguous:

- MUST enter silent mode  

---

# 6. Diff Determinism

Cycle diff MUST:

- compare canonical lists  

- detect new cycles deterministically  

- produce predictable ordering:

  1. cycles touching PR-diff files  

  2. fewer nodes  

  3. lexicographically smaller  

---

# 7. Safety Switch Determinism

SafetySwitch MUST:

- activate on inconsistencies  

- remain active deterministically  

- ensure **silent mode**  

Triggers include:

- repeated-run mismatch  

- canonicalization mismatch  

- alias ambiguity  

- performance >7s  

---

# 8. No Logging

ArcSight MUST NOT:

- log to console  

- print debug messages  

- emit errors except via silent mode  

Logging = nondeterminism.

---

# 9. No Randomness

ArcSight MUST NOT use:

- Math.random  

- crypto.randomUUID  

- Date.now  

- performance.now  

- timers  

Randomness breaks determinism.

---

# 10. **No Hidden State or Caching (NEW)**

ArcSight MUST NOT:

- maintain mutable module-level state  

- persist caches across runs  

- memoize results between PR evaluations  

- retain mutable Maps/Sets  

- store resolved alias maps in global scope  

All module-level state MUST be:

- constant  

- static  

- readonly  

---

# 11. Repeated-Run Consistency Rule

For any test scenario or PR replay:

```
run wedge 3× → outputs must be byte-for-byte identical
```

If mismatch:

- determinism violated  

- SafetySwitch MUST activate  

- release/merge blocked  

---

# 12. CI Determinism Rules

CI MUST enforce:

- determinism suite  

- normalization pipeline  

- performance SLO/SLA  

- FS determinism  

Flakiness = failure.

---

# 13. Deterministic Normalization Pipeline (NEW)

All modules MUST apply normalization in the exact sequence:

1. **Normalize Paths**

   - POSIX (`/`)

   - lowercase

2. **Sort Inputs**

   - file lists  

   - import edges  

   - alias map keys  

   - directories  

3. **Strip Noise**

   - type-only imports  

   - whitespace-only differences  

4. **Canonicalize Cycles**

   - POSIX normalize  

   - lowercase  

   - rotate smallest  

   - join `" → "`  

   - sort  

No module may:

- reorder these steps  

- skip steps  

- canonicalize before sorting  

- mix normalization into detection logic  

Violation → determinism failure → silent mode.

---

# 14. Authority

This document is binding for:

- implementation  

- testing  

- review  

- release  

It is subordinate only to the Foundation Contract.

