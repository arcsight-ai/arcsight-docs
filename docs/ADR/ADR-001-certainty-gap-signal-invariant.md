# ADR-001 — Certainty Gap Signal Invariant

## Status

Proposed

## Context

ArcSight’s Constitution defines its ontology:

- ArcSight exists to replace uncertainty about architectural impact with evidence.  
- ArcSight reports architectural certainty gaps.  
- There are exactly four canonical certainty gaps: **Structural**, **Semantic**, **Temporal**, **Epistemic**.  
- Any architectural signal must reduce exactly one certainty gap.  

To keep this ontology enforceable and mechanically checkable, the requirement that every architectural signal declare exactly one certainty gap MUST be treated as a core semantic contract, not an implementation detail. This ADR formalizes that requirement as an invariant that governs contracts and types.

## Decision

Every architectural signal in ArcSight MUST declare exactly one **CertaintyGap**.

- The allowed certainty gaps are: **Structural**, **Semantic**, **Temporal**, **Epistemic**.  
- **Zero** gaps is forbidden.  
- **Multiple** gaps for a single signal is forbidden.  
- **Inferred**, **implicit**, or **default** gaps are forbidden.  

This invariant is enforced at the **contract/type level**, not at the feature or implementation level:

- Contracts and types MUST require an explicit `CertaintyGap` for every architectural signal.  
- Engine enforcement of this invariant is mechanical and debate-free once adopted.  

This ADR is the single source of truth for the **CertaintyGap** invariant.

## Consequences

- `types.md` MUST define a `CertaintyGap` enum (Structural, Semantic, Temporal, Epistemic) and require it for all architectural signals.  
- All existing architectural signals MUST be assigned a single `CertaintyGap` value (for v1, these are expected to be **Structural**).  
- Any new architectural signal MUST declare exactly one `CertaintyGap` at the type/contract boundary before implementation.  
- Any attempt to introduce a signal without a declared `CertaintyGap`, with multiple gaps, or with an inferred/implicit gap MUST be treated as a contract violation.  
- Once wired into types and contracts, engine enforcement becomes a mechanical check (e.g., missing or invalid `CertaintyGap` values cause build/test failure).

## Non-Goals

- No change to analyzer behavior.  
- No new signals.  
- No heuristics, scoring, ranking, or explanation logic.  
- No changes to what users see in outputs.  
- No future-looking expansion of allowable certainty gaps.  
- No modification of existing runtime or CLI semantics.

This ADR exists only to unlock:

- `types.md` updates (adding `CertaintyGap` and wiring it into signal types).  
- Mechanical engine enforcement of the invariant.

## Governance

- Changes to the **CertaintyGap** ontology (adding, removing, or redefining gaps) require **founder approval**.  
- Changes to this invariant (e.g., allowing zero, multiple, or inferred gaps) require **founder approval**.  
- This ADR is the single source of truth for the CertaintyGap signal invariant.  
- Once adopted, all downstream contracts, types, and engine checks MUST align with this ADR; conflicts MUST be resolved in favor of this invariant.


