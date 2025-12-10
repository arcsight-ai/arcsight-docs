# Cursor Development Protocol

## Operational Specification for AI-Assisted Development in ArcSight

**State: Active**  
**Tier: C (Operational)**  
**Enforces:** [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md), [Invariants Contract](./06-invariants-contract.md), [Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md), [Determinism Contract](./07-determinism-contract.md), [Golden Test Governance](./09-golden-test-governance.md), [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md), [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md), [RepoSnapshot Contract](./14-repo-snapshot-contract.md), [Envelope Format Spec](./15-envelope-format-spec.md), [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md), [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md), [Drift Detection Contract](./19-drift-detection-contract.md), [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 1. Purpose

This document defines the strict rules that govern how Cursor (AI coding assistant) MUST operate within the ArcSight project.

Cursor MUST:

- maintain determinism
- enforce invariants
- preserve schema stability
- protect analyzer boundaries
- avoid introducing undocumented or nondeterministic behavior
- ensure all changes remain aligned with the architecture & contract suite

This protocol ensures Cursor accelerates development without breaking ArcSight's correctness model.

---

## 2. Scope

**Applies to Cursor's actions in:**

✔ Engine code generation  
✔ Runtime code generation  
✔ CLI tools  
✔ Sentinel code  
✔ Schema evolution  
✔ Rulepack logic  
✔ Documentation editing  
✔ Golden test updates  
✔ Refactors  
✔ Analyzer promotion work  

**Not in scope:**

✘ Human-written code  
✘ Business/architecture decision-making  
✘ Enterprise rulepacks (external vendors)  

---

## 3. Definitions

**Cursor**  
AI developer operating under this protocol.

**Human Author**  
Developer requesting an action.

**Deterministic Surface**  
Any behavior protected by [Determinism Contract](./07-determinism-contract.md) & [Invariants Contract](./06-invariants-contract.md).

**Contract Documents (Tier B)**  
Documents enforcing required structure ([Invariants Contract](./06-invariants-contract.md), [Determinism Contract](./07-determinism-contract.md), [Schema Evolution Contract](./11-schema-evolution-contract.md), [RepoSnapshot Contract](./14-repo-snapshot-contract.md), etc).

**Operational Documents (Tier C)**  
Documents describing workflows, processes, protocols ([Cursor Development Protocol](./10C-cursor-development-protocol.md)).

---

## 4. Global Behavioral Rules

### 4.1 Cursor MUST NOT:

❌ Invent new invariants  
❌ Invent new schema fields / envelope shapes  
❌ Invent new rulepacks  
❌ Invent new analyzer behavior  
❌ Modify `analyzerVersion` or `schemaVersion` unprompted  
❌ Update golden tests unless explicitly requested  
❌ Change contract documents without maintaining all required sections  
❌ Introduce nondeterministic behavior (`Date.now`, random, locale ops, unstable Maps)  
❌ Reorganize directory structure unless explicitly instructed  
❌ Move snapshot builder logic prematurely  
❌ Create new package boundaries (e.g., `arc-shared`) without explicit instruction  
❌ Overwrite or delete change logs  
❌ Auto-promote analyzers  
❌ Introduce new config parameters without [Deterministic Config Contract](./20-deterministic-config-contract.md) changes  

### 4.2 Cursor MUST:

✔ Follow [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md) authoring rules  
✔ Maintain cross-document consistency  
✔ Respect dependency graph ([Dependency Contract](./5A-dependency-contract.md))  
✔ Enforce invariants ([Invariants Contract](./06-invariants-contract.md))  
✔ Respect deterministic rules ([Determinism Contract](./07-determinism-contract.md))  
✔ Ensure snapshot purity ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))  
✔ Preserve envelope structure ([Envelope Format Spec](./15-envelope-format-spec.md))  
✔ Maintain rulepack versioning rules ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))  
✔ Follow drift & promotion rules ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md), [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md), [Drift Detection Contract](./19-drift-detection-contract.md))  
✔ Validate changes against ALL contract docs  

---

## 5. Cursor Rules for Code Generation

### 5.1 Engine (arc-engine)

**Cursor MUST:**

- generate pure deterministic code
- use no environment, no randomness, no timers
- maintain canonical orderings & stable hashing
- reference exact document sections when modifying sensitive areas

e.g., "Following [Full System Roadmap](./03-full-system-roadmap.md) §5.1, cycle detection must remain deterministic…"

**Cursor MUST NOT:**

- alter cycle detection semantics unless tied to [Full System Roadmap](./03-full-system-roadmap.md) or [Determinism Contract](./07-determinism-contract.md)
- modify `schemaVersion`, `analyzerVersion`
- change Envelope structure without explicit schema evolution request
- introduce new extension namespaces

### 5.2 Runtime (arc-runtime)

**Cursor MUST:**

- treat engine as black box
- build snapshots deterministically
- follow [RepoSnapshot Contract](./14-repo-snapshot-contract.md) snapshot rules
- avoid PR metadata influence ([Deterministic Config Contract](./20-deterministic-config-contract.md))

**Cursor MUST NOT:**

- mutate envelopes
- retry analysis with altered limits
- re-run engine in fallback mode
- derive dynamic config from runtime context

### 5.3 CLI (arc-cli)

**Cursor MUST:**

- only use public engine API
- preserve deterministic golden diffing

**Cursor MUST NOT:**

- interact with GitHub APIs
- modify analyzer output

### 5.4 Sentinel (arc-sentinel)

**Cursor MUST:**

- enforce rules in [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- perform drift detection deterministically
- preserve drift classifications

**Cursor MUST NOT:**

- modify envelopes
- create fallback branches
- change analyzer behavior

---

## 6. Cursor Rules for Documentation Editing

### 6.1 Cursor MUST:

✔ Maintain all 10 required contract sections  
✔ Self-update references when filenames or numbering changes  
✔ Cite document sections explicitly when making architectural or engine-impacting changes  

**Example:**

"As required by [Schema Evolution Contract](./11-schema-evolution-contract.md) §4.3, extension fields must remain backwards compatible."

**Cursor MUST verify:**

- schema rules ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- adapter rules ([Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md))
- invariants ([Invariants Contract](./06-invariants-contract.md))
- drift rules ([Drift Detection Contract](./19-drift-detection-contract.md))
- analyzer promotion rules ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))

### 6.2 Cursor MUST NOT:

❌ Invent new docs unless explicitly asked  
❌ Change doc numbering  
❌ Remove or alter past change logs  

---

## 7. Cursor Rules for Golden Test Updates

Golden tests MUST NOT be modified unless explicitly asked.

**When permitted:**

- updates MUST follow [Golden Test Governance](./09-golden-test-governance.md)
- updates MUST be deterministic & canonical
- engine behavior drift must be intentional
- `analyzerVersion` bump must be validated

**Cursor MUST check:**

- invariants still pass
- drift classification expectations remain unchanged

---

## 8. Cursor Rules for Schema Evolution

Cursor MUST NOT evolve schema unless explicitly instructed.

**When requested:**

- MUST follow [Schema Evolution Contract](./11-schema-evolution-contract.md) & [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- MUST create full adapter chain `vN → vN+1`
- MUST update envelope spec ([Envelope Format Spec](./15-envelope-format-spec.md))
- MUST ensure backwards compatibility
- MUST regenerate type definitions in engine/CLI/shared

---

## 9. Cursor Rules for Analyzer Promotion & Drift

Cursor MUST enforce all rules from [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md) and [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md):

- drift calculation is deterministic
- shadow & live must use identical config hashes
- snapshot & envelope must not be mutated
- promotion requires human approval

**Cursor MUST NOT:**

- auto-promote
- alter drift logic
- silence drift mismatches

---

## 10. Cursor Refactor Protocol (NEW)

Before applying any refactor, Cursor MUST evaluate whether the change affects:

✔ deterministic surfaces ([Determinism Contract](./07-determinism-contract.md))  
✔ invariants ([Invariants Contract](./06-invariants-contract.md))  
✔ envelope shape ([Envelope Format Spec](./15-envelope-format-spec.md))  
✔ snapshot purity ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))  
✔ schema adapters ([Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md))  
✔ `analyzerVersion` stability ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))  
✔ golden tests ([Golden Test Governance](./09-golden-test-governance.md))  

**If any surface is affected:**

Cursor MUST warn:

"This refactor affects deterministic surfaces or contract boundaries. Human approval is required per [Cursor Development Protocol](./10C-cursor-development-protocol.md)."

---

## 11. Cursor File Structure Protection Rules (NEW)

**Cursor MUST NOT:**

- reorganize directories
- move Phase-1 snapshot builder out of wedge prematurely
- introduce `arc-shared` before Phase 2 monorepo migration
- modify package boundaries unless explicitly instructed

This prevents premature architecture drift.

---

## 12. Cursor Contract Compliance Self-Check (NEW Required Section)

Before producing any code, Cursor MUST self-validate that:

✔ It respects dependency rules ([Dependency Contract](./5A-dependency-contract.md))  
✔ It satisfies invariants ([Invariants Contract](./06-invariants-contract.md))  
✔ It does not introduce nondeterminism ([Determinism Contract](./07-determinism-contract.md))  
✔ It respects envelope spec ([Envelope Format Spec](./15-envelope-format-spec.md))  
✔ It does not cause drift mismatches ([Drift Detection Contract](./19-drift-detection-contract.md))  
✔ It does not mutate snapshots or envelopes post-build ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))  
✔ It does not alter user-visible behavior without version bumps  

**If any violation is detected, Cursor MUST respond:**

"This request violates ArcSight's deterministic or architectural rules. I cannot complete it without explicit instruction citing the governing document."

---

## 13. Cursor Cross-Document Citation Requirement (NEW)

When making changes affecting architecture, engine behavior, schema, or drift detection, Cursor MUST reference:

- the exact doc
- the exact section
- the exact rule

**Example:**

"Updating cycle detection requires compliance with [Full System Roadmap](./03-full-system-roadmap.md) §5.1 and [Determinism Contract](./07-determinism-contract.md) §4.2. This change modifies a deterministic surface and may require `analyzerVersion` bump per [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md) §4.1."

This prevents silent semantic shifts.

---

## 14. Change Log (Append-Only)

**v1.1.0** — Added Micro-Enhancements

- Added explicit cross-document citation requirement
- Added file structure protection rules
- Added refactor protocol
- Added pre-output compliance self-check
- Added explicit "must not reorganize packages" rule

**v1.0.0** — Initial Release

- Full Cursor governance specification
- Defined deterministic coding guardrails
- Defined documentation compliance rules
- Golden tests + schema evolution protections
- Analyzer promotion protections

