# Drift Detection Contract (FINAL UPDATED, Phase 2)

## 1. Purpose

This document defines the deterministic, semantic, and structural rules governing drift detection between analyzer versions.

Drift detection ensures that new analyzers:

- do not regress behavior
- preserve determinism
- maintain envelope correctness
- evolve safely across versions
- respect rulepack, schema, and extension invariants

Drift detection is performed exclusively by Sentinel, which evaluates live vs. shadow analyzer results.

This contract guarantees that analyzer promotion remains safe, deterministic, and auditable.

---

## 2. Scope

### Included

This contract specifies:

- drift categories (benign, warning, blocker)
- drift comparison rules
- deterministic ordering and normalization
- rules for core, rulepack, and extension drift
- preservation of unknown extension fields
- invariant pre-checks
- cycle/graph drift semantics
- shadow vs. live comparison rules
- block conditions

### Excluded

This contract does NOT define:

- analyzer promotion lifecycle → see [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- schema evolution rules → [Schema Evolution Contract](./11-schema-evolution-contract.md)
- envelope format → [Envelope Format Spec](./15-envelope-format-spec.md)
- rulepack versioning → [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- shadow analyzer execution rules → [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- invariants → [Invariants Contract](./06-invariants-contract.md)
- deterministic behavior in general → [Determinism Contract](./07-determinism-contract.md)

---

## 3. Definitions & Terms

**Drift**  
Any difference between the normalized envelopes of two analyzer versions.

**Drift Unit**  
A drift comparison performed on schema-upgraded, invariant-validated envelopes.

**Live Analyzer**  
Current production `analyzerVersion`.

**Shadow Analyzer**  
Candidate `analyzerVersion` used for drift evaluation.

**Benign Drift**  
Safe, non-semantic differences.

**Warning Drift**  
Differences requiring human review, but not automatically blocking.

**Blocker Drift**  
Any change that MUST stop analyzer promotion.

---

## 4. Rules / Contract

### 4.1 Drift MUST be computed on valid, schema-normalized envelopes

Before ANY drift comparison:

1. Apply schema adapters to both envelopes → produce `CurrentSchemaEnvelope`.
2. Run all invariants from [Invariants Contract](./06-invariants-contract.md).
3. If either envelope fails invariants → Sentinel MUST classify as BLOCKER drift immediately.
4. Normalize ordering using `stableStringify`.
5. Strip nondeterministic fields (`meta.generated_at` only).

**Comparison MUST NOT occur before invariant validation.**

### 4.2 Drift Classification Categories

Drift MUST be classified into exactly one of:

#### Category 1 — Benign Drift (Allowed)

Benign drift includes:

- new optional extension fields under known rulepack namespaces
- deterministic removal of cycles where cycles truly disappeared
- deterministic improvements in canonicalization logic
- reduction in graph size if structurally correct
- additional rulepacks introduced for the first time
- strictly additive extension fields

Benign drift MUST NOT block promotion.

#### Category 2 — Warning Drift (Requires Manual Review)

Warning drift includes:

- increased cycle count caused by legitimately discovered cycles
- changes in extension values that are meaningful but backward-compatible
- rulepack evolution with proper versioning
- new degraded → success transitions or vice-versa that appear legitimate
- addition of new rulepacks (first appearance only)

Warning drift MAY be approved for promotion but MUST NOT be auto-promoted.

#### Category 3 — Blocker Drift (MUST STOP Promotion)

Blocker drift includes:

**Structural Drift**

- any violation of envelope spec ([Envelope Format Spec](./15-envelope-format-spec.md))
- required fields missing
- `schemaVersion` mismatch without adapter
- malformed extension namespaces
- broken ordering or canonicalization

**Semantic Drift**

- new cycles appearing without an explicitly documented `analyzerVersion` change to cycle detection logic
- cycles missing unexpectedly
- inconsistent `graph_stats`
- shadow produces degraded/error while live succeeds
- shadow produces different `degraded_reason` unexpectedly

**Rulepack Drift**

- extension fields removed without major `rulepackVersion` bump
- changes to meaning of existing extension fields
- nondeterministic extension output
- namespace collisions
- violation of enterprise namespace rules ([Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md))

**Determinism Drift**

- different signatures for identical inputs
- nondeterministic ordering
- nondeterministic hashing
- floating-point instability
- nondeterministic truncation behavior

**Snapshot Drift**

- shadow analyzer interprets RepoSnapshot differently
- inconsistent path normalization
- different import resolution logic without justification

Any Blocker Drift MUST halt promotion immediately.

### 4.3 Drift Comparison MUST Preserve Unknown Extension Fields

To support enterprise rulepacks ([Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)):

- Drift detection MUST NOT remove unknown extension namespaces.
- Drift detection MUST NOT rename or collapse unknown extension fields.
- Unknown extension fields MUST appear identically in both normalized envelopes.
- Unknown extension drift is only compared structurally, not semantically.

This ensures enterprise and vendor rulepacks evolve safely and independently.

### 4.4 Allowed Drift Rules (Detailed)

Allowed drift includes:

- additive extension fields
- deterministic canonicalization improvements
- deterministic graph changes documented in `analyzerVersion` release notes
- rulepack minor updates with version bump
- new rulepacks appearing under `extensions.*`

Allowed drift MUST be:

- deterministic
- stable across environments
- documented in version notes
- backward-compatible

### 4.5 Forbidden Drift Rules (Detailed)

Forbidden drift includes:

- ANY nondeterministic behavior
- ANY invariant failure ([Invariants Contract](./06-invariants-contract.md))
- ANY structural schema break ([Envelope Format Spec](./15-envelope-format-spec.md))
- deleting extension fields
- changing extension field semantics without version bump
- producing new cycles without documented cycle-detector change
- runtime-influenced outputs
- fallback or heuristic analysis ([Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- environment-based variation

Forbidden drift → automatic BLOCKER.

---

## 5. Determinism Impact

Drift detection is the foundation of:

- analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- rulepack evolution ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- deterministic shadow analysis ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- envelope stability ([Envelope Format Spec](./15-envelope-format-spec.md))
- schema adapter correctness ([Schema Evolution Contract](./11-schema-evolution-contract.md))

If drift detection becomes nondeterministic:

- promotions become unsafe
- regressions go undetected
- rulepacks drift unpredictably
- shadow analysis becomes invalid
- enterprise integrations break

Therefore: Drift evaluation MUST be 100% deterministic, auditable, and reproducible.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- supply identical RepoSnapshots to live + shadow
- NEVER modify envelopes before drift comparison
- NEVER retry analyses
- NEVER hide or compress drift
- NEVER apply fallbacks or heuristics

**Engine MUST:**

- produce deterministic envelopes
- adhere to all invariants ([Invariants Contract](./06-invariants-contract.md))
- operate under fixed limits ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))
- never depend on environment state

**Sentinel MUST:**

- classify drift deterministically
- block promotion on BLOCKER drift
- preserve unknown extension fields
- validate invariants before computing drift

---

## 7. Versioning Impact

- `analyzerVersion` MUST bump when:
  - deterministic output changes
  - core graph/cycle logic changes
  - rulepack semantics change
  - truncation behavior changes

- `rulepackVersion` MUST bump when:
  - extension semantics change ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))

- `schemaVersion` MUST bump when:
  - envelope structure changes ([Schema Evolution Contract](./11-schema-evolution-contract.md))

- `snapshotVersion` MUST bump when:
  - snapshot format changes ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))

Drift detection enforces correctness of all these version boundaries.

---

## 8. Testing Requirements

### 8.1 Drift Classification Tests

Tests MUST validate:

- benign drift
- warning drift
- blocker drift
- invariant failure → blocker classification

### 8.2 Cross-Environment Determinism Tests

Drift detection MUST produce identical results across:

- Linux
- macOS
- all supported Node LTS versions

### 8.3 Schema Adapter Drift Tests

Adapters MUST NOT introduce drift beyond the expected schema transformation.

### 8.4 Regression Drift Tests

CI MUST ensure:

- expected drift matches golden drift baselines
- no unexpected structural or semantic drift

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added enhancements:

- Invariant validation required before drift detection
- New cycles require documented `analyzerVersion` change
- Unknown extension field preservation required
- Expanded forbidden drift section
- Clarified drift normalization pipeline

**v1.0.0** — Initial drift detection contract

