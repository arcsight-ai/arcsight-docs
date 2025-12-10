# Invariant Enforcement Checker (CI Gate Specification)

## Final Version — Phase 1 & Phase 2

This document defines the CI enforcement layer that guarantees ArcSight remains:

- deterministic
- stable
- invariant-correct
- schema-correct
- drift-safe
- analyzer-version-safe

It is one of the core governance systems of the entire ArcSight platform.

---

## 1. Purpose

The Invariant Enforcement Checker ensures:

- every invariant in [Invariants Contract](./06-invariants-contract.md) is always enforced
- no code can weaken or bypass invariants
- engine determinism cannot regress
- schema and envelope correctness cannot drift
- forbidden behaviors are blocked at CI
- analyzer upgrades are always safe
- golden tests never silently degrade
- shadow analyzer correctness is preserved in Phase 2

If this gate fails, the analyzer MUST NOT ship.

---

## 2. Scope

This checker governs:

- invariant evaluation ([Invariants Contract](./06-invariants-contract.md))
- invariant test suite enforcement ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md))
- error code mapping ([Error Codes Catalog](./06B-error-codes-catalog.md))
- determinism guarantees ([Determinism Contract](./07-determinism-contract.md))
- schema correctness ([Envelope Format Spec](./15-envelope-format-spec.md))
- envelope correctness
- snapshot determinism ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- rulepack safety ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md), [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md))
- engine/runtime separation ([Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- config determinism ([Deterministic Config Contract](./20-deterministic-config-contract.md))
- limit/sandbox correctness ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))

**Applies to all packages:**

✔ arc-engine  
✔ arc-cli  
✔ arc-runtime  
✔ arc-sentinel  

**Does not apply to:**

✘ Dashboard UI logic  
✘ External GitHub failures  
✘ Business logic not affecting analysis correctness  

---

## 3. Definitions & Terms

**Invariant**  
A correctness rule defined in [Invariants Contract](./06-invariants-contract.md).

**Invariant Violation**  
A failed invariant, always producing a code defined in [Error Codes Catalog](./06B-error-codes-catalog.md).

**CI Gate**  
A pipeline stage that MUST fail and block merge on any violation.

**Enforcement Checker**  
The program + static analyzers that enforce all rules in this document.

---

## 4. Rules / Contract

### 4.1 Required Enforcement Categories

The Enforcement Checker MUST enforce all of the following.

#### 4.1.1 Invariant Evaluation ([Invariants Contract](./06-invariants-contract.md))

For every golden repo and selected random repos:

- run engine
- check all invariants
- if any invariant fails → CI FAIL
- invariant must map to error code in [Error Codes Catalog](./06B-error-codes-catalog.md)

**Coverage includes:**

- graph invariants
- cycle invariants
- ordering invariants
- extension namespace rules
- envelope shape requirements
- `identityChain` correctness
- deterministic signature behavior
- degraded mode rules
- schema versioning invariants

#### 4.1.2 Invariant Test Suite Enforcement ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md))

CI MUST verify:

- every invariant in [Invariants Contract](./06-invariants-contract.md) has a corresponding test
- no invariant test is missing
- no invariant test is skipped (`it.skip`, `xit`)
- no invariant test lost coverage
- new invariants require new tests

**This MUST block merging if:**

- invariant list grows → tests not added → CI FAIL
- invariant removed → test not removed → CI FAIL

#### 4.1.3 Error Code Enforcement ([Error Codes Catalog](./06B-error-codes-catalog.md))

For every invariant failure:

- error code MUST exist in [Error Codes Catalog](./06B-error-codes-catalog.md)
- no unknown codes permitted
- no code reuse with changed meaning
- no code renames unless `analyzerVersion` bumps
- no code containing forbidden terms ("ERROR", HTTP-style codes)

Error surface MUST remain stable long-term.

#### 4.1.4 Determinism Enforcement ([Determinism Contract](./07-determinism-contract.md))

CI MUST detect nondeterminism:

- run engine twice on same RepoSnapshot
- run across OS variants (Linux, macOS; Alpine optional)
- `stableStringify` outputs MUST match byte-for-byte

**CI MUST detect:**

- randomness (`Math.random`, crypto without seed)
- timestamps inside engine
- locale-dependent operations
- unordered maps
- unstable iterators
- nondeterministic truncation
- nondeterministic config merging

Violation → IMMEDIATE BLOCKER.

#### 4.1.5 Forbidden Import Enforcement

The checker MUST statically validate:

**Engine MAY NOT import:**

- `fs` (outside tests)
- network libraries
- timers
- environment access
- runtime
- CLI

**CLI MAY NOT import:**

- runtime

**Runtime MAY NOT import:**

- CLI
- engine internals

**Sentinel MAY NOT import:**

- runtime
- CLI

**Boundaries MUST be enforced using:**

- eslint
- project references
- static dependency graph checker

Any forbidden import → CI FAIL.

#### 4.1.6 Schema & Envelope Validation ([Envelope Format Spec](./15-envelope-format-spec.md))

The enforcement checker MUST validate:

- envelope shape
- required fields
- forbidding unknown top-level fields
- canonical ordering
- extensions namespace preservation
- `degraded_reason` enum correctness
- signature format (lowercase hex)
- `generated_at` excluded from signature

If envelope fails schema → FAIL.

#### 4.1.7 Rulepack Safety Enforcement

Rulepacks MUST NOT:

- access filesystem
- call network
- depend on runtime context
- use nondeterministic APIs
- mutate snapshots
- delete historical extension fields

Violation → CI FAIL.

#### 4.1.8 Limit Enforcement ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))

The CI gate MUST verify:

- sandbox limits are applied deterministically
- limit values do not change mid-run
- truncation performed after canonical ordering
- truncated outputs are deterministic
- degraded mode behavior is correct

Limit drift → BLOCKER.

#### 4.1.9 Snapshot Determinism ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))

**Explicit requirement (added refinement):**

CI MUST verify:

- identical repo input → identical snapshot
- snapshot hash remains stable
- no environment-derived values
- no PR metadata influence

If `snapshotHash` drifts → CI FAIL.

### 4.2 CI Behavior Requirements

CI MUST:

- block merge on ANY violation
- fail fast (stop pipeline on first blocker)
- print diagnostic report
- NEVER silence stderr/stdout during invariant evaluation
- NEVER auto-retry analysis with altered parameters

### 4.3 Required CI Pipeline Steps

#### 4.3.1 Step 1 — Lint + Import Boundaries

Detect:

- forbidden imports
- project reference violations
- rulepack boundary violations

#### 4.3.2 Step 2 — Determinism Runner

Run engine twice:

```
env1 = analyze(snapshot)
env2 = analyze(snapshot)
assert stableStringify(env1) === stableStringify(env2)
```

**Across:**

- Linux
- macOS
- Node LTS + Node latest

If drift → FAIL.

#### 4.3.3 Step 3 — Invariant Test Suite ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md))

Run:

- unit invariant tests
- integration invariant tests
- golden invariant tests

**Check:**

- coverage
- stability
- no skipped tests
- deterministic ordering

**Optional refinement included:**

Tests MUST NOT run in nondeterministic order.  
Parallel execution MUST NOT reorder golden tests.

#### 4.3.4 Step 4 — Golden Tests ([Golden Test Governance](./09-golden-test-governance.md))

Golden outputs MUST:

- remain unchanged unless `analyzerVersion` increments
- be byte-for-byte deterministic

Unexpected change → FAIL.

#### 4.3.5 Step 5 — Error Code Validation

Check:

- error codes match [Error Codes Catalog](./06B-error-codes-catalog.md) catalog
- new codes added require spec updates
- no unregistered codes exist

#### 4.3.6 Step 6 — Schema Enforcement ([Envelope Format Spec](./15-envelope-format-spec.md))

Reject:

- malformed envelopes
- unknown fields
- broken extension namespaces

#### 4.3.7 Step 7 — Forbidden Behavior Static Scan

Reject:

- locale-dependent operations
- randomness
- dynamic config merging
- partially applied config
- global mutable state

#### 4.3.8 Step 8 — Workspace Purity Check (Optional Refinement Added)

CI MUST verify:

- no unexpected file changes in `/packages`
- only allowed build artifacts appear in `dist/`
- no auto-generated drift

Unexpected changes → FAIL.

### 4.4 Required CI Outputs

CI MUST produce:

- invariant pass/fail report
- drift report
- error code mapping
- schema validation output
- forbidden import violations
- determinism comparison dumps
- rulepack policy audit

---

## 5. Determinism Impact

The Enforcement Checker is the foundation of:

- deterministic analysis ([Determinism Contract](./07-determinism-contract.md))
- reproducible golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- stable analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- accurate drift detection ([Drift Detection Contract](./19-drift-detection-contract.md))
- safe schema evolution ([Schema Evolution Contract](./11-schema-evolution-contract.md))

If the Enforcement Checker is bypassed or weakened:

- invariants can drift undetected
- determinism can regress
- golden tests become unreliable
- analyzer promotion becomes unsafe
- shadow analysis becomes invalid

Therefore: The Enforcement Checker MUST be comprehensive, deterministic, and non-bypassable.

---

## 6. Runtime / Engine Boundary Impact

**CI MUST:**

- enforce engine/runtime boundaries ([Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- reject code that violates import rules
- block runtime from influencing engine determinism
- prevent engine from accessing runtime context

**Engine MUST:**

- pass all invariant checks
- emit only registered error codes
- maintain determinism guarantees
- never bypass enforcement

**Runtime MUST:**

- never mutate envelopes
- never inject nondeterministic values
- never retry with altered config
- respect engine boundaries

---

## 7. Versioning Impact

- `analyzerVersion` MUST bump when:
  - any invariant behavior changes
  - any enforcement rule changes
  - any error code is added or removed

- `schemaVersion` MUST bump when:
  - envelope structure changes
  - extension namespace structure changes

The Enforcement Checker guarantees version stability and prevents semantic drift.

---

## 8. Testing Requirements

### 8.1 Phase Behavior

**Phase 1:**

- enforce engine invariants
- enforce deterministic snapshots
- forbid rulepack complexity

**Phase 2:**

Additionally:

- shadow analyzer comparisons
- drift classification ([Drift Detection Contract](./19-drift-detection-contract.md))
- analyzer promotion gating ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- enterprise namespace checks ([Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md))

### 8.2 Enforcement Test Coverage

The Enforcement Checker itself MUST be tested:

- all enforcement rules have corresponding tests
- CI gate failures are deterministic
- enforcement bypass attempts are detected
- error reporting is accurate

---

## 9. Cross-References

This document enforces:

- [Invariants Contract](./06-invariants-contract.md)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)
- [Error Codes Catalog](./06B-error-codes-catalog.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 10. Change Log (Append-Only)

**v1.2.0** — Added Documentation Consistency Monitor

- Added requirement for automatic documentation consistency monitoring after every push
- Integrated with GitHub Actions workflow (`.github/workflows/docs-consistency-monitor.yml`)
- Monitor validates cross-references, structure compliance, and file existence

**v1.1.0** — Final Version with Micro-Improvements

- Added snapshot hash drift enforcement
- Added explicit ban on suppressing stdout/stderr
- Added deterministic test execution requirement
- Added workspace purity enforcement
- Added locale-dependent enforcement rule

**v1.0.0** — Initial Release

- Full CI enforcement spec
- Integration of invariants, determinism, schema, limits, and error codes

