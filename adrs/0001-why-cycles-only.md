# ADR 0001 — Why ArcSight V1 Is Cycles-Only

Status: Accepted (Frozen)  

Date: 2025-01-15  

Context: Deterministic structural signals for PR-time safety.

---

# Decision

ArcSight V1 SHALL ONLY surface **new dependency cycles** as its PR-time safety signal.  

No other signals may be implemented before wedge validation.

---

# Why Cycles?

Cycles are:

- Binary (exist or not)

- Deterministic (graph-theoretic truth)

- Universally understood by developers

- Always harmful to maintainability

- Diff-friendly (base → head)

- Segmentation-light (requires minimal project structure)

- Language-agnostic within JS/TS environments

- Guaranteed to support zero false positives

These properties make cycles the ONLY safe, deterministic, universally meaningful V1 signal.

---

# Alternatives Considered (Rejected)

### God-file warnings  

Heuristic, subjective thresholds.

### Forbidden imports  

Requires config; high noise risk.

### Boundaries / layers  

Requires architecture inference; forbidden in V1.

### Exposure, fragility, cohesion metrics  

Non-deterministic; prone to noise.

### Full drift analysis  

Too noisy for a wedge.

---

# Consequences

- PR bot trust increases  

- Developer acceptance becomes feasible  

- Wedge purity preserved  

- All expansion deferred until wedge success  

---

# Scope Lock Clause (NEW)

This decision is locked for V1.  

Any modification requires a new ADR and explicit SSOT update.

