# ArcSight Wedge — New Signal Onboarding Guide (Post-Wedge Only)

**Status:** Frozen until wedge validation succeeds  

**Authority:** Foundation Contract + V1 Safety Wedge Contract + ADRs  

**Purpose:** Ensures safe, deterministic, low-noise onboarding of new structural signals.

---

# 1. Definition: What Is a "Signal"? (NEW)

A **signal** is a deterministic, user-visible PR-surface output describing a structural safety issue.

Examples of valid signals (post-wedge):

- new dependency cycle  

- forbidden import violation  

- god-file worsening  

NOT signals:

- logs  

- metrics  

- noise  

- diagnostic data  

- UI charts  

- platform dashboards  

Only signals affect user-facing PR behavior.

---

# 2. Preconditions (ALL Required)

A new signal MAY ONLY be considered if:

- wedge success criteria achieved  

- wedge freeze completed  

- 0% false positives sustained  

- partner approves wedge behavior  

- ADR created for the new signal  

- SSOT updated  

If ANY precondition is missing → the signal is rejected.

---

# 3. Required Documentation

### 3.1 ADR

Including:

- motivation  

- deterministic definition  

- silent-mode triggers  

- failure modes  

- error-path behavior  

### 3.2 API Contract Addendum

Clearly define:

- new fields  

- allowed values  

- type signatures  

- diff behavior  

### 3.3 Testing Plan

Must include:

- determinism tests  

- replay harness validation  

- canonicalization invariance  

- error-path silence  

### 3.4 Determinism Proof

Demonstrate:

- normalization pipeline applies  

- ordering invariant  

- no hidden state introduced  

---

# 4. Safety Requirements

Any new signal MUST:

- have zero false positives  

- follow silent-on-uncertainty rules  

- be diff-based (only warn on new issues)  

- be universally understood by developers  

- avoid architectural inference  

- NOT require AST parsing  

- NOT rely on module semantics  

---

# 5. Prohibited Signal Classes (Hard Ban)

Not allowed in wedge expansions:

- architecture inference  

- layering violations  

- implicit boundaries  

- heuristic metrics  

- scoring systems  

- runtime views  

- platform governance  

These require full platform maturity.

---

# 6. Implementation Requirements

Signal modules MUST:

- be pure  

- be deterministic  

- follow module boundaries  

- avoid shared state  

- avoid caching/memoization  

- preserve safety switch logic  

---

# 7. Release Criteria

A signal MAY ship only after:

- tests fully pass  

- determinism suite green  

- replay harness yields 0 FPs  

- partner approves  

- ADR and SSOT updated  

---

# 8. Authority

This guide governs all future signals.  

Cursor MUST NOT implement new signals without ADR approval.

