# ADR 0005 — Why ArcSight Requires a 30-Day Wedge Freeze

Status: Accepted (Frozen)  

Date: 2025-01-15  

Context: Ensuring stability after V1 activation.

---

# Decision

ArcSight MUST enter a **30-day Wedge Freeze** immediately after V1 release.  

Only bug fixes allowed; NO new features or signals.

---

# Why Freeze?

1. Developer trust requires stability.  

2. Want >30 days of real-world evidence that false positives = 0%.  

3. Prevents feature creep during sensitive early days.  

4. Ensures reproducibility across many repos.  

5. Guarantees that expansion begins from a stable foundation.

---

# Freeze Behavior

Allowed:

- bug fixes  

- determinism tightening  

- error-path refinements  

- safety switch improvements  

Forbidden:

- new signals  

- architectural changes  

- performance optimizations risky to determinism  

- UI or platform upgrades  

---

# Consequences

- Reduces risk  

- Improves partner confidence  

- Prepares for Stage 1 signal expansion  

- Ensures platform roadmap sequencing is correct  

---

# Scope Lock Clause (NEW)

This decision is locked for V1.  

Any modification requires a new ADR and explicit SSOT update.

