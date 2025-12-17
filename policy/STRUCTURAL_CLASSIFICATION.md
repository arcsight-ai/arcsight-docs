# Structural Classification Contract

This document defines where logic is allowed to live
and what it is allowed to depend on.

Violations of this contract are architectural errors.

This is not guidance.
This is a routing table.

---

## Discover Analyzers

**Location**
- `src/core/analyzers/**/discover*`

**Purpose**
- Enumerate engine truth from a single snapshot.

**May**
- Read snapshot graphs
- Consume canonical structural substrates
- Emit violations

**Must NOT**
- Read diffs or PR state
- Compare snapshots
- Perform temporal analysis
- Define heuristics or thresholds
- Depend on Check logic

---

## Check Analyzers

**Location**
- `src/core/analyzers/**/check*`

**Purpose**
- Compare truth across time.

**May**
- Read diffs
- Compare snapshots
- Consume Discover truth

**Must NOT**
- Detect new structural truth
- Canonicalize independently
- Emit Discover violations
- Run during Discover execution

---

## Structural Substrates

**Location**
- `src/core/domains/**`

**Purpose**
- Perform pure, deterministic structural computation.

**May**
- Compute graphs, reachability, ordering, normalization

**Must NOT**
- Emit violations
- Read diffs or PR state
- Know about Discover or Check
- Encode policy or relevance

---

## Invariants

**Location**
- `src/invariants/**`

**Purpose**
- Enforce internal consistency and impossibilities.

**May**
- Validate outputs
- Assert invariants

**Must NOT**
- Detect new truth
- Replace analyzers
- Modify results
- Introduce alternative interpretations

---

## Rulepacks

**Location**
- `src/rulepacks/**`

**Purpose**
- Filter, scope, or suppress existing truth.

**May**
- Filter violations
- Scope relevance

**Must NOT**
- Detect
- Compute
- Canonicalize
- Create new signals

---

## Default Rule (Hard Stop)

If a file does not clearly belong to one of the classifications above,
it is architecturally invalid by default.

Do not invent new categories.
Do not blur boundaries.

Stop and escalate instead.

---

## Governing Principle

Truth is detected once.
Everything else consumes it.

Placement enforces meaning.



