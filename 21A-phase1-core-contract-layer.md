# Phase 1 Core Contract Layer

**Author:** Joe Penman  
**Version:** v1.0.0  
**Date:** 2024-12-19  
**Status:** Phase 1 Implementation Spec

---

## Related Documents

- [Contract Layer Spec v1](./21-contract-layer-spec-v1.md) — Abstract schema definitions
- [Contract Authority Index](./00A-phase1-contract-authority-index.md) — Phase 1 contract classification
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) — Snapshot schema specification
- [Envelope Format Spec](./15-envelope-format-spec.md) — Envelope schema specification
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements

---

## Quick Reference Table

| Schema/Interface | File | Type/Function | Purpose |
|------------------|------|---------------|---------|
| **Violation** | `violation.ts` | `interface Violation` | Rulepack-detected issue with deterministic ID and location |
| **Violation ID** | `violation.ts` | `generateViolationId()` | Generates deterministic UUID v5 for violations |
| **Violation Sort** | `violation.ts` | `sortViolations()` | Sorts violations by ID (lexicographically) |
| **Analyzer** | `analyzer.ts` | `type Analyzer` | Pure function type that all analyzers must implement |
| **Analyzer Input** | `analyzer.ts` | `interface AnalyzerInput` | Input to analyzer: snapshot, config, optional graph |
| **Analyzer Output** | `analyzer.ts` | `interface AnalyzerOutput` | Output from analyzer: violations and extension data |
| **Analyzer Config** | `analyzer.ts` | `interface AnalyzerConfig` | Rulepack-specific configuration (deterministic) |
| **Analyzer Normalize** | `analyzer.ts` | `normalizeAnalyzerOutput()` | Normalizes analyzer output for determinism |
| **Rulepack** | `rulepack.ts` | `interface Rulepack` | Rulepack structure: namespace, version, analyzers, config |
| **Rulepack Analyzer** | `rulepack.ts` | `interface RulepackAnalyzer` | Individual analyzer within a rulepack |
| **Rulepack Registry** | `rulepack.ts` | `interface RulepackRegistry` | Map of namespace → Rulepack |
| **Rulepack Sort** | `rulepack.ts` | `sortRulepacks()` | Sorts rulepacks by namespace |
| **Rulepack Order** | `rulepack.ts` | `getRulepackExecutionOrder()` | Returns deterministic execution order |
| **Snapshot** | `snapshot.ts` | `type Snapshot` | Type alias to RepoSnapshot |
| **Snapshot Validate** | `snapshot.ts` | `validateSnapshotSchema()` | Validates unknown value is valid Snapshot |
| **Envelope Schema** | `envelope.ts` | `type EnvelopeSchema` | Type alias to Envelope |
| **PR Summary** | `envelope.ts` | `interface PRSummary` | Deterministic JSON for GitHub Check Runs |
| **Violations Canonicalize** | `coreCanonical.ts` | `canonicalizeViolations()` | Canonicalizes violations array |
| **Extension Canonicalize** | `coreCanonical.ts` | `canonicalizeExtensionData()` | Deep-sorts extension data |
| **PR Summary Canonicalize** | `coreCanonical.ts` | `canonicalizePRSummary()` | Deep-sorts PR Summary |

**File Locations:** All files are in `src/core/` directory. Main exports are in `index.ts`.

---

## 1. Overview

The Phase 1 **core contract layer** defines the **TypeScript schemas and interfaces** that form the contract between the wedge engine, analyzers, and runtime. Located in `src/core/`, this layer enforces:

- **Determinism:** All outputs are stable and reproducible across environments
- **Type Safety:** Fully typed TypeScript definitions for all contract schemas
- **Separation of Concerns:** Clearly defined boundaries between `core/` (contract layer) and `types/` (runtime/execution types)
- **Contract Compliance:** Aligns with [Contract Layer Spec v1](./21-contract-layer-spec-v1.md) and [Contract Authority Index](./00A-phase1-contract-authority-index.md)

### Phase 1 Scope

This contract layer provides **type definitions and interfaces only**. It does **not** contain:

- ❌ Analyzer implementations (these live in `src/rulepacks/`)
- ❌ Engine execution logic (these live in `src/graph/`, `src/envelopes/`, etc.)
- ❌ Runtime orchestration code

The contract layer is the **authoritative source** for:
- Violation schemas
- Analyzer interface contracts
- Rulepack structure definitions
- Snapshot and envelope type references
- PR Summary schemas
- Contract-layer canonicalization helpers

---

## Data Flow Architecture

The core contract layer types flow through the engine pipeline as follows:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Runtime                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  RepoSnapshot (from tarball/filesystem)                   │ │
│  └───────────────────┬───────────────────────────────────────┘ │
└──────────────────────┼──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Engine                                    │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/types/RepoSnapshot (runtime type)                    │ │
│  │         │                                                 │ │
│  │         ▼                                                 │ │
│  │  src/core/snapshot.ts: Snapshot (contract type)          │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/core/analyzer.ts: AnalyzerInput                      │ │
│  │    - snapshot: Snapshot                                   │ │
│  │    - config: AnalyzerConfig                               │ │
│  │    - graph?: Graph                                        │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/rulepacks/*/ (analyzer implementations)              │ │
│  │    └─> Analyzer(snapshot, config, graph?)                │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/core/analyzer.ts: AnalyzerOutput                     │ │
│  │    - violations: Violation[]                              │ │
│  │    - extension_data?: Record<string, any>                 │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/core/coreCanonical.ts                                │ │
│  │    - canonicalizeViolations()                             │ │
│  │    - canonicalizeExtensionData()                          │ │
│  └───────────────────┬───────────────────────────────────────┘ │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  src/types/Envelope (implementation type)                 │ │
│  │         │                                                 │ │
│  │         ▼                                                 │ │
│  │  src/core/envelope.ts: EnvelopeSchema (contract type)    │ │
│  └───────────────────┬───────────────────────────────────────┘ │
└──────────────────────┼──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Runtime                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  EnvelopeSchema → GitHub Check Run                        │ │
│  │         │                                                 │ │
│  │         ▼                                                 │ │
│  │  src/core/envelope.ts: PRSummary (for display)           │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Key Points:**

- **Contract types** (`src/core/`) define the public API boundaries
- **Runtime types** (`src/types/`) are used internally by engine implementation
- **Analyzers** consume `AnalyzerInput` and produce `AnalyzerOutput` (both contract types)
- **Canonicalization** ensures determinism before envelope construction
- **PR Summary** is derived from Envelope for GitHub Check Run display

This architecture ensures clean separation between contract layer (public API) and implementation details (runtime types).

See also: [Wedge Architecture Spec v1](./22-wedge-architecture-spec-v1.md) for detailed engine pipeline architecture.

---

## 2. Directory Structure

### 2.1 Core Contract Layer (`src/core/`)

```text
src/core/
├── README.md           # Boundary documentation and usage guide
├── violation.ts        # Violation Schema v1
├── analyzer.ts         # Analyzer Interface v1
├── rulepack.ts         # Rulepack Schema v1
├── snapshot.ts         # Snapshot Schema v1 (RepoSnapshot reference)
├── envelope.ts         # Envelope Schema v1 + PRSummary
├── coreCanonical.ts    # Contract-layer canonicalization helpers
└── index.ts            # Main exports (public API)
```

### 2.2 File Purpose Reference

| File | Purpose |
|------|---------|
| `violation.ts` | Defines `Violation` interface with deterministic ID generation (`generateViolationId()`) and `sortViolations()` helper |
| `analyzer.ts` | Defines `Analyzer` function type, `AnalyzerInput` / `AnalyzerOutput` interfaces, and `normalizeAnalyzerOutput()` helper |
| `rulepack.ts` | Defines `Rulepack`, `RulepackAnalyzer` interfaces, and registry structure. Helpers: `sortRulepacks()`, `getRulepackExecutionOrder()` |
| `snapshot.ts` | Type alias to `RepoSnapshot` with `validateSnapshotSchema()` for schema validation |
| `envelope.ts` | Type alias to `Envelope` with `PRSummary` interface for GitHub Check Run display |
| `coreCanonical.ts` | Contract-layer canonicalization: `canonicalizeViolations()`, `canonicalizeExtensionData()`, `canonicalizePRSummary()` |
| `index.ts` | Exports all core types and helpers for easy consumption by engine and runtime |

### 2.3 Boundary: `core/` vs `types/`

**`src/core/` (Contract Layer)**

- **Purpose:** Phase 1 schemas defining the **public contract** between runtime and engine
- **Contains:** Violation, Analyzer, Rulepack, Snapshot, Envelope schemas and contract canonicalization helpers
- **Authority:** [Contract Layer Spec v1](./21-contract-layer-spec-v1.md)
- **Exports:** Public API contract types and helpers
- **Dependencies:** May import from `src/utils/` (deepSort, stableStringify) only
- **MUST NOT:** Import from `src/types/` or engine implementation modules

**`src/types/` (Runtime / Execution Types)**

- **Purpose:** Runtime types used by engine **implementation** (not public contract)
- **Contains:** GraphTypes, IdentityChain, Limits, SandboxPolicy, ErrorCodes, RepoSnapshot (implementation types), Envelope (implementation types)
- **Authority:** Module contracts and implementation specs
- **Exports:** Internal engine types
- **Dependencies:** May import from `src/utils/` and `src/core/` (to reference contract schemas)
- **Usage:** Internal engine implementation only

**Boundary Enforcement:**

- `src/core/` MUST NOT import from `src/types/` (**enforced by CI/linting**)
- `src/types/` MAY import from `src/core/` (to reference contract schemas)
- Both MAY import from `src/utils/` (shared deterministic utilities)
- Contract layer types are the **authoritative source** for public API contracts

**CI/Lint Enforcement:**

CI and linting rules MUST enforce that `src/core/` cannot import from `src/types/`. This boundary enforcement is critical for maintaining separation between contract layer (public API) and runtime types (implementation details). Violations of this boundary MUST fail CI builds.

---

## 3. Schema & Interface Summary

All schemas conform to [Contract Layer Spec v1](./21-contract-layer-spec-v1.md).

### 3.1 Violation Schema v1 (`violation.ts`)

```typescript
interface Violation {
  id: string;                    // Deterministic UUID v5
  rulepack_namespace: string;    // e.g., "cycles", "dependencies"
  rule_id: string;               // Unique within namespace
  severity: "error" | "warning" | "info";
  category: string;              // e.g., "circular-dependency"
  file_path: string;             // Canonical POSIX path
  line_number: number | null;    // 1-indexed, null if file-level
  column_number: number | null;  // 1-indexed, null if line-level or file-level
  message: string;               // Constant template (no interpolation)
  context?: Record<string, any>; // Additional deterministic context (deep-sorted)
  detected_at: string;           // Fixed "v1" for Phase 1
  analyzer_version: string;      // From envelope.version.analyzer
}

/**
 * Generates a deterministic violation ID using UUID v5.
 * 
 * @param rulepack_namespace - Rulepack namespace (e.g., "cycles")
 * @param rule_id - Rule ID within namespace
 * @param canonical_location_hash - SHA-256 hash of canonical location
 * @returns Deterministic UUID v5 string
 */
function generateViolationId(
  rulepack_namespace: string,
  rule_id: string,
  canonical_location_hash: string
): string;

/**
 * Sorts violations deterministically by ID (lexicographically).
 * 
 * @param violations - Array of violations to sort
 * @returns Sorted array of violations
 */
function sortViolations(violations: Violation[]): Violation[];
```

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

### 3.2 Analyzer Interface v1 (`analyzer.ts`)

```typescript
interface AnalyzerInput {
  snapshot: RepoSnapshot;        // Canonical snapshot
  config: AnalyzerConfig;        // Rulepack-specific configuration
  graph?: Graph;                 // Optional: pre-built dependency graph
}

interface AnalyzerOutput {
  violations: Violation[];       // Sorted by id (deterministic)
  extension_data?: Record<string, any>; // Deterministic, deep-sorted extension output
}

interface AnalyzerConfig {
  enabled: boolean;
  severity_overrides?: Record<string, "error" | "warning" | "info">;
  [key: string]: any;            // Rulepack-specific config (must be deterministic)
}

/**
 * Pure function type that all analyzers must implement.
 * 
 * Analyzers MUST be pure (no side effects, no I/O, no randomness, no time).
 */
type Analyzer = (input: AnalyzerInput) => AnalyzerOutput;

/**
 * Normalizes analyzer output to ensure determinism:
 * - Sorts violations by ID
 * - Deep-sorts extension_data
 * 
 * @param output - Raw analyzer output
 * @returns Normalized analyzer output
 */
function normalizeAnalyzerOutput(output: AnalyzerOutput): AnalyzerOutput;
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

### 3.3 Rulepack Schema v1 (`rulepack.ts`)

```typescript
interface RulepackAnalyzer {
  id: string;                    // Unique within rulepack
  name: string;                  // Human-readable name
  description: string;           // What it detects
  analyzer_fn: Analyzer;         // The analyzer function
  default_severity: "error" | "warning" | "info";
  requires_graph?: boolean;      // Whether graph is required in input
}

interface Rulepack {
  namespace: string;             // Lowercase kebab-case, e.g., "cycles"
  version: string;               // Semver, e.g., "1.0.0"
  analyzers: RulepackAnalyzer[]; // Array of analyzer definitions
  config_schema?: JSONSchema;    // Optional: JSON Schema for config validation
  default_config: AnalyzerConfig;
}

interface RulepackRegistry {
  [namespace: string]: Rulepack;
}

/**
 * Sorts rulepacks deterministically by namespace (lexicographically).
 * 
 * @param rulepacks - Array of rulepacks to sort
 * @returns Sorted array of rulepacks
 */
function sortRulepacks(rulepacks: Rulepack[]): Rulepack[];

/**
 * Returns deterministic execution order for rulepacks in registry.
 * 
 * @param registry - Rulepack registry
 * @returns Array of namespace strings in execution order
 */
function getRulepackExecutionOrder(registry: RulepackRegistry): string[];
```

**Rulepack Rules:**

- Namespace MUST be unique across all rulepacks
- Version MUST follow semver
- Analyzer IDs MUST be unique within the rulepack namespace
- Execution order MUST be deterministic (sorted by namespace)
- Phase 1 includes ONLY built-in core rulepacks

### 3.4 Snapshot Schema v1 (`snapshot.ts`)

```typescript
import { RepoSnapshot } from '../types';

/**
 * Type alias to RepoSnapshot from types/.
 * The actual schema is defined in RepoSnapshot Contract.
 */
export type Snapshot = RepoSnapshot;

/**
 * Validates that an unknown value conforms to Snapshot schema.
 * 
 * @param value - Value to validate
 * @returns True if value is a valid Snapshot
 */
function validateSnapshotSchema(value: unknown): value is Snapshot;
```

**Snapshot Reference:**

The Snapshot schema is fully defined in [RepoSnapshot Contract](./14-repo-snapshot-contract.md). This module provides:
- Type alias for contract layer usage
- Schema validation helper
- Reference to canonical RepoSnapshot structure

### 3.5 Envelope Schema v1 (`envelope.ts`)

```typescript
import { Envelope } from '../types';

/**
 * Type alias to Envelope from types/.
 * The actual schema is defined in Envelope Format Spec.
 */
export type EnvelopeSchema = Envelope;

/**
 * PR Summary JSON schema for GitHub Check Run display.
 * Derived deterministically from Envelope structure.
 */
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

**Envelope Reference:**

The Envelope schema is fully defined in [Envelope Format Spec](./15-envelope-format-spec.md). The PR Summary schema is defined in [Contract Layer Spec v1](./21-contract-layer-spec-v1.md#45-pr-summary-json-schema).

### 3.6 Canonicalization (`coreCanonical.ts`)

Contract-layer canonicalization handles sorting and normalization specific to contract schemas:

```typescript
/**
 * Canonicalizes violations by sorting deterministically.
 * 
 * @param violations - Array of violations
 * @returns Canonicalized (sorted) violations
 */
function canonicalizeViolations(violations: Violation[]): Violation[];

/**
 * Canonicalizes extension data by deep-sorting all nested structures.
 * 
 * @param data - Extension data object
 * @returns Canonicalized (deep-sorted) extension data
 */
function canonicalizeExtensionData(
  data: Record<string, any>
): Record<string, any>;

/**
 * Canonicalizes PR Summary by deep-sorting all nested structures.
 * 
 * @param summary - PR Summary object
 * @returns Canonicalized (deep-sorted) PR Summary
 */
function canonicalizePRSummary(summary: PRSummary): PRSummary;
```

**Canonicalization Rules:**

1. **Violations:** Sorted by `id` lexicographically
2. **Extension Data:** Deep-sorted using `deepSort()` from `src/utils/`
3. **PR Summary:** All nested objects and arrays deep-sorted

**Dependencies:**

- Uses `deepSort()` from `src/utils/deepSort.ts`
- Uses `stableStringify()` from `src/utils/stableStringify.ts`
- MUST NOT duplicate canonicalization logic

---

## 4. Determinism Requirements

All contract layer types and helpers MUST adhere to the determinism requirements defined in [Determinism Contract](./07-determinism-contract.md):

### 4.1 Stable Identifiers

- **Violation IDs:** Generated deterministically using UUID v5 based on rulepack namespace, rule ID, and canonical location hash
- **No Random IDs:** No random number generation or UUID v4
- **Reproducible:** Same inputs MUST produce same IDs

### 4.2 Sorted Arrays

All arrays MUST be sorted deterministically:

- Violations sorted by `id` (lexicographically)
- Rulepacks sorted by `namespace` (lexicographically)
- PR Summary violations sorted by `id`
- Extension data keys sorted lexicographically

### 4.3 Deep-Sorted Objects

All nested objects MUST be deep-sorted before serialization:

- Extension data objects deep-sorted
- PR Summary nested structures deep-sorted
- Context objects in violations deep-sorted

**Implementation:**

Use `deepSort()` from `src/utils/deepSort.ts` for all deep-sorting operations.

### 4.4 Stable JSON Output

All JSON serialization MUST use `stableStringify()` from `src/utils/stableStringify.ts`:

- Ensures deterministic key ordering
- Ensures consistent serialization across environments
- Required for signature computation

### 4.5 No Nondeterministic Values

Contract layer schemas MUST NOT contain:

- ❌ Timestamps (except `detected_at: "v1"` constant for Phase 1)
- ❌ Random values
- ❌ Machine-specific identifiers
- ❌ Environment-dependent values
- ❌ OS-specific paths or metadata

---

## 5. Usage Example

```typescript
import {
  // Types
  Violation,
  Analyzer,
  AnalyzerInput,
  AnalyzerOutput,
  AnalyzerConfig,
  Rulepack,
  RulepackAnalyzer,
  RulepackRegistry,
  Snapshot,
  EnvelopeSchema,
  PRSummary,
  
  // Helpers
  generateViolationId,
  sortViolations,
  normalizeAnalyzerOutput,
  sortRulepacks,
  getRulepackExecutionOrder,
  validateSnapshotSchema,
  canonicalizeViolations,
  canonicalizeExtensionData,
  canonicalizePRSummary,
} from '../core';

// Example: Create a deterministic violation
const locationHash = SHA256(stableStringify({
  file_path: 'src/foo/bar.ts',
  line_number: 42,
  column_number: 10,
}));

const violation: Violation = {
  id: generateViolationId('cycles', 'circular-dependency-detector', locationHash),
  rulepack_namespace: 'cycles',
  rule_id: 'circular-dependency-detector',
  severity: 'error',
  category: 'circular-dependency',
  file_path: 'src/foo/bar.ts',
  line_number: 42,
  column_number: 10,
  message: 'Circular dependency detected',
  detected_at: 'v1',
  analyzer_version: '1.0.0',
};

// Example: Sort multiple violations deterministically
const sortedViolations = canonicalizeViolations([violation]);

// Example: Define an analyzer
const analyzer: Analyzer = (input: AnalyzerInput): AnalyzerOutput => {
  // Analyzer implementation...
  const violations: Violation[] = []; // ... generate violations
  const extensionData = { /* ... */ };
  
  return normalizeAnalyzerOutput({
    violations: violations,
    extension_data: canonicalizeExtensionData(extensionData),
  });
};

// Example: Define a rulepack
const cyclesRulepack: Rulepack = {
  namespace: 'cycles',
  version: '1.0.0',
  analyzers: [
    {
      id: 'circular-dependency-detector',
      name: 'Circular Dependency Detector',
      description: 'Detects cycles in dependency graphs',
      analyzer_fn: analyzer,
      default_severity: 'error',
      requires_graph: true,
    },
  ],
  default_config: { enabled: true },
};

// Example: Use rulepack registry
const registry: RulepackRegistry = {
  cycles: cyclesRulepack,
};

const executionOrder = getRulepackExecutionOrder(registry);
// executionOrder = ['cycles']
```

---

## 6. Phase 1 Restrictions

### 6.1 No Analyzer Implementations

- `src/core/` contains **ONLY** type definitions and interfaces
- Analyzer implementations live in `src/rulepacks/`
- Contract layer defines the **contract**, not the implementation

### 6.2 Only Built-in Core Rulepacks

- Phase 1 includes ONLY built-in core rulepacks (e.g., cycles detection)
- External rulepacks are **forbidden** until Phase 2
- Rulepack registry is limited to Phase 1 core rulepacks

### 6.3 Schemas Frozen at v1

- All schemas are v1 and **frozen** for Phase 1
- No schema evolution allowed until Phase 2
- Schema changes require:
  - `schemaVersion++`
  - `analyzerVersion++`
  - Schema adapter updates
  - Golden test updates

**Phase 1 Freeze:**

- **Contract layer is frozen at v1.0.0 for Phase 1**
- All changes must be backward-compatible or deferred to Phase 2
- Schema changes require `schemaVersion++` and `analyzerVersion++`
- No breaking changes to contract layer schemas allowed until Phase 2

---

## 7. Testing Requirements

### 7.1 Type Definition Tests

- All interfaces MUST have TypeScript type tests
- All type guards MUST be tested
- All exported functions MUST have unit tests

### 7.2 Canonicalization Tests

- `canonicalizeViolations()` MUST produce deterministic output
- `canonicalizeExtensionData()` MUST produce deterministic output
- `canonicalizePRSummary()` MUST produce deterministic output
- All canonicalization functions MUST be tested for stability

### 7.3 Determinism Tests

- Violation ID generation MUST be deterministic
- Sorting functions MUST produce stable output
- Deep-sorting MUST be tested across nested structures
- Repeat-run tests (run twice → identical output)

### 7.4 Boundary Tests

- `src/core/` MUST NOT import from `src/types/` (enforced by CI/linting)
- Type exports MUST match Contract Layer Spec v1
- Helper functions MUST match contract requirements

---

## 8. Cross-References

- [Contract Layer Spec v1](./21-contract-layer-spec-v1.md) — Abstract schema definitions
- [Contract Authority Index](./00A-phase1-contract-authority-index.md) — Phase 1 contract classification
- [Wedge Architecture Spec v1](./22-wedge-architecture-spec-v1.md) — Engine implementation using contract layer
- [Phase 1 Implementation Plan](./23-phase1-implementation-plan.md) — Implementation roadmap
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) — Snapshot schema specification
- [Envelope Format Spec](./15-envelope-format-spec.md) — Envelope schema specification
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements
- [Invariants Contract](./06-invariants-contract.md) — Invariant validation rules

---

## 9. Glossary & Terminology

Common terms used in this document:

- **Violation** — A rulepack-detected issue with deterministic classification, location, and metadata. See [Violation Schema v1](./21-contract-layer-spec-v1.md#41-violation-schema-v1).
- **Envelope** — The deterministic output structure produced by the engine. See [Envelope Format Spec](./15-envelope-format-spec.md).
- **Snapshot** — The canonical, normalized representation of a repository used as input to the engine. See [RepoSnapshot Contract](./14-repo-snapshot-contract.md).
- **Analyzer** — A pure function that processes a Snapshot and produces violations and optional extension data. See [Analyzer Interface v1](./21-contract-layer-spec-v1.md#42-analyzer-interface-v1).
- **Rulepack** — A collection of analyzers with a namespace, version, and configuration schema. See [Rulepack Schema v1](./21-contract-layer-spec-v1.md#43-rulepack-schema-v1).

For additional terminology, see the [Documentation Index & Authoring Protocol](./00-docs-index-and-authoring-protocol.md#11-glossary).

---

## 10. Change Log (Append-Only)

**v1.0.0** — Initial Phase 1 core contract layer specification

- Defined directory structure for `src/core/`
- Specified TypeScript type definitions for all contract schemas
- Established boundary between `core/` and `types/`
- Defined contract layer canonicalization helpers
- Documented Phase 1 restrictions and determinism requirements
- Added comprehensive usage examples
- Added Quick Reference Table for rapid schema lookup
- Added Data Flow Architecture diagram showing type flow through engine
- Added Glossary & Terminology section with cross-references
- Enhanced CI/Lint enforcement notes for boundary rules
- Clarified versioning freeze requirements (v1.0.0 frozen for Phase 1)

---

## 11. Appendix: File Structure Reference

### Complete `src/core/` Structure

```
src/core/
├── README.md
│   └── Boundary documentation, usage guide, and contract layer overview
│
├── violation.ts
│   ├── Violation interface
│   ├── generateViolationId()
│   └── sortViolations()
│
├── analyzer.ts
│   ├── Analyzer type
│   ├── AnalyzerInput interface
│   ├── AnalyzerOutput interface
│   ├── AnalyzerConfig interface
│   └── normalizeAnalyzerOutput()
│
├── rulepack.ts
│   ├── Rulepack interface
│   ├── RulepackAnalyzer interface
│   ├── RulepackRegistry interface
│   ├── sortRulepacks()
│   └── getRulepackExecutionOrder()
│
├── snapshot.ts
│   ├── Snapshot type alias
│   └── validateSnapshotSchema()
│
├── envelope.ts
│   ├── EnvelopeSchema type alias
│   └── PRSummary interface
│
├── coreCanonical.ts
│   ├── canonicalizeViolations()
│   ├── canonicalizeExtensionData()
│   └── canonicalizePRSummary()
│
└── index.ts
    └── Exports all core types and helpers
```

### Export Structure (`index.ts`)

```typescript
// Types
export type { Violation } from './violation';
export type { Analyzer, AnalyzerInput, AnalyzerOutput, AnalyzerConfig } from './analyzer';
export type { Rulepack, RulepackAnalyzer, RulepackRegistry } from './rulepack';
export type { Snapshot } from './snapshot';
export type { EnvelopeSchema, PRSummary } from './envelope';

// Helpers
export { generateViolationId, sortViolations } from './violation';
export { normalizeAnalyzerOutput } from './analyzer';
export { sortRulepacks, getRulepackExecutionOrder } from './rulepack';
export { validateSnapshotSchema } from './snapshot';
export { canonicalizeViolations, canonicalizeExtensionData, canonicalizePRSummary } from './coreCanonical';
```
