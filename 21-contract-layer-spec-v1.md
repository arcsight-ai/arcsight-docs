# Contract Layer Specification v1

## 1. Purpose

This document defines the canonical data schemas that form the contract layer of ArcSight Phase 1. These schemas are the single source of truth for all data moving through the system:

- Violation Schema v1
- Analyzer Interface v1
- Rulepack Schema v1
- Snapshot Schema v1
- PR Summary JSON Schema

This specification MUST be finalized before any wedge code implementation begins. All contracts, type definitions, and data transformations MUST conform to these schemas.

---

## 2. Scope

This document governs:

### Included

- Violation data structure and classification
- Analyzer plugin interface and execution contract
- Rulepack schema and versioning structure
- Snapshot schema (reference and refinement)
- PR Summary JSON output format
- Type definitions for all contract layer data
- Deterministic serialization rules for all schemas

### Not Included

- Engine implementation details (see [Wedge Architecture Spec](./22-wedge-architecture-spec-v1.md))
- Schema evolution rules (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- Envelope structure (see [Envelope Format Spec](./15-envelope-format-spec.md))
- Runtime-specific formats (see [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))

---

## 3. Definitions & Terms

**Violation**  
A rulepack-detected issue in the analyzed repository. Contains deterministic classification, location, and metadata.

**Analyzer Interface**  
The contract that all rulepack analyzers must implement. Defines input, output, and execution semantics.

**Rulepack Schema**  
The structure defining a rulepack's identity, version, configuration, and analyzer functions.

**Snapshot Schema**  
The canonical RepoSnapshot format (see [RepoSnapshot Contract](./14-repo-snapshot-contract.md) for full specification).

**PR Summary**  
A deterministic JSON summary of analysis results formatted for display in GitHub Check Runs.

---

## 4. Rules / Contract

### 4.1 Violation Schema v1

A Violation represents a single issue detected by a rulepack analyzer.

```typescript
interface Violation {
  // Identity
  id: string;                    // Deterministic UUID v5 based on: rulepack_namespace + rule_id + canonical_location
  rulepack_namespace: string;    // e.g., "cycles", "dependencies", "security"
  rule_id: string;               // Unique within namespace, e.g., "circular-dependency-detected"
  
  // Classification
  severity: "error" | "warning" | "info";
  category: string;              // e.g., "circular-dependency", "security", "performance"
  
  // Location
  file_path: string;             // Canonical POSIX path (from RepoSnapshot)
  line_number: number | null;    // 1-indexed, null if file-level
  column_number: number | null;  // 1-indexed, null if line-level or file-level
  
  // Message
  message: string;               // Deterministic, constant message template
  context?: Record<string, any>; // Additional deterministic context (deep-sorted)
  
  // Metadata
  detected_at: string;           // Fixed "v1" for Phase 1 (no timestamp)
  analyzer_version: string;      // From envelope.version.analyzer
}
```

**Violation Determinism Rules:**

- `id` MUST be deterministic based on rulepack namespace, rule ID, and canonical location
- `message` MUST be a constant template (no variable interpolation in message itself; use `context`)
- `context` MUST be deep-sorted before serialization
- All fields MUST be JSON-serializable
- Violations MUST be sorted deterministically in arrays (by `id` lexicographically)

**Violation ID Generation:**

```
violation_id = UUIDv5(
  namespace: "arcsight-violation",
  name: `${rulepack_namespace}:${rule_id}:${canonical_location_hash}`
)

canonical_location_hash = SHA256(
  stableStringify({
    file_path: canonical_path,
    line_number: line_number || 0,
    column_number: column_number || 0
  })
)
```

### 4.2 Analyzer Interface v1

An Analyzer is a pure function that processes a RepoSnapshot and produces violations and optional extension data.

```typescript
interface AnalyzerInput {
  snapshot: RepoSnapshot;        // Canonical snapshot (see 4.4)
  config: AnalyzerConfig;        // Rulepack-specific configuration
  graph?: Graph;                 // Optional: pre-built dependency graph
}

interface AnalyzerOutput {
  violations: Violation[];       // Sorted by id
  extension_data?: Record<string, any>; // Deterministic, deep-sorted extension output
}

interface AnalyzerConfig {
  enabled: boolean;
  severity_overrides?: Record<string, "error" | "warning" | "info">;
  [key: string]: any;            // Rulepack-specific config (must be deterministic)
}

type Analyzer = (input: AnalyzerInput) => AnalyzerOutput;
```

**Analyzer Contract Rules:**

1. **Purity:** Analyzers MUST be pure functions with no side effects
2. **Determinism:** Same input MUST produce identical output
3. **Isolation:** Analyzers MUST NOT access filesystem, network, or environment
4. **No Time:** Analyzers MUST NOT use timestamps or time-based APIs
5. **No Randomness:** Analyzers MUST NOT use random number generation
6. **Sorted Output:** `violations` MUST be sorted by `id` lexicographically
7. **Deep Sorting:** `extension_data` MUST be deep-sorted before return
8. **Error Handling:** Analyzers MUST NOT throw; return violations with `severity: "error"` instead

**Analyzer Execution Model:**

```
For each rulepack:
  1. Load analyzer function
  2. Validate config
  3. Call analyzer(snapshot, config, graph?)
  4. Collect violations
  5. Collect extension_data
  6. Merge into envelope.extensions[namespace]
```

### 4.3 Rulepack Schema v1

A Rulepack defines a namespace, version, analyzers, and configuration schema.

```typescript
interface Rulepack {
  // Identity
  namespace: string;             // Lowercase kebab-case, e.g., "cycles", "dependencies"
  version: string;               // Semver, e.g., "1.0.0"
  
  // Analyzers
  analyzers: Array<{
    id: string;                  // Unique within rulepack
    name: string;                // Human-readable name
    description: string;         // What it detects
    analyzer_fn: Analyzer;       // The analyzer function
    default_severity: "error" | "warning" | "info";
    requires_graph?: boolean;    // Whether graph is required in input
  }>;
  
  // Configuration
  config_schema?: JSONSchema;    // Optional: JSON Schema for config validation
  default_config: AnalyzerConfig;
}

interface RulepackRegistry {
  [namespace: string]: Rulepack;
}
```

**Rulepack Rules:**

1. **Namespace Uniqueness:** Namespaces MUST be unique across all rulepacks
2. **Version Stability:** Version MUST follow semver; changes require version bump
3. **Analyzer IDs:** Analyzer IDs MUST be unique within the rulepack namespace
4. **Phase 1 Restriction:** Phase 1 includes ONLY built-in core rulepacks; external rulepacks forbidden
5. **Deterministic Execution:** Rulepack execution order MUST be deterministic (sorted by namespace)

**Phase 1 Built-in Rulepacks:**

Phase 1 includes ONLY built-in core rulepacks. External rulepacks are forbidden until Phase 2.

```typescript
// Core rulepack: cycles (Milestone 4)
{
  namespace: "cycles",
  version: "1.0.0",
  analyzers: [
    {
      id: "circular-dependency-detector",
      name: "Circular Dependency Detector",
      description: "Detects cycles in dependency graphs",
      analyzer_fn: detectCircularDependencies,
      default_severity: "error",
      requires_graph: true
    }
  ],
  default_config: { enabled: true }
}

// Core rulepack: drift (Milestones 5-9)
{
  namespace: "drift",
  version: "1.0.0",
  analyzers: [
    {
      id: "drift-detector",
      name: "Drift Detector",
      description: "Detects drift in repository structure and dependencies",
      analyzer_fn: detectDrift,
      default_severity: "warning",
      requires_graph: false
    }
  ],
  default_config: { enabled: true }
}

// Core rulepack: change-impact (Milestones 5-9)
{
  namespace: "change-impact",
  version: "1.0.0",
  analyzers: [
    {
      id: "change-impact-analyzer",
      name: "Change Impact Analyzer",
      description: "Analyzes impact of changes across modules",
      analyzer_fn: analyzeChangeImpact,
      default_severity: "info",
      requires_graph: true
    }
  ],
  default_config: { enabled: true }
}
```

**Rulepack Implementation Timeline:**

- **Cycles rulepack:** Implemented in Milestone 4 (core cycle detection)
- **Drift rulepack:** Implemented in Milestones 5-9 (drift detection analyzer)
- **Change Impact rulepack:** Implemented in Milestones 5-9 (change impact analyzer)

### 4.4 Snapshot Schema v1

The RepoSnapshot schema is fully defined in [RepoSnapshot Contract](./14-repo-snapshot-contract.md). This section provides a reference summary.

```typescript
interface RepoSnapshot {
  files: Array<{
    path: string;          // Canonical POSIX path
    content: string | null; // UTF-8 normalized text, or null for binary
    hash: string;          // SHA-256 hash
    is_binary: boolean;
  }>;
  
  metadata: {
    repo_fingerprint: string;
    file_count: number;
    total_bytes: number;
    snapshot_version: 1;
  };
}
```

**Key Requirements (from RepoSnapshot Contract):**

- Paths MUST be canonicalized (POSIX, normalized, sorted)
- Content MUST be UTF-8 normalized (NFC) or null for binary
- Files MUST be sorted lexicographically by path
- No filesystem metadata (timestamps, permissions, etc.)
- Deterministic hashing (SHA-256)

**Phase 1 Snapshot Storage:**

- Phase 1 uses **in-memory / temporary snapshot builder** (temporary location in engine)
- Snapshots are constructed on-demand and not persisted
- Snapshot builder will migrate to runtime in Phase 2 (see [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md))
- Future persistence mechanisms in Phase 2+ MUST conform to this schema

### 4.5 PR Summary JSON Schema

The PR Summary is a deterministic JSON structure summarizing analysis results for GitHub Check Run display.

```typescript
interface PRSummary {
  version: "1.0.0";
  
  summary: {
    total_violations: number;
    by_severity: {
      error: number;
      warning: number;
      info: number;
    };
    by_category: Record<string, number>;  // Sorted keys
  };
  
  violations: Array<{
    id: string;
    rulepack_namespace: string;
    rule_id: string;
    severity: "error" | "warning" | "info";
    category: string;
    file_path: string;
    line_number: number | null;
    column_number: number | null;
    message: string;
    context?: Record<string, any>;  // Deep-sorted
  }>;
  
  envelope_reference: {
    signature: string;              // Envelope signature for traceability
    analyzer_version: string;
    schema_version: number;
    repo_fingerprint: string;
  };
}
```

**PR Summary Determinism Rules:**

1. **Sorted Arrays:** `violations` MUST be sorted by `id` lexicographically
2. **Sorted Objects:** `by_category` keys MUST be sorted lexicographically
3. **Deep Sorting:** All nested objects MUST be deep-sorted
4. **No Timestamps:** MUST NOT include any timestamp fields
5. **Stable Serialization:** MUST use `stableStringify` for JSON output

**PR Summary Generation:**

```
PR Summary = {
  version: "1.0.0",
  summary: aggregateViolations(envelope.extensions),
  violations: flattenViolations(envelope.extensions),
  envelope_reference: {
    signature: envelope.meta.signature,
    analyzer_version: envelope.version.analyzer,
    schema_version: envelope.version.schema,
    repo_fingerprint: envelope.meta.repo_fingerprint
  }
}
```

---

## 5. Determinism Impact

All contract layer schemas MUST be deterministic:

- Violation IDs MUST be stable across runs
- Analyzer outputs MUST be identical for identical inputs
- Rulepack execution MUST be deterministic
- PR Summary MUST be byte-for-byte identical for identical envelopes
- Snapshot schema MUST follow RepoSnapshot Contract determinism rules

Violating determinism in any schema breaks:
- Golden tests
- Drift detection
- Shadow analyzer comparisons
- Signature stability
- Envelope reproducibility

---

## 6. Runtime / Engine Boundary Impact

**Engine MUST:**

- Produce violations conforming to Violation Schema v1
- Implement analyzers conforming to Analyzer Interface v1
- Execute rulepacks according to Rulepack Schema v1
- Accept RepoSnapshots conforming to Snapshot Schema v1
- Generate PR Summary conforming to PR Summary JSON Schema

**Runtime MUST:**

- Provide RepoSnapshots conforming to Snapshot Schema v1
- NOT modify violations or analyzer outputs
- NOT inject data into contract layer structures
- Use PR Summary for Check Run display only (no mutations)

---

## 7. Versioning Impact

**Contract Layer Versioning:**

- Violation Schema v1 → v2: Requires `analyzerVersion++` and schema adapter
- Analyzer Interface v1 → v2: Breaking change; requires `analyzerVersion++`
- Rulepack Schema v1 → v2: Rulepack version bumps; may require `analyzerVersion++` if core changes
- Snapshot Schema v1 → v2: See [RepoSnapshot Contract](./14-repo-snapshot-contract.md) versioning rules
- PR Summary v1.0.0 → v1.1.0: Backward-compatible additions only; no `analyzerVersion++` required

**Version Bump Requirements:**

- Structural changes to Violation Schema → `analyzerVersion++`
- Changes to Analyzer Interface → `analyzerVersion++`
- Core Rulepack Schema changes → `analyzerVersion++`
- PR Summary breaking changes → `analyzerVersion++`

---

## 8. Testing Requirements

### 8.1 Schema Validation Tests

All schemas MUST be validated with:

- JSON Schema validation (where applicable)
- TypeScript type checking
- Deterministic serialization tests
- Round-trip serialization tests

### 8.2 Violation Schema Tests

- Violation ID determinism across runs
- Violation sorting stability
- Context deep-sorting correctness
- Message template stability

### 8.3 Analyzer Interface Tests

- Pure function validation (no side effects)
- Deterministic output validation
- Error handling (no throws)
- Graph requirement handling

### 8.4 Rulepack Schema Tests

- Namespace uniqueness
- Analyzer ID uniqueness
- Config schema validation
- Default config correctness

### 8.5 PR Summary Tests

- Deterministic generation
- Sorted array/object correctness
- Deep-sorting validation
- Envelope reference accuracy

---

## 9. Cross-References

- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) — Snapshot Schema v1 full specification
- [Envelope Format Spec](./15-envelope-format-spec.md) — Envelope structure (consumes contract layer)
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md) — Boundary contracts
- [Wedge Architecture Spec](./22-wedge-architecture-spec-v1.md) — Engine implementation
- [Invariants Contract](./06-invariants-contract.md) — Invariant validation

---

## 10. Change Log (Append-Only)

**v1.1.0** — Enhanced rulepack specification and snapshot storage notes

- Added explicit listing of drift and change-impact rulepacks to Phase 1 built-in rulepacks
- Added rulepack implementation timeline (Milestones 4-9)
- Added Phase 1 snapshot storage notes (in-memory/temporary, future persistence reference)

**v1.0.0** — Initial contract layer specification

- Violation Schema v1 defined
- Analyzer Interface v1 defined
- Rulepack Schema v1 defined
- Snapshot Schema v1 reference
- PR Summary JSON Schema v1.0.0 defined

