# ArcSight Wedge — Glossary (V1 • Final)

**Status:** Stable  

**Purpose:** Defines authoritative terminology.  

**Rule:** ALL terms listed here are frozen for V1 and MUST NOT be replaced with synonyms.

---

# Term Stability Rule (NEW)

All terms in this Glossary are binding vocabulary.  

Cursor and contributors MUST NOT introduce:

- synonyms  

- alternate phrases  

- conceptual variations  

- abbreviations  

- reformulations  

If a new concept is needed, it MUST first be added to this glossary before used anywhere else.

This prevents semantic drift and AI-induced terminology creep.

---

# A

**Alias Resolution**  

Mapping import-path aliases to filesystem paths deterministically.

**Alias Ambiguity (NEW)**  

When an alias maps to multiple possible filesystem targets.  

ArcSight MUST remain silent.

---

# C

**Canonical Cycle**  

A dependency cycle transformed into its deterministic representation using the normalization pipeline.

**Confidence Score**  

A 0–1 score determining whether the wedge is allowed to speak.  

<0.8 → silent.

**Cycle (Dependency Cycle)**  

A directed import loop between modules or files.

**Cycle Diff**  

The set of cycles introduced by the PR (base → head).

---

# D

**Determinism**  

Same input → same output, always.

---

# F

**False Negative (FN)**  

A missed cycle. Acceptable.

**False Positive (FP)**  

A wrong warning. Forbidden.

---

# M

**Monorepo**  

Repo with multiple independent packages.  

Forced Low confidence in V1.

---

# N

**Normalization Pipeline**  

Required processing sequence:

1. POSIX normalize paths  

2. lowercase  

3. lexicographically sort inputs  

4. strip type-only imports  

5. canonicalize cycles  

6. sort cycle list  

---

# P

**PR Update Invariance**  

ArcSight MUST NOT update the PR comment unless the cycle set OR root-cause edges change.

---

# R

**Root-Cause Edge**  

The specific new import line that closes a cycle introduced in the PR.

---

# S

**Safety Switch**  

A total-silence mechanism triggered by ANY anomaly.

**Silent Mode**  

ArcSight emits nothing when uncertain.

**Snapshot Graph**  

Long-term structural history (post-wedge feature, not implemented in V1).

---

# W

**Wedge**  

Minimal, deterministic PR-surface: cycles-only.

**Wedge Freeze**  

30-day period post-release where no features may be added.

