# ArcSight Schema Evolution & Adapter Governance (FINAL VERSION)

This document defines how the ArcSight Envelope schema evolves, how backward compatibility is maintained, and how all historical envelopes are safely normalized into a single internal canonical schema.

It applies to:

- `arcsight-wedge` (schema producer)
- `arcsight-github-app` (schema consumer)
- `arcsight-cli`
- `arc-sentinel` (Phase 2)
- `arc-dashboard` (Phase 2)

Schema evolution is one of ArcSight's core architectural invariants.

---

## 0. Canonical Schema Definition (Pointer)

The authoritative schema lives in:

```
/arcsight-wedge/src/types/Envelope.ts
```

**ALL schema changes MUST start by modifying this file.**

---

## 1. Schema Evolution Goals

ArcSight schema evolution must:

- Maintain perpetual backward compatibility
- Normalize all stored envelopes into `CurrentSchemaEnvelope`
- Preserve deterministic, canonical output
- Support Phase 2 rulepack growth
- Support shadow-mode analyzer rollouts
- Avoid schema proliferation and branching logic
- Keep core stable and evolve safely under extensions

---

## 2. What Triggers a Schema Version Bump

### 2.1 Schema bump REQUIRED when:

- A required field is added to `core`/`identity`/`meta`
- A field changes meaning or type
- A field is removed or renamed
- Envelope structure changes in non-compatible ways
- Signature semantics change
- Core cycle or graph structures are redefined
- Extensions become required

### 2.2 Schema bump NOT required when:

- Adding new optional fields under `extensions`
- Adding optional metadata under `meta`
- Adding new rulepack outputs (as long as optional)
- Updating documentation
- Analyzer semantic updates that do not change structure

**Schema versioning governs structure, not semantics.**

---

## 3. Schema Versioning Rules

Schema version:

- MUST be a positive integer
- MUST increment by exactly +1 on bump
- MUST never decrease
- MUST NOT encode ranges
- MUST NOT depend on analyzer or rulepack versions

- **Analyzer version** = semantic behavior.
- **Rulepack version** = modular analysis unit.
- **Schema version** = envelope shape.

---

## 4. Adapter Chain Contract

Adapters convert:

```
OldSchemaEnvelope → CurrentSchemaEnvelope
```

So that no consumer ever branches on schema versions.

### 4.1 NEW — Consumers MUST NOT branch on raw schema version

Consumers may **NOT** write:

```typescript
if (env.version.schema === 1) { ... }
```

**All consumers MUST assume:**

Envelope has already passed through the full adapter chain.

This prevents code divergence.

### 4.2 Chain Model

Adapters MUST be contiguous:

```
v1 → v2 → v3 → … → vN
```

❌ **NOT permitted:** `v1→v3`  
✔ **REQUIRED:** `v1→v2 → v2→v3`

### 4.3 Adapter Requirements (UPDATED with new rules)

Adapters MUST:

- Be pure, deterministic, idempotent
- Handle exactly one version step
- Deep-sort objects before returning
- Preserve all known fields
- **Preserve unknown extension fields exactly as-is (NEW)**
- **Operate only on schema-visible fields (NEW)**
- **Maintain deterministic field ordering (NEW)**
- Use no I/O, network, env vars, randomness
- Produce a full `CurrentSchemaEnvelope`

#### Preservation of unknown extension fields (NEW — CRITICAL)

Adapters MUST:

- Leave all unknown fields under `extensions.*` untouched
- Preserve shape, casing, and values exactly
- Never drop, rename, reorder, or transform unknown extension fields

This ensures forward compatibility for custom enterprise rulepacks.

#### Operate only on schema-visible fields (NEW — CRITICAL)

Adapters MUST NOT:

- Read or modify fields not specified in `/types/Envelope.ts`
- Introduce internal engine/runtime-only fields
- Normalize or adapt runtime artifacts accidentally included in payloads

This prevents schema creep, security leaks, and nondeterministic contamination.

#### Maintain deterministic field ordering (NEW — HIGH IMPORTANCE)

Adapters MUST:

- Return envelopes whose top-level field ordering matches the deterministic ordering used by:
  - `buildCoreEnvelope()`
  - `signEnvelope()`

This ensures:

- Stable diffs
- Snapshot consistency
- Golden test reliability

### 4.4 Adapter Registry Example

```typescript
interface SchemaAdapter {
  fromVersion: number;
  toVersion: number;
  upgrade(env: any): any;
}

export const SCHEMA_ADAPTERS = [
  adapter_v1_to_v2,
  adapter_v2_to_v3,
  ...
];
```

### 4.5 Envelope Read Path

```typescript
let e = rawEnvelope;

while (e.version.schema < CURRENT_SCHEMA_VERSION) {
    e = applyAdapter(e.version.schema, e);
}

return e; // CurrentSchemaEnvelope
```

---

## 5. Schema Stability Rules

### 5.1 Core Section Stability

`core.*` must remain structurally stable.  
Require schema bump for any modification.

### 5.2 Identity Section Stability

`identity.*` is runtime-provided metadata.  
Adapters must preserve these fields unchanged.

### 5.3 Extensions Are the Growth Zone

New features MUST live under:

```json
"extensions": { ... }
```

Adding optional extension fields does **NOT** require schema bump.

Unknown extension fields MUST be preserved.

### 5.4 Meta Section Stability

`meta.*` fields must remain stable. Optional additions are allowed; required ones are not.

---

## 6. Determinism Requirements for Adapters

Adapters MUST comply with the Determinism Contract:

- No timestamps
- No randomness
- No environment-derived fields
- No nondeterministic sorting
- No algorithmic dependency on object insertion order

Adapters MUST use:

```
deepSort → stableStringify
```

before returning output.

---

## 7. Adapter Testing Requirements

### 7.1 Fixture Tests

Example:

```
/fixtures/schema_v1/envelope.json  
/fixtures/schema_v2/envelope.json
```

Each MUST:

- Upgrade to `CurrentSchemaEnvelope`
- Match canonical expected output exactly

### 7.2 Idempotence

```
upgrade(upgrade(env)) === upgrade(env)
```

### 7.3 Golden Stability

Golden tests MUST pass after schema upgrades.  
`analyzer_version` bump required only if envelope semantics change.

### 7.4 Cross-environment reproducibility

Adapters MUST behave identically across:

- Linux
- macOS
- Node versions

---

## 8. Developer Workflow for Schema Changes

To change schema:

1. Update `/types/Envelope.ts`
2. Add new adapter (`vN→vN+1`)
3. Add fixtures for old schema
4. Add adapter unit tests
5. Update documentation
6. Update golden tests (if semantics change)
7. Bump `schemaVersion.ts`
8. Bump `analyzerVersion.ts` (if semantics change)

**No schema change may skip this.**

---

## 9. Interaction with Analyzer & Rulepack Versions

- Schema ≠ analyzer version
- Schema ≠ rulepack version
- Schema only changes when structure changes
- Analyzer version changes when semantics change
- Rulepack version changes per-rulepack

---

## 10. Interaction With Shadow-Mode Analyzer Rollouts (Phase 2)

Shadow analyzer outputs MUST use current schema only.

Runtime MUST always:

- Upgrade both live + shadow envelopes
- Compare normalized envelopes
- Detect drift or unexpected changes

This ensures safe analyzer rollouts without customer impact.

---

## 11. Deprecation Policy

To remove a field:

1. Mark deprecated
2. Adapter migrates old field to new location
3. Field continues reading indefinitely
4. Remove only after schema bump

ArcSight never loses historical meaning.

---

## 12. Optional Semantic Collapse Invariant (NEW)

ArcSight MUST guarantee:

**No two semantically different envelopes may adapt into the same canonical envelope shape.**

This prevents:

- accidental semantic loss
- incompatible historical data merging
- silent adapter corruption

This rule ensures semantic preservation over time.

---

## 13. Summary

ArcSight schema evolution is governed by these invariants:

- Always backward compatible
- All envelopes normalize to one canonical shape
- Adapters are pure, deterministic, idempotent
- Unknown extension fields preserved
- Only schema-visible fields are touched
- Deterministic field ordering maintained
- No consumer branches on schema version
- Core stays stable; extensions grow freely
- Schema bumps only for structural changes
- Shadow analyzer rollouts rely on schema normalization

This document is binding across engine, runtime, CLI, sentinel, and dashboard.

---

## Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Dependency Contract](./5A-dependency-contract.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added Change Log section and standardized Cross-References heading.

**v1.0.0** — Initial version.

