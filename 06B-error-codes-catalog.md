# Invariant Error Codes Catalog

## Stable Machine-Readable Error Surface for All Invariants

**Status: Final (Updated with Optional Enhancements A–D)**  
**Tier: Critical**  
**Applies to: Engine, CLI, Runtime, Sentinel, Dashboard**

---

## 1. Purpose

This document defines the canonical set of invariant violation error codes emitted by the ArcSight engine.

These codes:

- ensure deterministic handling of invariant failures
- provide Sentinel with a stable error surface
- formalize machine-readable semantics for drift detection
- enable schema adapters to preserve invariant failures
- make engine behavior testable and comparable across versions
- prevent semantic drift over time

These codes MUST remain stable forever unless:

- `analyzerVersion` bumps (mandatory)
- `schemaVersion` bumps (if envelope structure changes)

---

## 2. Scope

This catalog applies to errors produced during:

- RepoSnapshot validation
- Graph construction
- Cycle detection
- Envelope validation
- Rulepack execution
- Determinism checks
- IdentityChain construction
- Extension namespace validation

**Not included:**

- runtime errors (GitHub, rate limiting, tarball fetch issues)
- CI wrapper errors
- Sentinel-level comparison errors (optional future Doc #16B)

---

## 3. Definitions & Terms

**Error Code**  
A stable, machine-readable identifier for a specific invariant violation.

**Severity Level**  
Classification of error impact: BLOCKER, MAJOR, MINOR.

**BLOCKER**  
Invalid analysis output that MUST fail CI and block analyzer promotion.

**MAJOR**  
Analyzer correct, but rulepack or extension changed unexpectedly.

**MINOR**  
Cosmetic, metadata-level shifts (rare and extension-only).

---

## 4. Rules / Contract

### 4.1 Error Code Structure

All invariant error codes MUST follow:

```
CATEGORY_DESCRIPTOR
```

Where:

- `CATEGORY` ∈ { `SNAPSHOT`, `GRAPH`, `CYCLE`, `ENVELOPE`, `IDENTITY`, `RULEPACK`, `EXTENSION`, `DETERMINISM` }
- `DESCRIPTOR` = `SCREAMING_SNAKE_CASE` describing violation

### 🔥 Updated Rule (Enhancement B):

**Error codes MUST NOT include the word "ERROR" or use numeric/HTTP-style suffixes.**

This ensures codes remain semantic, stable, and machine-readable.

**Examples:**

- `GRAPH_UNEXPECTED_CYCLE`
- `ENVELOPE_TIMESTAMP_DRIFT`
- `DETERMINISM_LOCALE_DRIFT`

Codes MUST:

- remain OS-independent
- remain Node-version-independent
- be JSON-serializable
- remain stable for all future versions

### 4.2 Severity Levels

Sentinel uses severity levels to classify drift:

**BLOCKER**

- Always indicates invalid analysis output
- MUST fail CI
- MUST block analyzer promotion
- MUST cause shadow-vs-live BLOCKER drift

**MAJOR**

- Analyzer correct, but rulepack or extension changed unexpectedly

**MINOR**

- Cosmetic, metadata-level shifts
- Rare and extension-only

06B focuses on BLOCKER errors (the majority).

### 4.3 Reserved Prefixes (Enhancement C)

These prefixes are reserved for future non-engine layers:

| Prefix | Intended For |
|--------|--------------|
| `SENTINEL_*` | Shadow/live comparison failures |
| `PROMOTION_*` | Analyzer promotion pipeline |
| `SYSTEM_*` | Bootstrapping and infrastructure issues |

Engine MUST NOT emit error codes with these prefixes.

### 4.4 Interaction with Test Suite ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md))

Every error code MUST have:

- a unit test
- an integration test
- an invariant test
- drift classification tests (shadow vs live)

If a new invariant is added:

- CI MUST FAIL until a corresponding test is added.

This is how ArcSight avoids hidden invariant regressions.

### 4.5 Schema Evolution Rules

Schema adapters MUST:

- preserve error nodes
- NEVER rename codes
- NEVER delete codes
- NEVER alter severity

If an invariant is removed:

- `analyzerVersion` MUST bump
- error code MUST be marked deprecated but NOT removed

---

## 5. Error Code Catalog

### 5.1 RepoSnapshot Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `SNAPSHOT_PATH_NORMALIZATION_FAILED` | Path failed canonical normalization | BLOCKER |
| `SNAPSHOT_DUPLICATE_PATH` | Two files resolved to same normalized path | BLOCKER |
| `SNAPSHOT_INVALID_ENCODING` | File is not valid UTF-8 | BLOCKER |
| `SNAPSHOT_UNSTABLE_ORDER` | Snapshot ordering not deterministic | BLOCKER |
| `SNAPSHOT_METADATA_FORBIDDEN` | Snapshot contained fields outside contract | BLOCKER |
| `SNAPSHOT_FILE_TOO_LARGE` | File exceeded per-file size limit | BLOCKER |
| `SNAPSHOT_TOO_MANY_FILES` | Snapshot exceeded allowed file count | BLOCKER |
| `SNAPSHOT_HASH_DRIFT` | Same snapshot produced different hash | BLOCKER |

### 5.2 Graph Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `GRAPH_MISSING_NODE` | Graph node missing | BLOCKER |
| `GRAPH_INVALID_EDGE` | Edge references nonexistent node | BLOCKER |
| `GRAPH_UNEXPECTED_CYCLE` | Cycle exists pre-rulepack | BLOCKER |
| `GRAPH_NODE_SHAPE_INVALID` | Node shape violates schema | BLOCKER |
| `GRAPH_STATS_DRIFT` | Node/edge counts inconsistent with snapshot | BLOCKER |
| `GRAPH_SORT_DRIFT` | Graph ordering nondeterministic | BLOCKER |

### 5.3 Cycle Detection Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `CYCLE_CLASSIFICATION_DRIFT` | Cycle classification changed unexpectedly | BLOCKER |
| `CYCLE_FALSE_POSITIVE` | Engine detected nonexistent cycle | BLOCKER |
| `CYCLE_FALSE_NEGATIVE` | Engine missed a real cycle | BLOCKER |
| `CYCLE_NONDETERMINISTIC_ORDER` | Cycle ordering unstable | BLOCKER |

### 5.4 Envelope Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `ENVELOPE_VERSION_MISSING` | `analyzerVersion` missing | BLOCKER |
| `ENVELOPE_SCHEMA_MISSING` | `schemaVersion` missing | BLOCKER |
| `ENVELOPE_UNKNOWN_FIELD` | Envelope emitted unrecognized fields | BLOCKER |
| `ENVELOPE_IDENTITY_DRIFT` | `identityChain` unstable | BLOCKER |
| `ENVELOPE_TIMESTAMP_DRIFT` | Timestamp nondeterministic | BLOCKER |
| `ENVELOPE_EXTENSION_SHAPE_INVALID` | Extension namespace structure invalid | BLOCKER |
| `ENVELOPE_RULEPACK_NAMESPACE_FORBIDDEN` | Unauthorized namespace used | BLOCKER |
| `ENVELOPE_INVARIANT_VIOLATION` | Generic invariant failure fallback | BLOCKER |
| `ENVELOPE_SIGNATURE_INVALID` | Envelope signature malformed or unverifiable | BLOCKER |

**🔥 Enhancement D (added):**

`ENVELOPE_SIGNATURE_INVALID` makes signature errors first-class and drift-detectable.

### 5.5 Identity Chain Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `IDENTITY_MISSING_CHECKSUM` | Checksum missing | BLOCKER |
| `IDENTITY_STABLE_HASH_DRIFT` | Hash unstable across runs | BLOCKER |
| `IDENTITY_CHAIN_OUT_OF_ORDER` | Identity steps misordered | BLOCKER |

### 5.6 Determinism Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `DETERMINISM_LOCALE_DRIFT` | Locale-dependent operation used | BLOCKER |
| `DETERMINISM_RANDOMNESS` | Random value found | BLOCKER |
| `DETERMINISM_TIME_DEPENDENCE` | Clock accessed in deterministic context | BLOCKER |
| `DETERMINISM_UNSTABLE_ORDER` | Sorting nondeterministic | BLOCKER |
| `DETERMINISM_CONFIG_PARTIAL` | Config partially applied or merged dynamically | BLOCKER |
| `DETERMINISM_ENVIRONMENT_DEPENDENCE` | Env var changed engine behavior | BLOCKER |
| `DETERMINISM_GLOBAL_STATE` | Global mutable state detected | BLOCKER |

### 5.7 Rulepack Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `RULEPACK_SCHEMA_VIOLATION` | Rulepack emitted invalid schema | BLOCKER |
| `RULEPACK_IMPORT_FORBIDDEN` | Rulepack used forbidden library | BLOCKER |
| `RULEPACK_DETERMINISM_DRIFT` | Rulepack produced nondeterministic result | BLOCKER |
| `RULEPACK_VERSION_DRIFT` | Rulepack behavior inconsistent with version | BLOCKER |

### 5.8 Extension Namespace Error Codes

| Code | Meaning | Severity |
|------|---------|----------|
| `EXTENSION_MISSING_NAMESPACE` | Missing required extension namespace | BLOCKER |
| `EXTENSION_UNSTABLE_FIELDS` | Extension output nondeterministic | BLOCKER |
| `EXTENSION_UNKNOWN_FIELDS` | Extension emitted unknown fields | BLOCKER |
| `EXTENSION_DELETION_FORBIDDEN` | Extension fields removed across versions | BLOCKER |

---

## 6. Determinism Impact

Error codes are the foundation of:

- deterministic error handling ([Determinism Contract](./07-determinism-contract.md))
- stable drift detection ([Drift Detection Contract](./19-drift-detection-contract.md))
- safe analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- reproducible golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- schema adapter correctness ([Schema Evolution Contract](./11-schema-evolution-contract.md))

If error codes are unstable or missing:

- drift detection becomes unreliable
- analyzer promotion becomes unsafe
- golden tests become meaningless
- schema evolution becomes unpredictable

Therefore: Error codes MUST be stable, comprehensive, and machine-readable.

---

## 7. Runtime / Engine Boundary Impact

**Engine MUST:**

- emit only codes defined in this catalog
- use consistent severity levels
- never invent new codes without `analyzerVersion` bump
- preserve error codes through schema adapters

**Runtime MUST:**

- reject envelopes with unknown error codes
- never mutate error codes
- never hide error codes
- pass error codes to Sentinel unchanged

**Sentinel MUST:**

- classify BLOCKER errors as BLOCKER drift
- treat unknown error codes as BLOCKER
- never downgrade error severity
- preserve error codes in drift reports

---

## 8. Versioning Impact

- `analyzerVersion` MUST bump when:
  - any invariant behavior changes
  - any error code meaning changes
  - any error code is added or removed

- `schemaVersion` MUST bump when:
  - envelope structure changes
  - extension namespace structure changes

Error codes guarantee version stability and prevent semantic drift.

---

## 9. Testing Requirements

### 9.1 Sentinel Drift Semantics

Sentinel MUST classify as:

**BLOCKER drift if:**

- invariant violated
- error code unexpected or missing
- error code meaning changes
- new error code added without `analyzerVersion` bump

**MAJOR drift if:**

- rulepack behavior shifts while invariants hold

**MINOR drift if:**

- metadata-only extension changes occur

### 9.2 Error Code Test Coverage

Every error code MUST have:

- unit test that triggers the error
- integration test that validates error handling
- invariant test that verifies error stability
- drift classification test (shadow vs live)

---

## 10. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 11. Change Log (Append-Only)

**v1.1.0** — Final Updated Version

- Added `ENVELOPE_SIGNATURE_INVALID`
- Added reserved error code prefix section
- Added explicit rule forbidding "ERROR" and numeric codes
- Clarified drift semantics and severity rules
- Synced with [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md) and [Deterministic Config Contract](./20-deterministic-config-contract.md)

**v1.0.0** — Initial published version

