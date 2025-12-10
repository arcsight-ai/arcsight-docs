# Deterministic Config Contract (FINAL UPDATED, Phase 2)

## 1. Purpose

This document defines the deterministic configuration model for the ArcSight engine and Sentinel drift detector.

AnalyzerConfig is a first-class deterministic input to the analyzer, equal in importance to:

- RepoSnapshot ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- Envelope schema ([Envelope Format Spec](./15-envelope-format-spec.md))
- Rulepacks ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- Limit model ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))

This contract ensures:

- identical analyzer behavior across machines and executions
- reproducible envelopes
- stable drift detection
- deterministic rulepack execution
- no environment-based drift
- strict separation of config and runtime context

AnalyzerConfig MUST always be canonical, normalized, immutable, and identical for all parallel analyzer executions (shadow/live/canary).

---

## 2. Scope

### Included

This contract governs:

- config shape, normalization, and serialization
- deterministic hashing (`config_snapshot_hash`)
- allowed and forbidden fields
- config merge rules
- interaction with RepoSnapshot, rulepacks, and limits
- runtime and engine boundaries
- shadow/live equivalence for Sentinel

### Excluded

(Not defined here)

- schema structure ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- snapshot format ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- rulepack semantics ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- drift comparison logic ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- promotion lifecycle ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))
- sandbox limits ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))

---

## 3. Definitions & Terms

**AnalyzerConfig**  
User-facing configuration model provided to the engine. Immutable and deterministic after normalization.

**NormalizedConfig**  
The fully resolved, canonical, deterministic configuration shape after merging defaults and applying validation.

**Config Snapshot Hash**  
SHA-256 hash of `stableStringify(NormalizedConfig)`. Included in `meta.config_snapshot_hash` of the Envelope.

**Deterministic Config**  
A config that produces the same `NormalizedConfig` on every environment, machine, OS, and execution.

---

## 4. Rules / Contract

### 4.1 Config MUST Be Deterministically Normalized

All AnalyzerConfig inputs MUST be:

- merged using deterministic, globally-defined rules
- stripped of unknown, invalid, or runtime-only fields
- validated against a strict schema
- deep-sorted lexicographically
- converted to canonical JSON via `stableStringify`

Normalization MUST NOT depend on:

- PR metadata
- environment variables
- runtime state
- file contents
- historical analysis output
- request context

**Added Requirement (NEW):**

Config MUST NOT depend on:

- PR titles
- branch names
- usernames
- commit messages
- timestamps
- request order

Any config value incorporating request context MUST cause rejection.

### 4.2 Shadow & Live MUST Use Identical Normalized Config

For deterministic drift detection, Sentinel requires:

```
normalizeConfig(config_live) === normalizeConfig(config_shadow)
```

If hashes differ:

- Sentinel MUST classify drift as BLOCKER
- and MUST NOT perform structural/semantic envelope diffing.

This prevents:

- mismatched rulepack settings
- different flag defaults
- partial upgrades
- divergent rulepack version inputs

### 4.3 Config MUST Be Immutable After Normalization

After `NormalizeConfig`:

- No new fields may be added
- No defaults may be re-applied
- No environment values may be injected
- No rulepack may mutate config
- No runtime layer may override values

The engine MUST treat `NormalizedConfig` as a frozen value.

### 4.4 Config MUST Be Fully Resolved Before Analysis (NEW)

Layered or staged config merging MUST NOT happen during analysis.

Config merges MUST be:

- static
- deterministic
- globally defined
- completed before analysis begins

Once `Analyze()` starts:

➡️ No further config merging or resolution is allowed.

This prevents nondeterministic "merge order drift."

### 4.5 Locale Independence (NEW)

AnalyzerConfig MUST NOT depend on machine locale.

**Forbidden:**

- locale-sensitive string lowercasing (e.g., Turkish "İ" bug)
- locale-specific sorting or collation
- locale-dependent number formatting (comma vs period)
- Unicode behaviors that vary by locale

All operations MUST use deterministic, locale-independent functions.

### 4.6 Config MUST NOT Use Dynamic Defaults

**Forbidden dynamic defaults:**

- defaults depending on RepoSnapshot (e.g., "if >200 files, enable X")
- defaults depending on historical analysis
- defaults depending on runtime or request context
- heuristics that change based on code size or language detection

**Allowed:**

- static defaults defined in the contract
- deterministic, globally constant defaults

Dynamic defaults risk producing nondeterministic envelopes.

### 4.7 Config MUST Be JSON-Serializable & Pure Data

Config MUST contain only:

- strings
- numbers
- booleans
- arrays
- plain objects

**Forbidden types:**

- Dates
- Maps / Sets
- BigInt
- functions
- class instances
- undefined
- NaN, Infinity
- Buffers or binary data

This ensures stable serialization and hashing.

### 4.8 Forbidden Config Sources

AnalyzerConfig MUST NOT draw from:

- environment variables
- user locale
- machine CPU count
- memory availability
- runtime clock
- file system
- network
- Git metadata
- PR-specific metadata

Config MUST be identical regardless of runtime conditions.

---

## 5. Determinism Impact

This contract directly ensures:

- reproducible signatures ([Envelope Format Spec](./15-envelope-format-spec.md))
- stable drift detection ([Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- deterministic rulepack output ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- invariant enforcement ([Invariants Contract](./06-invariants-contract.md))
- deterministic golden tests ([Golden Test Governance](./09-golden-test-governance.md))
- reliable shadow vs live analysis ([Analyzer Promotion Policy](./17-analyzer-promotion-policy.md))

Any nondeterminism in config invalidates the entire ArcSight deterministic model.

---

## 6. Runtime / Engine Boundary Impact

**Engine MUST:**

- accept only normalized configs
- treat config as immutable
- use config consistently across rulepacks and analysis steps
- reject configs that fail deterministic rules

**Runtime MUST:**

- produce AnalyzerConfig through a deterministic merge pipeline
- normalize before sending to engine
- NEVER inject request-derived or environment-derived fields
- compute config snapshot hash via `stableStringify`

**Runtime MUST NOT:**

- mutate config after normalization
- override config per request
- modify config for shadow or canary analyzers

---

## 7. Versioning Impact

- `analyzerVersion` MUST bump if:
  - config semantics change
  - default values change
  - rulepack behavior governed by config changes
  - schema for AnalyzerConfig evolves

- `schemaVersion` MUST NOT bump due to config alone.

- `rulepackVersion` MUST bump if rulepack-specific config changes semantics.

---

## 8. Testing Requirements

### 8.1 Config Normalization Tests

Test:

- sorting
- canonicalization
- removal of unknown fields
- stable hashing

### 8.2 Cross-Environment Consistency Tests

Run identical config through:

- Linux
- macOS
- Windows (if applicable)
- multiple Node versions

Output MUST match exactly.

### 8.3 Drift Detection Config Equivalence Tests

Shadow/live config equality MUST be enforced.

If mismatched → BLOCKER drift.

### 8.4 Locale Independence Tests

Explicit checks for locale-dependent behavior.

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added:

- locale-independence rule
- full config resolution requirement
- restriction on request-context metadata
- dynamic-defaults prohibition
- shadow/live config hash equivalence rule

**v1.0.0** — Initial deterministic config contract.

