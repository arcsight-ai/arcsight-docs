# Invariants Contract (FINAL, Phase 2)

## Deterministic Conditions That MUST Hold Across Engine, Runtime, Sentinel & Drift Detection

This document defines all ArcSight invariants — the hard, global correctness rules that MUST always hold true across:

- RepoSnapshot
- Graph
- Cycle structures
- Envelope
- Degraded behavior
- Analyzer/Rulepack output
- Signature computation
- Schema evolution
- Drift detection

If any invariant is violated, the engine MUST:

- produce a deterministic error envelope
- set degraded: "error" or a canonical error_code
- NEVER publish a partial or corrupted envelope
- NEVER continue analysis

These invariants form the foundation of all deterministic behavior in ArcSight.

---

## 1. Purpose

This contract defines the mandatory conditions that all ArcSight components must satisfy for:

- deterministic analysis
- reproducible golden tests
- stable analyzer promotion
- shadow-mode comparisons
- long-term schema evolution
- enterprise rulepack safety

These invariants define what it means for an ArcSight result to be correct.

Violating an invariant is equivalent to an engine bug.

---

## 2. Scope

This contract applies to:

- RepoSnapshot
- Graph & cycle detection
- Invariants inside rulepacks
- Envelope & signatures
- Degraded mode
- Error classification
- Engine ↔ Runtime boundary
- Drift detection in Sentinel

**NOT included:**

- Schema evolution rules (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- Rulepack versioning rules (see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- Drift classification semantics (see [Drift Detection Contract](./19-drift-detection-contract.md))
- Snapshot format (see [RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- Determinism guarantees (see [Determinism Contract](./07-determinism-contract.md))

---

## 3. Definitions

**Invariant**  
A rule that MUST always hold for the system to be considered correct.

**Hard invariant**  
Violation is a BLOCKER-level failure.

**Soft invariant**  
Violation may enter degraded mode but MUST still output a valid envelope.

**Graph**  
Canonical representation of relationships between files/nodes.

**Envelope**  
Final deterministic output produced by the engine.

**Signature**  
Canonical stable hash of the envelope ([Envelope Format Spec](./15-envelope-format-spec.md)).

---

## 4. Rules / Contract

### 4.1 Global Invariants

These invariants MUST hold across all components:

#### 4.1.1 Deterministic Input → Deterministic Output

Given a canonical RepoSnapshot:

```
analyze(snapshot_A) === analyze(snapshot_A)
```

No entropy, no randomness, no time-based variance.

#### 4.1.2 Canonical Ordering Everywhere

All lists MUST be:

- sorted deterministically
- stable across platforms
- normalized according to [Determinism Contract](./07-determinism-contract.md) canonicalization rules

Includes:

- files
- graph nodes
- edges
- cycles
- extension fields
- rulepack outputs

#### 4.1.3 Envelope MUST Always Be Valid

Engine MUST NEVER emit:

- missing required fields
- partial data
- undefined fields
- runtime-dependent values (except allowed timestamps)

Envelope MUST pass:

- `schemaVersion` validation
- rulepack invariants
- identity chain validation
- extension namespace validation

#### 4.1.4 No Invariant May Depend on Runtime Context

Invariants MUST NOT depend on:

- branch name
- PR metadata
- user ID
- commit message
- time of day
- machine locale
- environment variables

This protects replay, shadow mode, and deterministic caching.

### 4.2 RepoSnapshot Invariants

RepoSnapshot ([RepoSnapshot Contract](./14-repo-snapshot-contract.md)) MUST satisfy:

#### 4.2.1 Canonical Path Rules

Every path MUST:

- be UTF-8
- be normalized to POSIX
- contain no `..` escapes
- contain no symlinks
- contain no illegal characters
- remain within repository root

If any violation occurs → hard failure.

#### 4.2.2 File Content Invariants

File bytes MUST:

- be read raw
- not undergo CRLF conversion
- not undergo Unicode normalization
- not strip BOMs

Hash MUST be computed on exact bytes.

#### 4.2.3 Snapshot Size Invariants

Snapshot MUST fail if:

- file count > limit
- total bytes > limit
- max depth > limit
- path length > limit

Snapshot MUST NOT be constructed if limits exceeded.

#### 4.2.4 Snapshot MUST Represent a Single Git Tree

Snapshot MUST NOT include:

- untracked files
- temporary files
- root escapes
- dynamic generated files

### 4.3 Graph Invariants

Graph MUST satisfy:

#### 4.3.1 Node & Edge Type Invariants

Nodes MUST have:

- valid type
- stable identity
- canonical path

Edges MUST be:

- valid
- normalized
- type-safe

Unknown edge types → fail.

#### 4.3.2 No Missing Nodes

Every `edge.from` and `edge.to` MUST reference a node.

If edge refers to non-existent node → BLOCKER.

#### 4.3.3 Cycle Invariants

If a cycle exists:

- cycles MUST be detected deterministically
- cycles MUST be sorted deterministically
- cycle grouping MUST be stable

A cycle MUST NOT disappear unless `analyzerVersion` changes.

New cycles appearing unexpectedly → BLOCKER drift ([Drift Detection Contract](./19-drift-detection-contract.md)).

#### 4.3.4 Graph MUST Be Canonical

Graph MUST NOT depend on traversal order.

DFS/BFS starting points MUST NOT influence results.

### 4.4 Rulepack Invariants

Rulepacks MUST satisfy:

#### 4.4.1 Rulepacks MUST NOT Modify Core Structures

Rulepacks MAY emit extension fields but MUST NOT:

- modify RepoSnapshot
- modify Graph
- modify identity chain
- remove fields
- reorder nodes or edges

#### 4.4.2 Rulepack Output MUST Be Deterministic

Rulepack output MUST:

- be stable under canonical ordering
- contain JSON-serializable values only
- never include timestamps unless explicitly allowed
- never include random IDs

Unknown fields MUST remain preserved.

#### 4.4.3 Rulepacks MUST NOT Fail Open

Any internal rulepack failure MUST:

- surface as deterministic `rulepack_error`
- NOT silently skip analysis
- NOT mutate envelope

### 4.5 Envelope Invariants

Envelope ([Envelope Format Spec](./15-envelope-format-spec.md)) MUST satisfy:

#### 4.5.1 Required Fields MUST Always Exist

- `analyzer_version`
- `schema_version`
- `signature`
- `graph`
- `identity_chain`
- `extensions` (possibly empty)

Missing fields → hard failure.

#### 4.5.2 Envelope MUST Be Valid JSON

Envelope MUST NOT contain:

- `undefined`
- `NaN`
- `Infinity`
- Maps/Sets
- Dates
- cyclic references

#### 4.5.3 Identity Chain MUST Be Canonical

Identity chain MUST:

- include repo fingerprint
- include snapshot hash
- reflect `analyzerVersion`
- be stable for identical snapshots

#### 4.5.4 Signature MUST Be Stable

Signature MUST:

- use SHA-256
- hash canonical `stableStringify(envelope)`
- never normalize away fields
- never introduce ordering differences

### 4.6 Degraded Mode Invariants

If limits are exceeded, engine MUST:

#### 4.6.1 Still Produce a Valid Envelope

Even in degraded mode, MUST output:

- valid envelope
- deterministic `degraded_reason` enum
- stable signature

#### 4.6.2 Degraded MUST Be Deterministic

Same input → same `degraded_reason`.

Degraded vs non-degraded mismatch between live and shadow → BLOCKER.

#### 4.6.3 Degraded MUST NOT Hide Failures

Graph truncation MUST:

- occur after canonical ordering
- use prefix-based truncation
- include deterministic annotations

### 4.7 Error Invariants

#### 4.7.1 Errors MUST Be Deterministic

No error message may contain:

- timestamps
- machine-specific paths
- stack traces unless normalized

#### 4.7.2 Errors MUST Use Canonical Error Codes

Examples:

- `snapshot-invalid`
- `graph-invariant-violated`
- `cycle-detection-error`
- `rulepack-failure`
- `envelope-invariant-violated`

Error codes MUST be stable across runs.

### 4.8 Sentinel & Drift Invariants

Sentinel MUST rely on invariants to classify drift:

#### 4.8.1 Drift MUST NOT Be Calculated If Invariants Fail

If shadow envelope violates invariants →

Sentinel MUST classify as BLOCKER without diffing.

(Per [Drift Detection Contract](./19-drift-detection-contract.md).)

#### 4.8.2 Live vs Shadow MUST Use Identical Config

If config hash differs → BLOCKER.

(Per [Deterministic Config Contract](./20-deterministic-config-contract.md).)

#### 4.8.3 Unknown Extensions MUST Be Preserved

No adapter or drift logic may:

- delete
- rename
- collapse

unknown vendor fields.

### 4.9 Runtime / Engine Boundary Invariants

#### 4.9.1 Runtime MUST NOT Influence Engine Behavior

No:

- env-based behavior
- PR-based config
- per-request flags
- heuristic adjustments

#### 4.9.2 Engine MUST NOT Depend on Runtime Data

Engine MUST not read:

- timestamps
- branch names
- commit metadata
- user metadata

---

## 5. Determinism Impact

Invariants are the foundation of:

- deterministic analysis ([Determinism Contract](./07-determinism-contract.md))
- reproducible golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- stable analyzer promotion ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- shadow-mode comparisons ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- long-term schema evolution ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- enterprise rulepack safety ([Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md))

If invariants are violated:

- determinism breaks
- golden tests become unreliable
- shadow analysis becomes invalid
- drift detection becomes meaningless
- schema evolution becomes unsafe

Therefore: Invariants MUST be enforced at every stage of analysis.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- supply canonical RepoSnapshots that satisfy all snapshot invariants
- NEVER modify envelopes after engine produces them
- NEVER retry analysis with different config
- NEVER hide invariant violations

**Engine MUST:**

- validate all invariants before producing envelope
- produce deterministic error envelopes for invariant violations
- NEVER continue analysis after invariant failure
- NEVER produce partial or corrupted envelopes

**Sentinel MUST:**

- validate invariants before computing drift
- classify invariant failures as BLOCKER immediately
- NEVER compare envelopes if either violates invariants

---

## 7. Versioning Impact

Violation of any invariant requires:

- `analyzerVersion` bump at minimum
- possibly `rulepackVersion` bump
- `schemaVersion` bump if invariant change modifies structure

Invariants guarantee version stability.

---

## 8. Testing Requirements

Engine MUST include:

### 8.1 Invariant Regression Tests

Every invariant MUST have corresponding tests.

### 8.2 Cross-Version Stability Tests

Shadow vs live MUST pass invariants before drift comparison.

### 8.3 Golden Tests MUST Encode Invariants

Golden tests MUST fail if invariants violated.

---

## 9. Cross-References

- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)
- [Drift Detection Contract](./19-drift-detection-contract.md)
- [Deterministic Config Contract](./20-deterministic-config-contract.md)

---

## 10. Change Log (Append-Only)

**v1.0.0** — Initial Phase-2 Release

Defines all invariants across snapshot, graph, cycles, envelope, degraded mode, signatures, rulepacks, error modes, runtime boundaries, and drift detection.

