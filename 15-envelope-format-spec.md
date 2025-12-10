# Envelope Format Specification (FINAL UPDATED VERSION, Phase 1)

## 1. Purpose

This document defines the canonical Envelope format, the deterministic output of the ArcSight engine (`arcsight-wedge`).

The Envelope is the:

- sole output of the analyzer
- contract consumed by runtime, sentinel, dashboards, drift detectors
- long-lived API surface supporting backward and forward compatibility
- canonical artifact validated by golden tests

This specification ensures:

- deterministic envelope construction
- stable structural schema
- compatibility across schema versions
- drift detection reliability
- safety for future rulepack and shadow-analyzer expansion

---

## 2. Scope

This document governs:

### Included

- Envelope top-level shape
- Required, optional, extension fields
- Deterministic ordering rules
- Signature and hashing rules
- Runtime vs engine responsibilities
- Forbidden nondeterministic fields
- Canonical handling of `generated_at`
- Compatibility rules for schema adapters

### Not Included

- Schema evolution (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- Rulepack versioning (see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- Snapshot definition (see [RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- Determinism rules (see [Determinism Contract](./07-determinism-contract.md))
- Engine/runtime boundary (see [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))

---

## 3. Definitions & Terms

**Envelope**  
Deterministic output of the engine.

**CurrentSchemaEnvelope**  
Envelope after schema adapters, always normalized to latest schema version.

**Signature**  
A SHA-256 hash uniquely identifying the deterministic content of the envelope.

**Extensions**  
Namespaced rulepack outputs that may grow indefinitely without schema bumps.

**SchemaVersion**  
Strict integer defining envelope structure (v1, v2, v3…).

---

## 4. Rules / Contract

### 4.1 Top-Level Envelope Shape (Canonical)

Every Envelope MUST match this exact top-level structure:

```typescript
interface Envelope {
  version: {
    analyzer: string;  // semver e.g. "1.3.0"
    schema: number;    // integer, e.g. 1
    rulepack: Record<string, string>; // namespace → version
  };

  identity: {
    installation_id: string;
    org_id: string;
    repo_id: string;
    pr_number: number | null;
    sha: string;
    analysis_id: string;
    request_id: string;
  };

  core: {
    graph_stats: {
      node_count: number;
      edge_count: number;
    };
    cycles: Array<Array<string>>;   // canonical ordering
    status: "success" | "degraded" | "error";
    degraded_reason?: string;
    error_code?: string;
  };

  extensions: {
    [namespace: string]: any; // namespaced rulepack outputs
  };

  meta: {
    snapshot_version: number;    // from RepoSnapshot
    repo_fingerprint: string;    // from RepoSnapshot
    signature: string;           // lowercase hex, 64 chars
    generated_at: string;        // ISO timestamp (runtime ONLY)
  };
}
```

No additional top-level fields are allowed.

### 4.2 Envelope Shape Contract Summary Table (NEW)

| Section | Deterministic? | Produced by | Notes |
|---------|----------------|-------------|-------|
| `version.*` | YES | Engine | Structural + semantic versions |
| `identity.*` | YES | Runtime | Immutable, opaque to engine |
| `core.*` | YES | Engine | Deterministic graph + cycle output |
| `extensions.*` | YES | Engine rulepacks | Namespaced; may grow without bump |
| `meta.snapshot_version` | YES | RepoSnapshot | Deterministic integer |
| `meta.repo_fingerprint` | YES | RepoSnapshot | Deterministic hash |
| `meta.signature` | YES | Engine | Deterministic, excludes generated_at |
| `meta.generated_at` | NO | Runtime | Only nondeterministic field; excluded from signature |

### 4.3 Deterministic Field Ordering

Field ordering in the Envelope MUST follow:

1. `version`
2. `identity`
3. `core`
4. `extensions`
5. `meta`

Within each object:

- keys MUST be sorted lexicographically
- arrays MUST use stable deterministic ordering
- cycles MUST be sorted following cycle-rooting rules ([Determinism Contract](./07-determinism-contract.md))

Breaking ordering = breaking determinism.

### 4.4 Extension Namespace Rules

Extensions MUST follow:

- lowercase kebab-case for namespace names
- each namespace corresponds to one rulepack
- no collisions allowed (CI failure)
- extension fields may grow indefinitely without schema bump
- extension fields MUST be deterministic
- extension fields MUST NOT depend on runtime or environment
- extension fields MUST NOT contain timestamps, random values, or nondeterministic ordering

**Critical: Preservation Rule (NEW)**

During schema evolution, adapters:

- MUST preserve unknown extension namespaces exactly
- MUST NOT rename, drop, reorder, or transform unknown extension fields
- MUST NOT assume meaning of fields from other rulepacks

This is essential for Phase 2 multi-analyzer compatibility and enterprise rulepack safety.

### 4.5 Signature Rules (UPDATED)

Signature = deterministic hash of the envelope content.

Signature MUST:

- use SHA-256
- be lowercase hexadecimal
- be exactly 64 characters long
- be computed AFTER adapter normalization
- be computed using `stableStringify`
- be deterministic across machines, OSes, Node versions

Signature MUST NOT include:

- `meta.generated_at`
- any runtime timestamps
- non-deterministic rulepack fields
- nondeterministic ordering

The signature canonically identifies the meaning of the envelope.

### 4.6 meta.generated_at Determinism (UPDATED)

`generated_at` MUST:

- be generated by the runtime
- be an ISO-8601 string
- NOT be deterministic across runs
- NOT be included in signatures
- NOT affect drift detection
- NOT appear in golden tests
- be the only allowed nondeterministic field in the entire Envelope

Engine MUST NOT generate timestamps.

### 4.7 Core Section Rules

The core fields MUST be:

- deterministic
- stable across OS
- validated by golden tests
- complete even in error mode

cycles MUST use deterministic cycle-rooting from [Determinism Contract](./07-determinism-contract.md).

### 4.8 Forbidden Fields (UPDATED)

The Envelope MUST NOT contain:

- timestamps in any field except `meta.generated_at`
- environment info
- machine identifiers
- absolute file paths
- nondeterministic arrays
- unordered maps
- random values
- rulepack version inference fields

If present, runtime MUST block the envelope.

### 4.9 Error & Degraded Envelope Rules

Even in error mode, envelope MUST contain:

- `version.*`
- `identity.*`
- `core.status = "error"`
- `meta.snapshot_version`
- `meta.repo_fingerprint`
- `meta.signature` (computed over error envelope content)

Missing required fields makes the envelope invalid.

---

## 5. Determinism Impact

The Envelope is the deterministic contract of ArcSight.

Breaking envelope determinism breaks:

- golden tests
- signature stability
- drift detection
- shadow analyzer comparison
- rulepack consistency
- schema adapter correctness
- long-term analytics

This spec ensures Envelope determinism for the next 10+ years.

---

## 6. Runtime / Engine Boundary Impact

**Engine MUST:**

- produce deterministic envelopes
- compute signature
- exclude `generated_at` from signature
- apply schema adapters
- preserve identity fields
- treat `snapshot_version` & `repo_fingerprint` as authoritative

**Runtime MUST:**

- provide identity fields
- generate `generated_at`
- NEVER modify envelopes
- NEVER correct or edit envelope content
- NEVER generate signatures
- treat envelope as final

---

## 7. Versioning Impact

**Changes requiring `schemaVersion++`:**

- required fields added
- required fields removed
- meaning of fields changed
- envelope structure changes

**Changes requiring `analyzerVersion++`:**

- semantic behavior changes
- rulepack behavior updates
- cycle or graph changes

**Changes that MUST NOT change `schemaVersion`:**

- new extension fields
- rulepack output additions
- performance improvements
- reordering of rulepack internals

---

## 8. Testing Requirements

### 8.1 Golden Envelope Tests

Goldens MUST verify:

- deterministic ordering
- deterministic cycles
- correct signature behavior
- exclusion of `generated_at`
- adapter-upgraded envelopes
- rulepack extension preservation

### 8.2 Schema Adapter Tests

Adapters MUST be tested to ensure:

- unknown extension preservation
- field order preservation
- deterministic output

### 8.3 Error & Degraded Tests

Error envelopes MUST be:

- structurally valid
- signature-stable
- schema-correct

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

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added:

- signature = lowercase hex requirement
- exclusion of `generated_at` from signature
- full `meta.generated_at` deterministic rules
- preservation of unknown extension namespaces
- envelope shape summary table
- clarified deterministic ordering rules

**v1.0.0** — Initial Envelope v1 definition

