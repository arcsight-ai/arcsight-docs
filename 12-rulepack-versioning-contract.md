# Rulepack Versioning Contract (Final, Phase 1 + Phase 2)

## 1. Purpose

This document defines the permanent rules governing:

- rulepack versioning
- rulepack namespace allocation
- rulepack purity
- deterministic rulepack execution
- compatibility requirements
- extension-field stability

Rulepacks are modular analysis units that produce deterministic fields inside:

```
extensions.<namespace>.*
```

This contract ensures:

✔ deterministic rulepack behavior  
✔ semantic versioning stability  
✔ namespace isolation  
✔ safe long-term extensibility  
✔ compatibility with shadow analyzers (Phase 2)  
✔ no namespace collisions  
✔ guaranteed envelope stability  

---

## 2. Scope

This contract applies to:

- all built-in rulepacks (cycle detector, drift, hotspots, etc.)
- all enterprise/partner rulepacks
- all future Phase-2 rulepack families
- the engine's rulepack loader
- the `analyzerVersion` and `rulepackVersion` bump process

This contract does NOT cover:

- schema evolution (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- envelope structure (see [Envelope Format Spec](./15-envelope-format-spec.md))
- snapshot canonicalization (see [Repo Snapshot Contract](./14-repo-snapshot-contract.md))
- deterministic engine core behavior (see [Determinism Contract](./07-determinism-contract.md))

---

## 3. Definitions & Terms

**Rulepack**  
A pure, deterministic module that analyzes a `RepoSnapshot` and produces structured data under `extensions.<namespace>.*`.

**RulepackVersion**  
A semantic version (`major.minor.patch`) tied ONLY to the rulepack's behavior—NOT schema changes.

**Namespace**  
The unique identifier for a rulepack's extension fields (e.g., `cycles`, `drift`, `enterprise.acme-security`).

**Extensions**  
The flexible portion of the Envelope where rulepacks append their results.

**AnalyzerVersion**  
Global version of the engine's semantic behavior.

---

## 4. Rules / Contract

### 4.1 Rulepack Semantic Versioning Rules

**Version Format**

```
rulepackVersion: "MAJOR.MINOR.PATCH"
```

**MAJOR version bump required when:**

- output schema changes
- meaning of fields changes
- rulepack namespace changes (forbidden, but if forced → major bump)
- output becomes incompatible with previous versions

**MINOR version bump required when:**

- output gains new optional fields
- new analysis logic is added but does not break consumers

**PATCH version bump required when:**

- internal logic fixes that do not change output shape or meaning

**RulepackVersion MUST NOT change for:**

- engine refactors
- cycle ordering improvements (unless rulepack-specific)
- performance enhancements that preserve semantics

### 4.2 Canonical Rulepack Namespace Governance (NEW — REQUIRED)

Every rulepack MUST declare a canonical namespace:

```
extensions.<namespace>.*
```

**Namespace Rules**

Namespaces MUST:

- be lowercase
- use kebab-case
- be deterministic
- be permanent once created
- never be renamed
- never collide with another namespace
- be validated in CI
- be used consistently across all analyzer versions

**Forbidden:**

- snake_case
- CamelCase
- dots inside namespace except for enterprise namespaces
- renaming namespaces after creation

**Enterprise Namespace Rule (Reserved for Doc #18)**

Enterprise rulepacks MUST use:

```
extensions.enterprise.<vendor>.*
```

**Examples:**

```
extensions.enterprise.acme-security.*
extensions.enterprise.bigbank.*
```

This prevents namespace collisions across customers and partners.

### 4.3 Rulepacks MUST Be Pure Functions (NEW — REQUIRED)

Rulepacks MUST behave like:

```
output = rulepack(snapshot, config)
```

**Rulepacks MUST NOT:**

- mutate `RepoSnapshot`
- mutate `AnalyzerConfig`
- retain caches between runs
- use module-level mutable state
- depend on previous rulepack outputs
- read global state
- modify or reorder extension data from other rulepacks
- store analysis results across runs

**Rulepacks MUST:**

- produce deterministic results
- operate solely on inputs
- use no environment data
- use no timestamps
- be fully idempotent

This ensures deterministic replays, caching, and shadow-analyzer correctness.

### 4.4 Deterministic Rulepack Loading & Execution Order (NEW — REQUIRED)

**Discovery Rules**

Rulepacks MUST be discovered by the engine in:

- lexicographic order of namespace

**Execution Order**

Rulepacks MUST execute in:

- lexicographic order of namespace

**Example:**

```
cycles
drift
hotspots
security-lint
```

**Consequences of nondeterministic ordering**

If ordering is not fixed:

- envelopes differ across runs
- signatures break
- goldens fail
- shadow analyzer diffing becomes impossible

**Therefore: execution order MUST be deterministic.**

**Namespace Collision Handling**

If two rulepacks declare the same namespace:

➡️ CI MUST fail  
➡️ Analyzer MUST refuse to load

This prevents ambiguous extension-field ownership.

### 4.5 Deterministic Output Requirements

Rulepacks MUST:

- output ONLY JSON-serializable data
- never include timestamps
- never include host/environment data
- `deepSort` all objects before returning
- produce stable array ordering
- avoid floating-point nondeterminism
- use stable hashing for any fingerprinting

All rulepack output MUST remain deterministic across:

- machines
- OS
- Node versions
- execution times

### 4.6 Envelope Interaction Rules

Rulepack output MUST be placed only under:

```
extensions.<namespace>.*
```

**Rulepacks MUST NOT:**

- write to `core`
- write to `meta`
- modify `identity`
- modify `version` fields

**Rulepacks MAY:**

- add new optional fields within their own namespace
- evolve within semantic versioning rules

### 4.7 Shadow Analyzer Compatibility (Phase 2)

Shadow analyzer MUST load:

- the same rulepack list
- the same rulepack versions
- in the same deterministic order

Sentinel MUST:

- compare rulepack output by namespace
- allow new fields only if minor version increases
- block promotions on incompatible changes

### 4.8 Rulepack Promotion Rules

A rulepack MAY be promoted to new versions only if:

- golden tests pass
- no namespace collisions exist
- deterministic output is maintained
- `analyzerVersion` bump is performed if required
- shadow analyzer results match expectations

---

## 5. Determinism Impact

This document interacts directly with:

- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)

Rulepacks are an extension mechanism that MUST never break determinism.

If a rulepack introduces nondeterminism, it becomes a **Critical determinism violation**.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- NEVER load rulepacks
- NEVER interpret rulepack output
- NEVER modify rulepack namespaces
- NEVER branch on rulepack versions

**Engine MUST:**

- load rulepacks in deterministic order
- isolate them from runtime concerns
- treat rulepack output as immutable

---

## 7. Versioning Impact

- `RulepackVersion` controls semantic evolution of rulepack outputs.
- `AnalyzerVersion` MUST bump if rulepack behavior changes global semantics.
- `SchemaVersion` MUST NOT bump unless structural envelope changes occur.

---

## 8. Testing Requirements

**Rulepack Tests MUST validate:**

- deterministic output
- purity (no mutation)
- namespace correctness
- rulepack version bump correctness
- adapter compatibility (if adding new structured outputs)

**Golden Tests MUST validate:**

- rulepack outputs remain stable
- stable lexicographic ordering
- signature stability

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Repo Snapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Enterprise Extension Namespaces](./18-enterprise-extension-namespaces.md) (reserved)

---

## 10. Change Log (Append-Only)

**v1.0.0** — Initial creation with namespace governance, purity guarantees, deterministic ordering, and Phase-2 compatibility rules.

