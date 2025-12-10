# Schema Evolution Contract (Final, Updated Phase 1 & Phase 2)

## ArcSight Envelope Format Evolution Rules & Adapter Pipeline

Schema evolution is one of the highest-risk areas in any static-analysis system.  
The Envelope schema defines the public, long-lived structure of ArcSight's analysis output.

This contract guarantees:

- deterministic reproducibility
- backward compatibility
- safe forward migration
- adapter-driven evolution
- structural stability
- long-term viability of dashboards, drift detectors, and enterprise integrations

It ensures ArcSight's schema remains stable for 10+ years of evolution.

---

## 0. Envelope Shape Contract (NEW SECTION)

To establish clarity from the start, the Envelope has a frozen structural shape:

### Envelope Shape Contract

The following top-level sections are structurally frozen and cannot change without a schema version bump:

- `version.*`
- `identity.*`
- `core.*`
- `extensions.*` (optional fields may grow freely; requiredness cannot change without bump)
- `meta.*`

**Rules:**

- Fields may be added only under `extensions.*` without a schema bump.
- Fields may NOT be removed, moved, or made required without a schema bump.
- The meaning of stable fields must remain identical across versions unless `schemaVersion` increments.

This summary prevents schema misuse by future contributors.

---

## 1. Purpose of This Contract

This document ensures:

✔ Schema changes are deliberate, safe, and rare  
✔ Historical envelopes remain valid forever  
✔ Dashboards and sentinel always consume a single normalized shape  
✔ Upgrading analyzers never breaks stored data  
✔ Most evolution occurs under `extensions.*` instead of the core schema  
✔ Determinism is maintained across all versions  

---

## 2. Schema Versioning Rules

### 2.1 Schema version is a global integer

```
schema_version: 1
schema_version: 2
schema_version: 3
...
```

No semantic versioning.  
No major/minor/patch semantics.  
`SchemaVersion` is purely structural.

### 2.2 Schema version increments ONLY when structure changes

- required fields added
- required fields removed
- field semantics change
- structured section added/removed
- `core`/`meta`/`identity` structure shifts

### 2.3 Schema version does NOT increment for:

- analyzer behavior changes
- rulepack output changes
- cycle detection improvements
- ordering improvements

Those require `analyzerVersion`, NOT `schemaVersion`.

### 2.4 Schema Version MUST NOT encode semantics (NEW RULE)

Schema version:

- is structural, not semantic
- MUST NOT encode analyzer rules
- MUST NOT encode rulepack versions
- MUST NOT mirror analyzer lifecycle
- MUST NOT change when semantics change but structure does not

Semantic meaning belongs exclusively to:

- `version.analyzer`
- `version.rulepack`

This prevents the "semantic schema drift" problem that affects many legacy analyzers.

---

## 3. Backward Compatibility Rules

ArcSight guarantees:

**All historical envelopes remain usable forever.**

This is achieved via a deterministic adapter chain that upgrades old schemas into the current canonical schema.

Thus:

- No reprocessing historical data
- No forked code paths in dashboards
- No per-schema branching
- No consumer divergence

Schema compatibility is engine-owned and cannot be delegated.

---

## 4. Schema Adapter Architecture

### 4.1 Adapter Chain MUST be contiguous

Each adapter performs exactly:

```
vN → vN+1
```

**NEW RULE:**  
Adapters MUST NOT skip versions (e.g., `v1→v3`).  
Every schema change MUST have a corresponding incremental adapter.

This guarantees clarity and future maintainability.

### 4.2 Adapter Responsibilities

Each adapter MUST:

- transform envelope cleanly to next schema
- preserve all existing fields
- preserve field ordering
- maintain determinism
- never remove unknown fields
- ensure structural validity of output

### 4.3 Adapters MUST preserve unknown extension fields (CRITICAL RULE)

Adapters MUST NOT:

- drop
- rename
- reorder
- transform

any unknown fields under `extensions.*`.

This enables:

- enterprise extensions
- partner rulepacks
- customer-specific rulepacks
- Phase 2 modular rulepack families

### 4.4 Adapters MUST operate ONLY on schema-declared fields (NEW RULE)

To prevent schema creep:

**Adapters MUST restrict transformations to schema-declared fields only.**  
Unknown or out-of-schema fields MUST NOT be touched.

### 4.5 Adapters MUST maintain deterministic ordering

Adapter output MUST respect:

- canonical field ordering rules
- stable stringify invariants
- deterministic nested object ordering

Ordering drift breaks:

- signatures
- goldens
- sentinel drift detection

### 4.6 Adapters MUST be pure and deterministic

Adapters must avoid:

- clocks
- randomness
- async nondeterminism
- environment variables
- machine state

They MUST produce identical output for identical input.

---

## 5. Adapter Invocation Rules

**Engine MUST:**

- Detect schema of incoming envelope.
- Apply adapters in order until reaching `CURRENT_SCHEMA_VERSION`.
- Produce a single `CurrentSchemaEnvelope` result.

**Runtime MUST NOT:**

- apply adapters
- branch on `schemaVersion`
- rewrite `schemaVersion`
- mutate envelopes

**Sentinel (Phase 2) MUST:**

- compare only normalized envelopes
- validate adapter chain correctness
- detect cross-version drift

---

## 6. What Requires a Schema Version Bump

`SchemaVersion` MUST bump when:

### 6.1 Adding new required fields

### 6.2 Removing required fields

### 6.3 Changing semantics of existing fields

### 6.4 Moving fields between sections

### 6.5 Changing structured representations

### 6.6 Changing extension requiredness

(e.g., extension becomes required)

### 6.7 Schema Drift Burden of Proof Requirement (NEW RULE)

**Any proposed schema bump MUST provide:**

- A written justification
- Explanation for why it cannot be placed under `extensions.*`
- A migration plan
- New golden tests
- New adapter logic

**Default answer:**

Put it under `extensions.*` unless absolutely required.

This prevents long-term schema bloat and uncontrolled structural creep.

---

## 7. What Does NOT Require a Schema Version Bump

- Adding optional extension fields
- Enhancing rulepack logic
- Changing analyzer semantics
- Changing cycle detection algorithm (if deterministic)
- Graph stats additions under extensions
- Adding new optional extension namespaces

**Anything optional or domain-specific belongs under `extensions.*`.**

---

## 8. Golden Tests & Schema Evolution

Goldens validate:

- adapter correctness
- deterministic normalization
- structural invariants
- versioned envelope integrity
- signature stability

When `schemaVersion` increments, goldens MUST be updated and `analyzerVersion` MUST be bumped.

Goldens MUST contain historical envelopes across versions.

---

## 9. Deterministic Requirements for Schema Evolution

Schema evolution MUST NOT:

- introduce nondeterministic fields
- add timestamps
- add environment-dependent data
- break envelope ordering
- collapse semantically distinct envelopes into identical forms
- cause non-invertible transformations

---

## 10. Runtime & Schema Evolution

Runtime MUST NOT:

- mutate schemas
- fix-up envelopes
- remove fields
- guess meanings of `schemaVersion`
- override or modify schema-driven fields
- drop extension fields
- inject timestamps
- implement fallback analyzers

Runtime is a transport layer only.

---

## 11. Sentinel & Schema Evolution (Phase 2)

Sentinel MUST:

- compare normalized envelopes only
- validate adapter chains
- detect drift across analyzer versions
- block promotion if adapters misbehave
- ensure semantic compatibility before rollout

Sentinel operates only on `CurrentSchemaEnvelope`.

---

## 12. Schema Freeze Windows (Optional)

Organizations MAY implement:

No schema changes between Fridays 12:00 UTC → Mondays 12:00 UTC.

Useful for enterprise stability but optional.

---

## 13. Forward Compatibility Invariants

These invariants guarantee the schema remains evolvable forever.

### 13.1 No semantic collapse

Adapters MUST NOT collapse two semantically different envelopes into identical shapes.

### 13.2 Unknown fields preserved forever

Ensures extensibility.

### 13.3 Structural validity guaranteed

No missing required fields, even in error envelopes.

### 13.4 Adapter chain permanence

Adapters MUST NEVER be removed.

---

## 14. Summary

✔ `SchemaVersion` = structure version, NOT semantic version  
✔ `AnalyzerVersion` handles semantic changes  
✔ `RulepackVersion` handles rule logic  
✔ Envelope shape is structurally frozen  
✔ Extensions provide safe evolution paths  
✔ Adapters guarantee backward compatibility  
✔ Runtime is schema-agnostic  
✔ Sentinel validates schema evolution in Phase 2  
✔ No schema drift, no breakage, no regressions  

ArcSight's schema evolution model is now hardened for long-term stability.

---

## Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Dependency Contract](./5A-dependency-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added Change Log section and standardized Cross-References heading.

**v1.0.0** — Initial version.

