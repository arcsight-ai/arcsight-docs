# Invariant Test Suite Specification (FINAL Phase 2+)

## ArcSight Engine Invariant Testing Protocol

**Status: Final**  
**Tier: Critical**  
**Applies to: Phase 1 (late), fully enforced in Phase 2+**

---

## 1. Purpose

This document defines the formal, executable test suite required to validate all invariants defined in:

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)

It ensures:

- invariants NEVER drift
- invariant failures always produce deterministic, stable error codes
- golden tests never hide invariant violations
- shadow analyzer drift detection is accurate
- analyzer promotion remains safe and predictable

This is the authoritative testing specification for ArcSight's invariant system.

---

## 2. Scope

This specification applies to:

✔ **arc-engine**  
(all invariants)

✔ **arc-cli**  
(executes invariant suites; prints failures)

✔ **arc-sentinel**  
(relies on invariant stability to classify drift)

✔ **arc-runtime**  
(must reject invalid envelopes)

✔ **arc-shared**  
(type stability for invariants + error codes)

**Not included:**

❌ Runtime logic  
❌ Dashboard / UI tests  
❌ Rulepack semantics (covered in [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))

---

## 3. Definitions & Terms

**Invariant**  
A condition that must always hold for every valid snapshot, graph, cycle set, envelope, or deterministic process.

**Invariant Category**  
Snapshot, Graph, Cycle, Envelope, Extension, Determinism.

**Invariant Test**  
A deterministic automated test verifying that one invariant always holds and failures are stable.

**Invariant Registry**  
A complete list of invariants with mapped test locations and error codes.

---

## 4. Rules / Contract

### 4.1 Required Invariant Test Layers

Each invariant MUST be tested at least in:

- Unit tests
- Integration tests
- Golden tests (where envelope- or cycle-related)
- Shadow analyzer drift tests (Phase 2 only)

These layers guarantee correctness, determinism, and stability.

### 4.2 Snapshot Invariant Tests

Ensures RepoSnapshot follows:

- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) (snapshot spec)
- Snapshot invariants from [Invariants Contract](./06-invariants-contract.md)

**Required tests include:**

| Invariant | Unit Test | Integration Test | Error Code |
|-----------|-----------|------------------|------------|
| Path normalization | ✔ | ✔ | `SNAPSHOT_PATH_NORMALIZATION_FAILED` |
| No duplicate paths | ✔ | ✔ | `SNAPSHOT_DUPLICATE_PATH` |
| UTF-8 encoding only | ✔ | ✔ | `SNAPSHOT_INVALID_ENCODING` |
| Sorted canonical order | ✔ | ✔ | `SNAPSHOT_UNSTABLE_ORDER` |
| No extra metadata | ✔ | ✔ | `SNAPSHOT_METADATA_FORBIDDEN` |

### 4.3 Graph Invariant Tests

Tests defined in [Invariants Contract](./06-invariants-contract.md) (Graph section).

| Invariant | Unit | Integration | Error Code |
|-----------|-----|-------------|------------|
| All nodes included | ✔ | ✔ | `GRAPH_MISSING_NODE` |
| No unexpected cycles | ✔ | ✔ | `GRAPH_UNEXPECTED_CYCLE` |
| Valid adjacency | ✔ | ✔ | `GRAPH_INVALID_EDGE` |
| Stable graph statistics | ✔ | ✔ | `GRAPH_STATS_DRIFT` |

### 4.4 Cycle Detection Invariant Tests

Tests for the cycle rulepack.

| Invariant | Test Layers | Error Code |
|-----------|-------------|------------|
| Deterministic cycle ordering | unit + golden | `CYCLE_CLASSIFICATION_DRIFT` |
| No false positives | unit | `CYCLE_FALSE_POSITIVE` |
| No false negatives | unit | `CYCLE_FALSE_NEGATIVE` |

### 4.5 Envelope Invariant Tests

Based on:

- [Envelope Format Spec](./15-envelope-format-spec.md) (Envelope Spec)
- [Invariants Contract](./06-invariants-contract.md) (Envelope Invariants)

**Required tests:**

| Invariant | Tests | Error Code |
|-----------|-------|------------|
| `analyzerVersion` present | unit | `ENVELOPE_VERSION_MISSING` |
| `schemaVersion` valid | unit | `ENVELOPE_SCHEMA_MISSING` |
| Identity chain deterministic | golden | `ENVELOPE_IDENTITY_DRIFT` |
| Canonical timestamp normalization | integration | `ENVELOPE_TIMESTAMP_DRIFT` |
| No unknown fields | unit | `ENVELOPE_UNKNOWN_FIELD` |

Golden snapshots MUST be identical across platforms and Node versions.

### 4.6 Rulepack Invariant Tests

Even in Phase 1 cycle-only rulepack:

| Invariant | Tests | Error Code |
|-----------|-------|------------|
| deterministic output | golden | `RULEPACK_DETERMINISM_DRIFT` |
| schema-safe extensions | unit | `RULEPACK_SCHEMA_VIOLATION` |
| no forbidden imports | CI | `RULEPACK_IMPORT_FORBIDDEN` |

### 4.7 Determinism Invariant Tests

Imported from:

- [Determinism Contract](./07-determinism-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

Tests MUST enforce:

| Invariant | Error Code |
|-----------|------------|
| No locale-dependent behavior | `DETERMINISM_LOCALE_DRIFT` |
| No randomness | `DETERMINISM_RANDOMNESS` |
| No clock-dependent behavior | `DETERMINISM_TIME_DEPENDENCE` |
| Stable ordering | `DETERMINISM_UNSTABLE_ORDER` |
| No dynamic or partially-applied config | `DETERMINISM_CONFIG_PARTIAL` |

### 4.8 Golden Invariant Tests

Golden tests verify:

- stable envelope structure
- stable identity chains
- stable cycle classification
- stable extension fields
- stable signature

Any golden drift requires:

- `analyzerVersion` bump
- updated golden expectations
- documented reason

Golden tests are REQUIRED for:

- all envelope invariants
- all cycle invariants
- all signature invariants

### 4.9 CI Enforcement Requirements

CI MUST fail if:

- ANY invariant test fails
- ANY invariant lacks a test
- Expected error code does not match [Error Codes Specification](./06B-error-codes-specification.md)
- A new invariant is added without a corresponding test (🔒 newly added rule)
- Golden invariant files drift without `analyzerVersion` bump
- Tests pass on one OS but fail on another (🔒 OS cross-run enforcement)

CI MUST run on:

- Linux
- macOS
- (Optional but recommended) Alpine

This prevents locale, filesystem, and ordering nondeterminism.

### 4.10 OS-Cross-Run Requirements

Invariant tests and golden tests MUST run identically across:

- Linux (CI default)
- macOS (developer + CI secondary runner)
- Alpine (optional but recommended for busybox filesystems)

Differences MUST surface as BLOCKER drift.

---

## 5. Determinism Impact

Invariant tests are the foundation of:

- deterministic analysis ([Determinism Contract](./07-determinism-contract.md))
- reproducible golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- stable analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- shadow-mode comparisons ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- accurate drift detection ([Drift Detection Contract](./19-drift-detection-contract.md))

If invariant tests are missing or failing:

- invariants can drift undetected
- golden tests become unreliable
- shadow analysis becomes invalid
- analyzer promotion becomes unsafe

Therefore: Invariant tests MUST be comprehensive, deterministic, and enforced in CI.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- reject envelopes with invariant violations
- NEVER correct invariant violations
- NEVER mutate envelope structure

**Sentinel MUST:**

- treat invariant drift as BLOCKER drift
- classify invariant mismatches before structural drift

---

## 7. Versioning Impact

- `analyzerVersion` MUST bump if:
  - invariant behavior changes
  - golden outputs change
  - invariant error codes change
  - serialization behavior changes

- `schemaVersion` MUST bump if:
  - envelope structure changes
  - extension field structure changes

- `rulepackVersion` MUST bump if:
  - rulepack invariants change

Invariant tests are the primary protection for all versioning rules.

---

## 8. Testing Requirements

### 8.1 Testing Requirements Summary

Every invariant MUST:

✔ Have unit tests  
✔ Have integration tests  
✔ Have golden tests if applicable  
✔ Have shadow-mode regression tests  
✔ Map to a documented error code  
✔ Be included in the invariant registry  
✔ Run in CI across OSes  
✔ Block CI if missing or failing

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added:

- Invariant Test Registry section
- OS-cross-run requirement
- CI MUST fail if new invariants lack tests
- Strengthened golden + shadow enforcement
- Fully aligned with 06, 07, 14, 15, 17, 19, 20

**v1.0.0** — Initial release.

---

## 11. Invariant Test Registry (REQUIRED)

Every invariant MUST appear in this table and MUST map to a file in `arc-engine/tests/**`.

This registry MUST be kept in-sync with:

- [Invariants Contract](./06-invariants-contract.md)
- [Error Codes Specification](./06B-error-codes-specification.md)
- Test suite folder structure

The registry MUST be checked by CI.

### Snapshot Invariants

| Invariant ID | Description | Test Files | Error Code |
|--------------|-------------|------------|------------|
| SNAP-01 | Paths normalized | `tests/invariants/snapshot/paths.test.ts` | `SNAPSHOT_PATH_NORMALIZATION_FAILED` |
| SNAP-02 | No duplicate paths | `tests/invariants/snapshot/duplicates.test.ts` | `SNAPSHOT_DUPLICATE_PATH` |
| SNAP-03 | UTF-8 only | `tests/invariants/snapshot/encoding.test.ts` | `SNAPSHOT_INVALID_ENCODING` |
| SNAP-04 | Stable ordering | `tests/invariants/snapshot/order.test.ts` | `SNAPSHOT_UNSTABLE_ORDER` |
| SNAP-05 | No metadata | `tests/invariants/snapshot/metadata.test.ts` | `SNAPSHOT_METADATA_FORBIDDEN` |

### Graph Invariants

| ID | Description | Test Files | Error Code |
|----|-------------|------------|------------|
| GRAPH-01 | Missing node | `graph/nodes.test.ts` | `GRAPH_MISSING_NODE` |
| GRAPH-02 | Unexpected cycle | `graph/cycles.test.ts` | `GRAPH_UNEXPECTED_CYCLE` |
| GRAPH-03 | Invalid adjacency | `graph/edges.test.ts` | `GRAPH_INVALID_EDGE` |
| GRAPH-04 | Stats drift | `graph/stats.test.ts` | `GRAPH_STATS_DRIFT` |

### Cycle Invariants

| ID | Description | Test File | Code |
|----|-------------|-----------|------|
| CYCLE-01 | Stable ordering | `cycle/stability.test.ts` | `CYCLE_CLASSIFICATION_DRIFT` |
| CYCLE-02 | No false positives | `cycle/false-positive.test.ts` | `CYCLE_FALSE_POSITIVE` |
| CYCLE-03 | No false negatives | `cycle/false-negative.test.ts` | `CYCLE_FALSE_NEGATIVE` |

### Envelope Invariants

| ID | Description | Test File | Code |
|----|-------------|-----------|------|
| ENV-01 | `analyzerVersion` required | `envelope/version.test.ts` | `ENVELOPE_VERSION_MISSING` |
| ENV-02 | `schemaVersion` required | `envelope/schema.test.ts` | `ENVELOPE_SCHEMA_MISSING` |
| ENV-03 | `identityChain` stable | `envelope/identity.test.ts` | `ENVELOPE_IDENTITY_DRIFT` |
| ENV-04 | Timestamp canonicalization | `envelope/timestamp.test.ts` | `ENVELOPE_TIMESTAMP_DRIFT` |
| ENV-05 | No unknown fields | `envelope/shape.test.ts` | `ENVELOPE_UNKNOWN_FIELD` |

### Determinism Invariants

| ID | Description | Test File | Code |
|----|-------------|-----------|------|
| DET-01 | No locale drift | `determinism/locale.test.ts` | `DETERMINISM_LOCALE_DRIFT` |
| DET-02 | No randomness | `determinism/random.test.ts` | `DETERMINISM_RANDOMNESS` |
| DET-03 | No clock usage | `determinism/time.test.ts` | `DETERMINISM_TIME_DEPENDENCE` |
| DET-04 | Stable ordering | `determinism/ordering.test.ts` | `DETERMINISM_UNSTABLE_ORDER` |
| DET-05 | No partial configs | `determinism/config.test.ts` | `DETERMINISM_CONFIG_PARTIAL` |

