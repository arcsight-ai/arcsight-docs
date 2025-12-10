# Invariant System Index & Master Specification

## Master Reference for Engine Correctness, Testing, Error Codes & CI Enforcement

**FINAL VERSION — Phase 1 & Phase 2**

Doc 06X unifies the entire invariant subsystem:

- [Invariants Contract](./06-invariants-contract.md)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)
- [Error Codes Catalog](./06B-error-codes-catalog.md)
- [Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md)

This is the authoritative root document for all invariant-related architecture.

---

## 1. Purpose

Doc 06X defines:

- the master invariant taxonomy
- mapping between invariants, error codes, tests and CI enforcement
- how invariants evolve
- how foundational and derived invariants relate
- how invariants interact with runtime, Sentinel, schema evolution, degraded mode, and rulepacks

This document ensures that the invariant subsystem remains:

- deterministic
- testable
- auditable
- drift-resistant
- promotable under multi-analyzer shadow mode

---

## 2. Scope

Doc 06X covers:

- invariant categories
- invariant lifecycle
- invariant dependency graph
- mapping to error codes, tests, CI rules
- invariant stability guarantees
- interactions with schema versions, adapters, and rulepacks

**Not included:**

- actual invariant definitions ([Invariants Contract](./06-invariants-contract.md) contains the rules)
- actual test definitions ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md) contains test specs)
- error code definitions ([Error Codes Catalog](./06B-error-codes-catalog.md) contains codes)

---

## 3. Definitions & Terms

**Invariant**  
A rule defined in [Invariants Contract](./06-invariants-contract.md) that MUST hold for snapshot, graph, envelope, namespaces, schema, identity, determinism, or degraded mode.

**Derived Invariant**  
An invariant whose correctness depends on one or more foundational invariants.

**Example:** Signature Determinism depends on:
- stable ordering
- stable stringify
- canonical envelope structure

**Invariant Lifecycle**  
Each invariant exists in one of these states:

| State | Meaning |
|-------|---------|
| Draft | Defined but not enforced; tests not required yet |
| Active | Fully enforced, error codes mapped, CI blocking |
| Deprecated | Still supported, but scheduled for removal in next `analyzerVersion` |
| Removed | No longer enforced; schema adapters preserve compatibility |

**Invariant Registry**  
The authoritative table that maps invariants → tests → error codes → CI rules. Generated manually in Phase 1, automatically in Phase 2.

---

## 4. Rules / Contract

### 4.1 The Invariant Subsystem Architecture

```
             +---------------------------+
             | 06 — Invariants Contract  |
             | (Rules that MUST hold)    |
             +---------------------------+
                           |
                           v
             +---------------------------+
             | 06A — Test Suite Spec     |
             | (How each invariant is    |
             |  tested)                  |
             +---------------------------+
                           |
                           v
             +---------------------------+
             | 06B — Error Codes Catalog |
             | (How invariants fail)     |
             +---------------------------+
                           |
                           v
             +---------------------------+
             | 06C — CI Enforcement Gate |
             | (How failure blocks merge)|
             +---------------------------+
```

Doc 06X sits above these four as the controlling index.

### 4.2 Invariant Category Index

All invariants MUST belong to exactly one category:

#### 4.2.1 Snapshot Invariants

- Path normalization
- Newline normalization
- Sorted file ordering
- Deterministic hash
- No duplicate entries
- No empty files or zero-length paths

#### 4.2.2 Graph Invariants

- All nodes referenced are defined
- No self-loops counted as cycles
- GraphStats must be consistent
- Degree counts must match edges

#### 4.2.3 Cycle Invariants

- Canonicalized cycle lists
- Stable cycle classification
- No phantom cycles
- No missing cycles

#### 4.2.4 Envelope Invariants

- Required fields present
- Forbidden fields absent
- Extension namespaces isolated
- `identityChain` preserved

#### 4.2.5 Namespace & Extension Invariants

- No cross-namespace writes
- No extension deletion
- No duplicate extension keys
- Unknown fields preserved

#### 4.2.6 Schema Invariants

- Adapters must preserve invariants
- No backward-incompatible changes
- `adapter(vᵢ→vᵢ₊₁)` must produce valid envelopes

#### 4.2.7 Determinism Invariants

- No randomness
- No `Date.now()`
- No locale-sensitive operations
- No environment access
- No partial/dynamic config merges

#### 4.2.8 Degraded Mode Invariants

- Limits applied deterministically
- Degrade reason must be included
- Complexity fallback must be stable

#### 4.2.9 Signature & Identity Invariants

- Signature must be reproducible
- `identityChain` must be deterministic
- No signature drift for identical envelopes

### 4.3 Derived Invariants (NEW)

Derived invariants MUST declare their dependencies.

| Derived Invariant | Depends On |
|-------------------|------------|
| Signature Determinism | Stable ordering, stable stringify, canonical envelope |
| Schema Upgrade Determinism | Adapter determinism, namespace invariants, envelope invariants |
| Cycle Classification Stability | Graph invariants, cycle canonicalization |
| Shadow Drift Stability | All invariants above |

**If a foundational invariant changes:**

- all dependent invariants MUST be revalidated
- tests updated
- error codes updated
- `analyzerVersion` bumped

This prevents silent breakage across dependent subsystems.

### 4.4 Invariant Lifecycle Model (NEW)

Each invariant has a lifecycle:

```
Draft → Active → Deprecated → Removed
```

**Draft**

- Defined in [Invariants Contract](./06-invariants-contract.md)
- No tests required
- No CI enforcement

**Active**

- Fully enforced
- Mapped to error code
- Tests mandatory
- CI blocks violations

**Deprecated**

- Still supported in engine
- Marked for removal
- Cannot be weakened without `analyzerVersion` bump

**Removed**

- No longer enforced
- Schema adapters maintain backward compatibility

This aligns ArcSight with real compiler and schema governance systems.

### 4.5 Mapping: Invariants → Error Codes

[Error Codes Catalog](./06B-error-codes-catalog.md) defines codes; Doc 06X defines the mapping index.

| Category | Prefix |
|----------|--------|
| Snapshot | `SNAPSHOT_*` |
| Graph | `GRAPH_*` |
| Cycles | `CYCLE_*` |
| Envelope | `ENVELOPE_*` |
| Namespace | `NAMESPACE_*` |
| Schema | `SCHEMA_*` |
| Determinism | `DETERMINISM_*` |
| Degraded | `DEGRADE_*` |
| Signature | `SIGNATURE_*` |

Every invariant MUST map to exactly one error code.

If missing: CI MUST fail.

### 4.6 Mapping: Invariants → Tests

[Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md) defines the test structure; Doc 06X defines the directories:

- `tests/invariants/snapshot/*.test.ts`
- `tests/invariants/graph/*.test.ts`
- `tests/invariants/cycles/*.test.ts`
- `tests/invariants/envelope/*.test.ts`
- `tests/invariants/namespace/*.test.ts`
- `tests/invariants/schema/*.test.ts`
- `tests/invariants/determinism/*.test.ts`
- `tests/invariants/degraded/*.test.ts`
- `tests/invariants/signature/*.test.ts`

**A valid invariant MUST have:**

- at least one unit test
- at least one integration test
- golden test coverage when applicable

Missing coverage → CI MUST fail.

### 4.7 Mapping: Error Codes → CI Enforcement

[Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md) specifies full enforcement. Doc 06X documents the mapping:

| Error Code Family | CI Behavior |
|-------------------|-------------|
| `SNAPSHOT_*` | Fail |
| `GRAPH_*` | Fail |
| `CYCLE_*` | Fail |
| `ENVELOPE_*` | Fail |
| `SCHEMA_*` | Fail |
| `DETERMINISM_*` | Fail |
| `DEGRADE_*` | Fail unless degrade expected |
| `SIGNATURE_*` | Fail |

No invariant violation may be "warning-only."

### 4.8 Full Compliance Matrix

CI in Phase 2 MUST auto-generate:

| Invariant | Lifecycle | Error Code | Unit Test | Integration Test | Golden Test | CI Rule |
|-----------|-----------|------------|-----------|------------------|-------------|---------|
| ✔ | Active | ✔ | ✔ | ✔ | ✔ | Enforced |

Anything missing MUST block merge.

### 4.9 Future Tooling: Automatic Invariant Registry Generation (NEW)

Phase 2 introduces:

```
pnpm run generate:invariant-registry
```

This script will:

- parse [Invariants Contract](./06-invariants-contract.md)
- extract invariant names
- check test coverage ([Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md))
- check error codes ([Error Codes Catalog](./06B-error-codes-catalog.md))
- check CI mapping ([Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md))
- regenerate a machine-readable registry under:

```
packages/arc-engine/invariants.json
```

This closes all human error gaps.

---

## 5. Determinism Impact

The invariant subsystem is the foundation of:

- deterministic analysis ([Determinism Contract](./07-determinism-contract.md))
- reproducible golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- stable analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- accurate drift detection ([Drift Detection Contract](./19-drift-detection-contract.md))
- safe schema evolution ([Schema Evolution Contract](./11-schema-evolution-contract.md))

If the invariant subsystem is unstable:

- determinism breaks
- golden tests become unreliable
- shadow analysis becomes invalid
- analyzer promotion becomes unsafe
- schema evolution becomes unpredictable

Therefore: The invariant subsystem MUST be comprehensive, deterministic, and auditable.

---

## 6. Runtime / Engine Boundary Impact

**Engine MUST:**

- enforce all active invariants
- emit only registered error codes
- never bypass invariant checks
- maintain invariant lifecycle state

**Runtime MUST:**

- never mutate envelopes after invariant validation
- never hide invariant violations
- pass error codes to Sentinel unchanged
- respect invariant boundaries

**Sentinel MUST:**

- use invariants to detect drift
- classify invariant violations as BLOCKER
- never downgrade invariant severity
- preserve invariant state in drift reports

---

## 7. Versioning Impact

- `analyzerVersion` MUST bump when:
  - any invariant behavior changes
  - any invariant lifecycle state changes
  - any derived invariant dependency changes
  - any error code is added or removed

- `schemaVersion` MUST bump when:
  - envelope structure changes
  - extension namespace structure changes

The invariant subsystem guarantees version stability and prevents semantic drift.

---

## 8. Testing Requirements

### 8.1 Sentinel Integration

Sentinel uses invariants to detect drift:

- live vs shadow envelope mismatch
- cycle classification drift
- adapter drift
- signature drift

If invariants are unstable, Sentinel becomes unreliable.

**Therefore:**

No invariant may change without bumping `analyzerVersion`.

### 8.2 Rulepack Integration

Rulepacks MUST obey:

- namespace invariants
- determinism invariants
- envelope invariants

Violations MUST emit:

- `NAMESPACE_*`
- `ENVELOPE_*`
- `DETERMINISM_*`

Rulepacks may NOT weaken or override core invariants.

### 8.3 Schema Adapters Integration

Schema adapters MUST:

- preserve all invariants
- enforce new invariants deterministically
- emit `SCHEMA_*` codes when failing

Adapters may evolve schema — but not weaken invariants.

---

## 9. Cross-References

Doc 06X depends on:

- [Invariants Contract](./06-invariants-contract.md)
- [Invariant Test Suite Specification](./06A-invariant-test-suite-specification.md)
- [Error Codes Catalog](./06B-error-codes-catalog.md)
- [Invariant Enforcement Checker](./06C-invariant-enforcement-checker.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 10. Change Log (Append-Only)

**v1.0.0** — Final Release

- Added derived invariants and dependency tracking
- Added invariant lifecycle (Draft → Removed)
- Added future invariant registry tooling
- Added complete mapping index for all 06-subdocs
- Added rulepack, schema, sentinel integration guarantees

