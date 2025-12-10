# Sentinel Shadow Analysis Contract (FINAL, Phase 2)

## 1. Purpose

This document defines the contract for Sentinel, ArcSight's shadow analysis and analyzer promotion system. Sentinel runs:

- the **LIVE analyzer** (current production analyzer), and
- the **SHADOW analyzer** (candidate next-version analyzer)

on the same RepoSnapshot, and compares their envelopes for:

- determinism
- semantic correctness
- drift
- backward compatibility
- rulepack stability

Sentinel determines whether a new `analyzerVersion` may be promoted from shadow → canary → production.

This is the central correctness mechanism of Phase 2.

---

## 2. Scope

### Included

Sentinel governs:

- running live and shadow analyzers in parallel
- envelope normalization and schema adapter rules
- deterministic comparison semantics
- drift classification
- rulepack compatibility rules
- promotion gating
- deterministic rejection reasons
- runtime integration expectations

### Excluded

This document does NOT cover:

- rulepack versioning (see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- schema evolution rules ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- envelope structure ([Envelope Format Spec](./15-envelope-format-spec.md))
- snapshot format ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- golden governance ([Golden Test Governance](./09-golden-test-governance.md))
- deterministic engine rules ([Determinism Contract](./07-determinism-contract.md))

---

## 3. Definitions & Terms

**Live Analyzer**  
Current production `analyzerVersion`. Must always produce trusted envelopes.

**Shadow Analyzer**  
Candidate analyzer under evaluation. Never user-visible.

**Envelope Drift**  
The semantic or structural difference between live and shadow envelopes.

**Promotion Pipeline**

```
shadow → canary → production
```

**Blocker Drift**  
Any drift that violates backward compatibility and MUST block promotion.

**Benign Drift**  
Expected drift due to deterministic improvements or minor rulepack evolution.

---

## 4. Rules / Contract

### 4.1 Live Analyzer Is Always Source of Truth

The Live analyzer's envelope is the baseline.

Shadow analyzer results:

- MUST NEVER be exposed to end users
- MUST NEVER override live analyzer results
- MUST NOT influence CheckRuns
- MUST NOT alter PR statuses

Under no circumstances may the SHADOW analyzer become authoritative until promoted.

### 4.2 Deterministic Execution Requirements

For each PR, Sentinel MUST run:

- Live analyzer (version X)
- Shadow analyzer (version X+1 candidate)

Both MUST run on the exact same RepoSnapshot.

Snapshot immutability is guaranteed by [RepoSnapshot Contract](./14-repo-snapshot-contract.md).

Shadow analysis MUST use:

- same limits
- same rulepack set
- same config
- same sandbox constraints

This ensures apples-to-apples comparison.

### 4.3 Envelope Normalization

Before comparison:

Both live and shadow envelopes MUST be transformed into:

**CurrentSchemaEnvelope**

via schema adapters ([Schema Evolution Contract](./11-schema-evolution-contract.md)).

All comparisons MUST use:

- `stableStringify`
- canonical ordering
- signature MUST NOT be included

Comparisons MUST occur on the envelope content, not its raw serialized form.

### 4.4 Drift Classification

Sentinel MUST classify drift into exactly one of the categories:

#### 4.4.1 BLOCKER Drift (Promotion MUST Fail)

The following require immediate rejection:

- **Schema violation**  
  Shadow envelope violates structural rules of [Envelope Format Spec](./15-envelope-format-spec.md).

- **Invariants violation**  
  Shadow envelope fails invariants from [Invariants Contract](./06-invariants-contract.md).

- **Cycle drift**  
  Cycles differ except for expected canonical ordering changes.

- **Graph drift**  
  `node_count` or `edge_count` differ without explicit justification.

- **Unexpected extension changes**  
  Rulepacks produce incompatible fields.

- **Removing fields**  
  Shadow removes ANY field that existed in live under same `schemaVersion`.

- **Unstable signature behavior**  
  Shadow produces nondeterministic signatures across runs.

- **New `degraded_reason` not declared in contract**  
  MUST block until declared.

#### 4.4.2 WARNING Drift (Promotion MAY Continue)

Requires human review:

- New optional extension fields
- Minor rulepack version bumps
- Deterministic improvements yielding fewer cycles
- Performance optimizations changing `graph_stats` slightly
- Truncation behavior changes that are documented and intended

#### 4.4.3 BENIGN Drift (Autopass)

No human action required:

- Updated `analyzerVersion` string
- Updated `rulepackVersion` numbers
- Different signature due only to `analyzerVersion` or `rulepackVersion` change
- Extensions added in new namespaces

### 4.5 Drift Comparison Rules

Sentinel MUST compare envelopes using:

#### 4.5.1 Field-by-field semantic comparison

Key rules:

- ordering differences MUST be ignored (adapter normalizes ordering)
- signature MUST be ignored (recomputed after comparison)
- `generated_at` MUST be ignored
- `identity.*` MUST match exactly
- `core.*` MUST match semantically unless documented improvements exist

#### 4.5.2 Cycle Comparison Logic

Shadow cycles MUST be:

- same set of cycles
- OR
- a strict deterministic improvement (fewer or equal cycles, no new cycles)

#### 4.5.3 Extension Comparison Logic

Extensions MUST be compared namespace-by-namespace:

- unknown fields MUST NOT be removed
- known fields MUST NOT change type
- values MUST remain deterministic

If a rulepack is upgraded from version A → A+1:

- new fields allowed
- field removals forbidden
- meaning changes require major version bump

### 4.6 Deterministic Drift Detection Algorithm

Sentinel MUST use the following algorithm:

1. Normalize both envelopes (apply adapters → current schema)
2. Remove signature + `generated_at`
3. Deep-sort both envelopes
4. Perform structural diff
5. Map diff into drift categories
6. Produce deterministic drift report

Any nondeterminism (orderings, set iteration, JSON key ordering) MUST be normalized before comparison.

### 4.7 Promotion Gating Logic

**Promotion MUST be blocked when:**

- BLOCKER drift detected
- invariants fail
- schema violated
- extension field removed or semantics changed
- rulepack violates versioning rules
- shadow produces degraded envelopes unexpectedly
- shadow produces error envelopes where live produces success

**Promotion MAY continue when:**

- only WARNING drift exists
- new extension fields are added
- performance changes alter counts within documented tolerances
- rulepacks evolve minor version

**Promotion MUST auto-pass when:**

- only BENIGN drift exists
- only version strings differ
- new extensions added under new namespaces

### 4.8 Error & Degraded Behavior Comparison

**Case 1 — Live success, Shadow degraded**

- BLOCKER drift
- → Promotion MUST fail.

**Case 2 — Live degraded, Shadow success**

- BENIGN
- → Promotion allowed, but MUST be logged.

**Case 3 — Both degraded**

- compare `degraded_reason` (canonical enum)
- differences may be BLOCKER unless justified.

**Case 4 — Live error, Shadow success**

- BENIGN but MUST log improvement.

**Case 5 — Shadow error**

- Always BLOCKER.

### 4.9 Forbidden Behaviors

**Sentinel MUST NOT:**

- retry the shadow analyzer
- attempt fallback analysis modes
- rewrite envelopes
- override degraded status
- infer missing fields
- bypass schema adapters
- ignore extension mismatches
- rewrite truncated cycles
- compare raw JSON instead of normalized schema

**Shadow analyzer MUST NOT:**

- access runtime
- change limits
- adjust behavior based on environment
- mutate the RepoSnapshot
- run additional rulepacks not present in live

**Live analyzer MUST NOT:**

- depend on shadow analyzer behavior
- change limits at runtime

---

## 5. Determinism Impact

Sentinel comparisons MUST be deterministic across:

- machines
- OS
- Node versions
- CPU architectures
- execution scheduling

Given the same RepoSnapshot, drift classification MUST always produce the same result.

Violations of determinism here invalidate analyzer promotion.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- provide identical RepoSnapshot to live + shadow
- never modify envelopes
- ensure identical config, limits, rulepack set
- send both envelopes to Sentinel

**Engine MUST:**

- guarantee deterministic analysis
- never access runtime metadata
- produce structurally valid envelopes even in degraded/error states

---

## 7. Versioning Impact

**Promotion requires:**

- `analyzerVersion` bump
- `rulepackVersion` updates where applicable
- golden updates
- drift classification approval

**SchemaVersion bump requires:**

- adapter updates
- drift logic updates

**RulepackVersion bumps require:**

- namespace-by-namespace drift evaluation

---

## 8. Testing Requirements

### 8.1 Shadow-vs-Live Comparison Tests

Test cases MUST include:

- no drift
- benign drift
- warning drift
- blocker drift

### 8.2 Adapter Consistency Tests

Adapters MUST normalize envelopes so drift is consistent independent of `schemaVersion`.

### 8.3 Round-Trip Snapshot Tests

Shadow & live MUST use identical input snapshots.

### 8.4 Fuzzing Tests

Sentinel must be tested against pathological repos to ensure drift classification remains deterministic.

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

---

## 10. Change Log (Append-Only)

**v1.0.0** — Initial Phase-2 Sentinel Contract

Defines live vs shadow execution, deterministic drift classification, promotion gating rules, extension comparison, rulepack compatibility, and all forbidden behaviors.

