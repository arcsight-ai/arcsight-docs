# ArcSight System Surfaces (Registry)

This document is a map of the system, not a promise.

It defines where truth is created, where it is guarded, where it is filtered, and where it is presented.

This registry prevents drift, prevents category errors, and protects the wedge.

---

## 1. Engine Truth Surfaces (Authoritative)

These produce truth.

They are frozen, scarce, and governed.

### Discover (Snapshot Truth)

- Dependency cycles (implemented)

(Deferred discover candidates live here, but are not active)

### Check (Temporal Truth)

- Change impact propagation
- Boundary crossing caused by change
- Protected surface intersection
- Cycle introduction / removal
- Dependency edge changes
- File add / delete / move

**These are the only places truth is created.**

---

## 2. Invariants (Truth Guardians)

These do not create truth.
They defend the engine against itself.

**Purpose:**

- Prevent false negatives
- Prevent classification drift
- Enforce determinism
- Enforce canonicalization stability

**Examples:**

- validateFalseNegative
- validateClassificationDrift
- validateUnexpectedCycle
- determinism checks
- frozen-core guards

**Rules:**

- Invariants must depend on engine truth
- Invariants must never invent facts
- Invariants may fail the run, but never emit signals

**This is a safety layer, not a signal layer.**

---

## 3. Rulepacks (Interpretation Filters)

Rulepacks do not detect anything.

They:

- Filter
- Scope
- Suppress
- Tag
- Route

**Examples:**

- Protected surface definitions
- "Only show cycles touching X"
- "Ignore boundary crossings in test/**"

**Rules:**

- Rulepacks may only consume Check/Discover output
- Rulepacks must never compute structure
- Rulepacks must never change truth
- Rulepacks are allowed to be opinionated

**This is where org meaning safely lives.**

---

## 4. Adapters (Translation Layer)

Adapters change language, not meaning.

**Examples:**

- TypeScript adapter
- Go adapter
- Python adapter

**Rules:**

- Adapters may build graphs
- Adapters must not infer semantics
- Adapters must output canonical engine inputs
- Adapters must not contain policy

**Adapters exist to feed the engine, not reshape it.**

---

## 5. Presentation / Surfaces (Output Only)

These render truth but never alter it.

**Examples:**

- CLI
- GitHub PR comments
- JSON output
- Dashboards

**Rules:**

- No aggregation that implies meaning
- No scoring
- No health summaries
- Silence must be preserved

---

## 6. Configuration (Declared Context)

Config is declared, never inferred.

**Examples:**

- Protected surfaces
- Ignored paths
- Adapter options

**Rules:**

- Config may narrow relevance
- Config must never expand truth
- Config must never explain absence

---

## 7. Explicitly Forbidden Surfaces

These must never appear anywhere.

- Scores
- Risk levels
- Health metrics
- "Architecture quality"
- "Why nothing was found"
- Completeness guarantees

---

## Why This Registry Exists

This registry does three critical things:

1. **Prevents drift**
   - New contributors know where something belongs

2. **Prevents category errors**
   - No more "should this be Discover or Check?"

3. **Protects the wedge**
   - Truth, policy, and presentation never collapse again

**Importantly:**

This list does not create pressure to build more.
It creates pressure to put things in the right place.

---

## Governance

This document is frozen.

Changes require explicit architectural decision and must not weaken the boundaries between truth, policy, and presentation.

See [CONSTITUTION.md](./CONSTITUTION.md) and [philosophy.md](../philosophy.md).


