# ADR 0003 — Why ArcSight MUST Remain Silent on Any Error

Status: Accepted (Frozen)  

Date: 2025-01-15  

Context: Ensuring total correctness under uncertainty.

---

# Decision

ArcSight V1 SHALL emit **no warnings at all** if ANY error, anomaly, or uncertainty is detected.

Silence is correctness.

---

# Why Silence?

1. Safety > ambition  

2. Zero false positives guaranteed  

3. Ambiguous alias → silence  

4. Segmentation confidence <0.8 → silence  

5. Performance >7s → silence  

6. Graph inconsistencies → silence  

7. Canonicalization mismatch → silence  

8. Any thrown error → silence  

This rule ensures ArcSight never lies.

---

# Examples of Error Conditions

- Ambiguous alias resolution  

- Non-deterministic cycle sets  

- Unstable import graph  

- Unexpected FS traversal  

- Type-only import artifacts  

- Normalization pipeline mismatch  

- Cycle canonicalization mismatch  

- Performance budget exceeded  

---

# NEW: Silent-First Priority Rule

When silence and completeness conflict, **silence MUST win**.  

Completeness must NEVER override Safety Contract guarantees.

This prevents attempts to "rescue" partially ambiguous cycles.

---

# Consequences

- PR bot is always safe  

- Trust becomes long-term moat  

- Implementation becomes simpler and more robust  

- Wedge reputation protected  

---

# Scope Lock Clause (NEW)

This decision is locked for V1.  

Any modification requires a new ADR and explicit SSOT update.

