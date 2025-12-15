⚠️ LOCKED DOCUMENT

This document defines ArcSight's permanent posture.

Any change that weakens these principles is a regression.

See Section 10 (Governance).

---

# ArcSight — Philosophy & Operating Model (Locked)

This document defines ArcSight's permanent posture. Any change that weakens these principles is a regression.

## 1. What ArcSight Is (Locked)

ArcSight reports observed architectural facts.

These facts are:

* Deterministic — reproducible on any machine

* Provable — derived from language semantics and structural graphs

* Irreversible — they cannot become false without code changes

* Non-argumentative — they require no interpretation

* Safe to suppress — silence is preferred to uncertainty

Observed facts may be incomplete by design. Absence of a signal does not imply absence of structure.

ArcSight does not explain, recommend, rank, or enforce by default. Details are always opt-in.

This is the core identity. Everything else serves it.

## 2. Silence Semantics (Constitutional)

Silence is an intentional outcome, not an error state.

* Silence indicates that ArcSight chose not to surface a signal

* Silence does not imply:

    * completeness

    * safety

    * absence of structure

    * absence of risk

Silence means:

"No architectural fact met the bar for safe, final reporting."

Silence must never be replaced with speculation, approximation, or guidance.

This rule is absolute.

## 3. The Three Gates for Core Signals (Frozen)

A signal may appear in ArcSight's default summary only if all three are true:

1️⃣ Provable

It is derivable from language semantics or structural graphs — not heuristics.

2️⃣ Irreversible

If it exists, the only way to make it disappear is to change the code.

3️⃣ Permanent

Once introduced, it is structurally expected to persist unless consciously removed.

Permanence refers to architectural nature, not historical duration.

Default summary = final + permanent only

Anything else belongs in:

* --details

* history

* rulepacks

* CI enforcement (explicitly wired)

This rule must never be weakened.

## 4. Architectural Finality (Bounded Expansion)

Final signals may be added over time. The definition of final must never loosen.

### Language-Scoped Finality

Final signals are scoped to language semantics, not universal across ecosystems.

Examples (illustrative only):

* JS / TS — import cycles, declared boundary violations

* Go — import cycles, init dependency loops

* Rust — ownership graph contradictions

* JVM — module / classloader boundary violations

Final signals must be grounded in:

* compiler behavior

* resolvers

* dependency graphs

They must not be grounded in:

* framework idioms

* community best practices

* stylistic norms

Finality is:

* closed per language

* open over time

* explicitly versioned

## 5. Rulepacks: Preferences Without Corruption

Rulepacks encode organizational preferences, not architectural facts.

They may include:

* thresholds

* policies

* conventions

* enforcement choices

Rules:

* Rulepacks are user-authored

* Rulepack failures never change default summary

* Rulepack output is visually and semantically segregated

* ArcSight evaluates rulepacks but never endorses them

* Rulepack output must be phrased as evaluation, never as fact

Example:

* ❌ "Layer violation detected"

* ✅ "Rulepack evaluated this as a violation"

ArcSight reports facts. Teams decide what they care about.

This line is sacred.

## 6. Architectural Memory (Append-Only)

ArcSight is not a diagnostic tool. It is a structural memory system.

Principles:

* Architectural memory is append-only

* Facts may resolve; history is never rewritten

* Replays must be reproducible

* Time is a first-class dimension, not a score

This enables:

* defensible reviews

* institutional memory

* blame-free accountability

## 7. Explicit Refusals (Permanent)

These are non-goals.

❌ ArcSight Will Never Rank Architecture  
No scores, grades, maturity ladders, or comparisons.

❌ ArcSight Will Never Predict Risk  
No probabilities, likelihoods, forecasts, or danger zones.

❌ ArcSight Will Never Suggest Refactors  
No fixes, advice, or best practices.

❌ ArcSight Will Never Infer Intent  
Only what exists structurally is observed.

Requests that violate these are rejected by design.

## 8. Expansion Model (Safe Forever)

ArcSight expands by asking the same question at larger scopes:

"What here is structurally true and non-negotiable?"

Valid expansion axes:

* file → module → package → subsystem

* repo → multi-repo → org graph

* snapshot → history → lineage

Invalid axes:

* heuristics

* AI judgment

* productivity metrics

* opinionated guidance

Same discipline. Larger surface.

## 9. Emotional & Social Contract

ArcSight must never be the thing engineers argue with.

Instead:

* engineers argue with facts (rare)

* or with their own rulepacks

This makes ArcSight safe in:

* CI

* design reviews

* post-mortems

* enterprise governance

## 10. Governance (Non-Optional)

This philosophy may be modified only if all are true:

* The change is backward-compatible with existing summaries

* Silence semantics are preserved

* No new signal bypasses the three gates

* The change is justified by a concrete, provable counterexample

Any change that weakens finality, permanence, or silence is a regression.

**No Further Debate Clause:**

When philosophy and usefulness conflict, philosophy wins.

---

## Final Assessment

This version is:

* future-proof

* legally defensible

* enterprise-safe

* culturally rare

* extremely hard to copy

Most tools collapse under success. This document is how ArcSight doesn't.

Nothing else should be added now.

Next work should be observation, not building.
