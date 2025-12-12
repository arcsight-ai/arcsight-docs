# ArcSight Platform Roadmap (Informational Only)

**Status:** Informational — NOT executable  

**Purpose:** Provides long-term direction AFTER wedge validation.  

**Authority:** Foundation Contract + ADRs + SSOT (post-V1 only)

---

# Roadmap Prohibition Clause (Critical)

NOTHING in this roadmap grants permission to implement ANY feature outside V1.

Cursor MUST treat ALL roadmap items as informational ONLY.

Implementation of ANY roadmap item requires:

- wedge success  

- ADR approval  

- SSOT update  

- design review  

V1 is cycles-only. The roadmap is NOT an instruction set.

---

# Stage 0: Safety Wedge (Current)

Deliver:

- cycles-only PR safety signal  

- silent fallback  

- deterministic engine  

- Snapshot Graph storage (hidden)  

Outcome:

- trust  

- usage  

- correctness  

---

# Stage 1: Additional Safety Signals (Post-Wedge)

Possible future signals:

- god-file worsening  

- forbidden imports (config-driven)  

- mutual imports  

- re-export cycles  

Constraints:

- deterministic  

- diff-only  

- silent on uncertainty  

---

# Stage 2: Drift Timeline & Observability

Potential features:

- cycle history timeline  

- drift visualizations  

- structural trend summaries  

Constraints:

- no governance  

- no scoring  

- silent fallback preserved  

---

# Stage 3: Team Insights Layer

Potential:

- module → team ownership mapping  

- cross-team drift  

- structural hotspots  

Constraints:

- explicit config  

- opt-in  

---

# Stage 4: Governance Layer (Optional)

Potential:

- drift budgets  

- team limits  

- organization-wide safety rules  

Constraints:

- warnings-only  

- opt-in  

- policy system MUST be deterministic  

---

# Stage 5: AI Structural Intelligence (Future)

Possible:

- cycle repair suggestions  

- module extraction proposals  

- structural forecasting  

Constraints:

- large snapshot dataset required  

- must remain deterministic when surfacing signals  

---

# Principle

Nothing may compromise:

- determinism  

- safety switch  

- silent fallback  

- wedge trust  

- developer experience  

The wedge ALWAYS remains the foundation.

