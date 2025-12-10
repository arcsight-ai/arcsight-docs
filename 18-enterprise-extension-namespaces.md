# Enterprise Extension Namespaces (FINAL, Phase 2)

## 1. Purpose

This document defines the namespace rules, stability guarantees, structural constraints, and compatibility requirements for enterprise/partner rulepacks that write output under:

```
extensions.enterprise.<vendor>.<pack>.*
```

The goal is to ensure:

- deterministic, isolated enterprise extensions
- no namespace collisions
- no schema drift
- no unintended interaction with core or other rulepacks
- consistent behavior across ArcSight upgrades
- safe long-term compatibility in multi-tenant environments

Enterprise rulepacks enable external vendors or internal teams to add their own analysis logic without affecting the core engine.

---

## 2. Scope

### Included

This document governs:

- enterprise rulepack namespace format
- determinism requirements
- schema stability rules
- extension growth rules
- runtime/engine boundary behavior
- vendor isolation
- adapter compatibility
- JSON-serialization rules

### Excluded

This document does not define:

- Rulepack versioning semantics → see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- Schema evolution → see [Schema Evolution Contract](./11-schema-evolution-contract.md)
- Envelope structure → see [Envelope Format Spec](./15-envelope-format-spec.md)
- Determinism rules → see [Determinism Contract](./07-determinism-contract.md)
- Drift classification → see [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)

---

## 3. Definitions & Terms

**Enterprise Rulepack**  
A deterministic rulepack maintained by a vendor or enterprise customer.

**Vendor Namespace**  
The `<vendor>` segment in:

```
extensions.enterprise.<vendor>.<pack>
```

**Pack Namespace**  
The `<pack>` segment—defines one logical rulepack.

**Extension Fields**  
Output values placed under the enterprise namespace.

**Schema Adapter**  
Transforms old envelopes → current schema without mutating enterprise fields.

---

## 4. Rules / Contract

### 4.1 Namespace Format (UPDATED)

Enterprise rulepacks MUST use:

```
extensions.enterprise.<vendor>.<pack>
```

Where:

- `<vendor>` = lowercase, kebab-case
- `<pack>` = lowercase, kebab-case

**MANDATORY rule:**

Enterprise namespaces MUST contain exactly two segments after `enterprise`:

✔ `extensions.enterprise.acme.security`  
✘ `extensions.enterprise.acme.security.insights` (FORBIDDEN)

Nested pack hierarchies are forbidden.

**Rationale:**  
Prevents namespace sprawl and simplifies adapter logic & deterministic ordering.

### 4.2 Namespace Ownership

- `<vendor>` uniquely identifies a vendor.
- `<pack>` uniquely identifies a rulepack under that vendor.
- CI MUST enforce no collisions.
- A namespace, once created, is permanent.
- Renaming namespaces is forbidden.

### 4.3 Enterprise Rulepacks MUST Be Deterministic

Enterprise rulepacks MUST:

- produce deterministic output
- have stable ordering of all fields and arrays
- sort keys lexicographically
- use `stableStringify`-compatible JSON values
- operate under the same sandbox & limits as core rulepacks

They MUST NOT:

- use timestamps
- use randomness
- depend on environment variables
- depend on ArcSight internal state outside the snapshot

### 4.4 Isolation Requirements

Enterprise rulepacks:

- MUST NOT depend on the output of other rulepacks
- MUST NOT mutate extensions outside their namespace
- MUST NOT read internal engine state
- MUST NOT influence `core.*` or `meta.*`

ArcSight MUST treat enterprise extension data as opaque.

### 4.5 Vendor Independence From ArcSight Release Cadence (NEW)

Enterprise rulepacks:

- MUST NOT depend on ArcSight `analyzerVersion` upgrade schedule
- MUST continue functioning as long as `schemaVersion` remains compatible
- MUST NOT branch logic based on specific `analyzerVersion` values
- MUST NOT assume fixed timing of ArcSight releases

This prevents tight coupling and brittle vendor integrations.

### 4.6 Schema Evolution Compatibility (UPDATED)

Enterprise rulepacks MUST NOT:

- remove historical fields
- rename fields
- change field meaning
- mutate previously emitted values
- break adapters by altering structure retroactively

All historical fields MUST be preserved to ensure:

- drift stability
- long-term analytics
- backward compatibility

Adapters MUST preserve unknown enterprise extensions exactly as-is (see [Schema Evolution Contract](./11-schema-evolution-contract.md) & [Envelope Format Spec](./15-envelope-format-spec.md)).

### 4.7 Forbidden Behaviors (UPDATED)

Enterprise rulepacks MUST NOT:

- depend on specific ArcSight `analyzerVersion` or `schemaVersion` cadence
- emit nondeterministic data
- use Maps, Sets, Dates, BigInts, or non-JSON-serializable types
- emit buffers, functions, Promises, or class instances
- interact with runtime APIs or environment variables
- create dynamic namespaces
- include generated timestamps without explicit approval

These break determinism, signatures, or adapter stability.

### 4.8 Allowed Behaviors (UPDATED)

Enterprise rulepacks MAY:

- add new optional fields inside their namespace
- add new sub-objects (lower camel-case keys)
- version themselves independently via `rulepackVersion`
- extend output structure as their pack evolves
- expose JSON-serializable values ONLY

**Allowed values:**

✔ strings  
✔ numbers  
✔ booleans  
✔ null  
✔ arrays  
✔ plain objects  
✔ stable-stringifiable structures

**Forbidden values:**

✘ Maps  
✘ Sets  
✘ BigInt  
✘ Dates  
✘ NaN / Infinity  
✘ Undefined  
✘ Class instances  
✘ Functions  
✘ Buffers

All values MUST serialize identically across:

- OS
- Node versions
- runtimes

---

## 5. Determinism Impact

Enterprise rulepacks contribute to Envelope signatures.

Therefore:

- ANY nondeterminism in enterprise values breaks signature stability
- deterministic replay MUST work across machines
- shadow analyzer comparisons MUST remain stable
- Sentinel drift MUST not be polluted by vendor nondeterminism

Enterprise extensions are part of the deterministic boundary.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- treat enterprise extension fields as opaque
- NEVER mutate enterprise extension output
- NEVER strip or normalize vendor fields
- NEVER impose ordering changes
- NEVER fix correctness errors

**Engine MUST:**

- load enterprise rulepacks deterministically
- isolate them from internal engine data
- enforce identical limits & sandbox rules ([Limits & Sandbox Policy](./13-limits-and-sandbox-policy.md))
- ensure enterprise outputs remain JSON-serializable

---

## 7. Versioning Impact

Enterprise rulepacks MUST follow [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md):

- Patch → fixes
- Minor → additive, non-breaking
- Major → backward-incompatible changes

- `schemaVersion` MUST NOT bump due to enterprise rulepacks.
- `analyzerVersion` MUST bump when enterprise rulepacks affect deterministic fields.
- `snapshotVersion` MUST NOT depend on enterprise rulepacks.

---

## 8. Testing Requirements

### 8.1 Determinism Tests

Each enterprise rulepack MUST test:

- identical output across multiple runs
- identical output across OSes
- stable ordering of keys
- stable array ordering

### 8.2 Serialization Tests

Test that all extension fields:

- are JSON serializable
- match `stableStringify`
- contain no forbidden types

### 8.3 Schema Adapter Tests

Adapters MUST:

- preserve unknown enterprise fields
- not mutate or drop vendor data
- pass round-trip compatibility tests

### 8.4 Drift Tests

Sentinel MUST assert:

- no unexpected enterprise field drift
- no removal of historical fields
- no schema regressions

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)
- [Analyzer Promotion Policy](./17-analyzer-promotion-policy.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added improvements:

- Explicit rule banning dependency on `analyzerVersion` cadence
- Namespace depth constraint (vendor + pack only)
- Enterprise rulepacks MUST NOT mutate or delete historical fields
- JSON-serialization stability rules

**v1.0.0** — Initial version of Enterprise Namespace Contract

