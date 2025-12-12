# ADR 0004 — Why Alias Resolution Is Not Guaranteed in V1

Status: Accepted (Frozen)  

Date: 2025-01-15  

Context: Safety-first deterministic import graph analysis.

---

# Decision

ArcSight V1 SHALL attempt alias resolution **only when deterministic**.  

If alias resolution is ambiguous, incomplete, multi-target, or tool-specific → ArcSight MUST remain silent.

Alias resolution is **best-effort**, not guaranteed.

---

# Why Not Guarantee It?

1. Aliases vary widely across build systems.  

2. Ambiguous paths create false positives.  

3. Heuristics violate determinism.  

4. AST parsing is explicitly forbidden (V1 contract).  

5. Silent fallback is safer and correct.

---

# Acceptable Behavior

- Extracting alias maps from tsconfig/jsconfig  

- Normalizing & sorting alias keys and values  

- Applying normalization pipeline to resolved paths  

---

# Unacceptable Behavior

- Guessing missing alias values  

- Using heuristics  

- Toolchain-specific inference  

- Non-deterministic path resolution  

---

# Consequences

- Less noise  

- Higher trust  

- Perfect alignment with Silent Contract  

- Safe foundation for future alias expansion  

---

# Scope Lock Clause (NEW)

This decision is locked for V1.  

Any modification requires a new ADR and explicit SSOT update.

