# ADR 0002 — Why False Negatives Are Acceptable in V1

Status: Accepted (Frozen)  

Date: 2025-01-15  

Context: Safety-first deterministic PR-time engine.

---

# Decision

ArcSight V1 SHALL ACCEPT **false negatives** (missed cycles) but SHALL NOT allow **any false positives**.

Zero false positives is the absolute priority.

---

# Why?

1. Developer trust is fragile.  

   One incorrect warning → PR bot muted.

2. Silent fallback is safer than guesswork.  

   Warning only with full certainty.

3. Determinism must not be compromised.

4. Completeness is optional; correctness is mandatory.

5. Developers will forgive missed warnings;  

   they will NOT forgive wrong warnings.

---

# Alternatives Considered (Rejected)

### Prioritizing completeness  

Leads to heuristics → non-determinism → noise.

### Warning on low confidence  

Violates the Safety Contract → guaranteed wedge death.

---

# Consequences

- Stable adoption  

- Zero noise  

- Strict purity of V1  

- Predictable PR-surface behavior  

---

# Scope Lock Clause (NEW)

This decision is locked for V1.  

Any modification requires a new ADR and explicit SSOT update.

