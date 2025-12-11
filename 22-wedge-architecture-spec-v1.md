# Wedge Architecture Specification v1

## 1. Purpose

This document defines the internal architecture of the `arcsight-wedge` engine, including:

- The snapshot → analyzers → canonicalize → envelope pipeline
- Module layering and dependencies
- Analyzer plugin system
- Internal type signatures
- Error handling rules
- Contract enforcement inside the engine

This specification MUST be finalized before implementing the wedge pipeline. It defines the structure, boundaries, and implementation contracts that all engine code MUST follow.

---

## 2. Scope

This document governs:

### Included

- Engine pipeline architecture (snapshot → envelope)
- Module organization and layering
- Analyzer plugin system design
- Internal type system
- Error handling and degraded mode
- Contract enforcement mechanisms
- Function signatures for all major components

### Not Included

- Data schemas (see [Contract Layer Spec](./21-contract-layer-spec-v1.md))
- Runtime boundaries (see [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- Determinism rules (see [Determinism Contract](./07-determinism-contract.md))
- Folder structure (see [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md))
- Implementation order (see [Phase 1 Implementation Plan](./23-phase1-implementation-plan.md))

---

## 3. Definitions & Terms

**Wedge**  
The deterministic engine package (`arcsight-wedge`).

**Pipeline**  
The sequential processing stages: snapshot → graph → analyzers → canonicalization → envelope.

**Analyzer Plugin**  
A rulepack analyzer function conforming to the Analyzer Interface (see [Contract Layer Spec](./21-contract-layer-spec-v1.md)).

**Module Layer**  
A level in the engine's dependency hierarchy (canonicalization, graph, rulepacks, envelope, etc.).

**Contract Enforcement**  
Runtime validation of invariants and contract compliance within the engine.

---

## 4. Rules / Contract

### 4.1 Engine Pipeline Architecture

The engine follows a strict pipeline architecture:

```
RepoSnapshot
  ↓
[1] Canonicalization
  ↓
Canonical RepoSnapshot
  ↓
[2] Graph Building
  ↓
Graph
  ↓
[3] Rulepack Execution (Analyzers)
  ↓
Violations + Extension Data
  ↓
[4] Canonicalization & Sorting
  ↓
Sorted Violations + Extension Data
  ↓
[5] Envelope Construction
  ↓
Envelope
  ↓
[6] Contract Enforcement & Validation
  ↓
[7] Schema Adaptation (if needed)
  ↓
[8] Signature Computation
  ↓
Final Envelope
```

**Pipeline Stages:**

1. **Canonicalization:** Normalize paths, newlines, Unicode; sort files
2. **Graph Building:** Construct dependency graph from snapshot
3. **Rulepack Execution:** Execute all enabled rulepack analyzers
4. **Canonicalization & Sorting:** Sort violations, deep-sort extension data
5. **Envelope Construction:** Build core envelope structure
6. **Contract Enforcement:** Validate invariants and contracts
7. **Schema Adaptation:** Apply schema adapters if needed (Phase 1: no-op)
8. **Signature Computation:** Compute deterministic signature

**Pipeline Invariants:**

- Each stage MUST be pure (no side effects)
- Each stage MUST be deterministic
- Stages MUST execute sequentially (no parallelism in Phase 1)
- Pipeline MUST fail fast on invariant violations

### 4.2 Module Layering

The engine is organized into strict dependency layers:

```
Layer 0: Utils (no dependencies)
  - deepSort.ts
  - stableStringify.ts
  - uuid.ts (deterministic UUIDv5)

Layer 1: Canonicalization (depends on Layer 0)
  - normalizePaths.ts
  - normalizeNewlines.ts
  - normalizeUnicode.ts
  - orderFiles.ts
  - canonicalHash.ts
  - canonicalRepoSnapshot.ts

Layer 2: Types (depends on Layer 0)
  - RepoSnapshot.ts
  - Envelope.ts
  - Violation.ts
  - GraphTypes.ts
  - IdentityChain.ts
  - ErrorCodes.ts

Layer 3: Graph (depends on Layer 1, 2)
  - buildGraph.ts
  - detectCycles.ts
  - graphStats.ts
  - Graph.ts

Layer 4: Rulepacks (depends on Layer 1, 2, 3)
  - registry.ts
  - executor.ts
  - cycles/index.ts
  - cycles/detectCircularDependencies.ts

Layer 5: Invariants (depends on Layer 2, 3, 4)
  - validateGraph.ts
  - validateEnvelope.ts
  - validateViolations.ts
  - contractEnforcer.ts

Layer 6: Envelope (depends on Layer 1, 2, 3, 4, 5)
  - buildCoreEnvelope.ts
  - attachExtensions.ts
  - signEnvelope.ts

Layer 7: Adapters (depends on Layer 2, 6)
  - schemaAdapters.ts

Layer 8: Public API (depends on all layers)
  - index.ts (analyze function)
```

**Layer Rules:**

- Modules MUST ONLY import from lower-numbered layers
- Circular dependencies are FORBIDDEN
- Layer boundaries MUST be enforced by CI
- Type definitions (Layer 2) may be imported by any higher layer

### 4.3 Analyzer Plugin System

Analyzers are registered and executed through a plugin system.

**Rulepack Registry:**

```typescript
interface RulepackRegistry {
  [namespace: string]: Rulepack;  // See Contract Layer Spec
}

function registerRulepack(rulepack: Rulepack): void;
function getRulepack(namespace: string): Rulepack | null;
function getAllRulepacks(): Rulepack[];
```

**Analyzer Execution:**

```typescript
interface ExecutionContext {
  snapshot: RepoSnapshot;
  graph?: Graph;
  config: Record<string, AnalyzerConfig>;
}

function executeAnalyzers(ctx: ExecutionContext): {
  violations: Violation[];
  extensions: Record<string, any>;
}

function executeAnalyzer(
  analyzer: Analyzer,
  input: AnalyzerInput
): AnalyzerOutput;
```

**Execution Order:**

1. Sort rulepacks by namespace (lexicographically)
2. For each rulepack:
   - Load config (default + overrides)
   - For each analyzer in rulepack:
     - Prepare `AnalyzerInput` (snapshot, config, graph if required)
     - Execute analyzer
     - Collect violations and extension data
3. Aggregate all violations
4. Aggregate all extension data by namespace
5. Sort violations by ID
6. Deep-sort extension data

**Error Handling in Analyzer Execution:**

- Analyzers MUST NOT throw exceptions
- If analyzer throws → catch, log error code, return empty violations
- Engine MUST continue with other analyzers
- Error MUST be recorded in envelope metadata (degraded mode)

### 4.4 Internal Type Signatures

**Core Function Signatures:**

```typescript
// Public API
function analyze(
  snapshot: RepoSnapshot,
  identity: IdentityChain,
  config?: AnalyzerConfig
): Envelope;

// Canonicalization
function canonicalizeSnapshot(snapshot: RepoSnapshot): RepoSnapshot;
function normalizePath(path: string): string;
function normalizeNewlines(content: string): string;
function normalizeUnicode(content: string): string; // NFC form
function canonicalHash(content: string | Uint8Array): string; // SHA-256

// Graph Building
interface Graph {
  nodes: Map<string, GraphNode>;
  edges: Map<string, Set<string>>;
}

function buildGraph(snapshot: RepoSnapshot): Graph;
function detectCycles(graph: Graph): Array<Array<string>>; // Canonically sorted
function graphStats(graph: Graph): { node_count: number; edge_count: number };

// Rulepack Execution
function executeRulepacks(
  snapshot: RepoSnapshot,
  graph: Graph,
  config: Record<string, AnalyzerConfig>
): {
  violations: Violation[];
  extensions: Record<string, any>;
};

// Envelope Construction
function buildCoreEnvelope(
  snapshot: RepoSnapshot,
  identity: IdentityChain,
  graph: Graph,
  cycles: Array<Array<string>>,
  violations: Violation[]
): CoreEnvelope;

function attachExtensions(
  envelope: CoreEnvelope,
  extensions: Record<string, any>
): Envelope;

function signEnvelope(envelope: Envelope): string; // SHA-256 signature

// Contract Enforcement
function validateGraph(graph: Graph): void | throws;
function validateEnvelope(envelope: Envelope): void | throws;
function validateViolations(violations: Violation[]): void | throws;

// Degraded Mode
function degradeForLimits(
  reason: string,
  snapshot: RepoSnapshot,
  identity: IdentityChain
): Envelope;

function degradeForComplexity(
  reason: string,
  snapshot: RepoSnapshot,
  identity: IdentityChain
): Envelope;
```

**Type Constraints:**

- All functions MUST be pure (no side effects)
- All functions MUST be deterministic
- All functions MUST NOT access filesystem, network, or environment
- Error handling MUST use deterministic error codes (no exceptions in normal flow)

### 4.5 Error Handling Rules

**Error Classification:**

1. **Invariant Violations:** Hard failures → deterministic error envelope
2. **Config Errors:** Validation failures → deterministic error envelope
3. **Analyzer Errors:** Analyzer throws → catch, continue, degraded mode
4. **Limit Exceeded:** Resource limits → degraded envelope
5. **Complexity Exceeded:** Analysis complexity → degraded envelope

**Error Envelope Requirements:**

- MUST be structurally valid
- MUST include all required fields
- MUST have deterministic `error_code`
- MUST have valid signature
- MUST NOT include stack traces or machine-specific data

**Error Propagation:**

```
[Function] → Error Code → [Contract Enforcer] → [Envelope Builder] → Error Envelope
```

**Deterministic Error Codes:**

```typescript
enum ErrorCode {
  INVALID_SNAPSHOT = "INVALID_SNAPSHOT",
  GRAPH_BUILD_FAILED = "GRAPH_BUILD_FAILED",
  CYCLE_DETECTION_FAILED = "CYCLE_DETECTION_FAILED",
  ANALYZER_EXECUTION_FAILED = "ANALYZER_EXECUTION_FAILED",
  ENVELOPE_VALIDATION_FAILED = "ENVELOPE_VALIDATION_FAILED",
  CONFIG_VALIDATION_FAILED = "CONFIG_VALIDATION_FAILED",
  LIMIT_EXCEEDED = "LIMIT_EXCEEDED",
  COMPLEXITY_EXCEEDED = "COMPLEXITY_EXCEEDED"
}
```

### 4.6 Contract Enforcement

Contract enforcement validates invariants and contract compliance at runtime.

**Enforcement Points:**

1. **Input Validation:** Validate RepoSnapshot structure and invariants
2. **Graph Validation:** Validate graph structure and invariants
3. **Violation Validation:** Validate violation schema compliance
4. **Envelope Validation:** Validate envelope structure and invariants
5. **Signature Validation:** Validate signature computation

**Contract Enforcer:**

```typescript
class ContractEnforcer {
  validateSnapshot(snapshot: RepoSnapshot): void;
  validateGraph(graph: Graph): void;
  validateViolations(violations: Violation[]): void;
  validateEnvelope(envelope: Envelope): void;
  
  enforceInvariant(
    condition: boolean,
    errorCode: ErrorCode,
    message: string
  ): void | throws;
}
```

**Invariant Enforcement:**

- MUST validate all invariants from [Invariants Contract](./06-invariants-contract.md)
- MUST fail fast on invariant violations
- MUST produce deterministic error envelopes
- MUST NOT log or emit non-deterministic data

### 4.7 Degraded Mode Architecture

Degraded mode handles cases where full analysis cannot complete but a valid envelope must still be produced.

**Degradation Triggers:**

1. **Resource Limits:** File count, total bytes, path length exceeded
2. **Complexity Limits:** Graph size, cycle count, analysis time exceeded
3. **Analyzer Failures:** Critical analyzers fail (non-fatal)

**Degraded Envelope Structure:**

```typescript
interface DegradedEnvelope extends Envelope {
  core: {
    ...CoreEnvelopeFields,
    status: "degraded",
    degraded_reason: string;  // Deterministic reason code
  };
}
```

**Degradation Rules:**

- MUST still produce valid envelope
- MUST include `degraded_reason`
- MUST preserve identity and metadata
- MUST compute valid signature
- MUST NOT include partial or corrupted data

---

## 5. Determinism Impact

The architecture enforces determinism at every layer:

- Pipeline stages MUST be deterministic
- Module layering prevents nondeterministic dependencies
- Analyzer execution MUST be deterministic
- Error handling MUST be deterministic
- Contract enforcement MUST be deterministic

Any architecture change that breaks determinism is forbidden.

---

## 6. Runtime / Engine Boundary Impact

**Engine Internal:**

- Pipeline is completely internal to engine
- Module layers are internal implementation details
- Analyzer plugin system is internal to engine
- Contract enforcement is internal to engine

**Public API:**

- Only `analyze()` function is public
- Internal types (Graph, etc.) are NOT exported
- Only RepoSnapshot and Envelope types are exported

**Runtime Interaction:**

- Runtime calls `analyze(snapshot, identity, config?)`
- Engine returns `Envelope`
- No other interaction points

---

## 7. Versioning Impact

**Architecture Versioning:**

- Architecture changes that break determinism → `analyzerVersion++`
- Module layer changes → `analyzerVersion++`
- Analyzer interface changes → `analyzerVersion++` (see Contract Layer Spec)
- Error code additions → backward compatible, no version bump
- Internal refactoring → no version bump if behavior unchanged

**Versioning Rules:**

- Internal refactoring that preserves behavior → no version bump
- Performance improvements → no version bump
- Bug fixes → may require `analyzerVersion++` if output changes

---

## 8. Testing Requirements

### 8.1 Pipeline Tests

- Each pipeline stage MUST have unit tests
- Pipeline integration tests MUST validate end-to-end flow
- Pipeline MUST produce identical output for identical input

### 8.2 Module Layer Tests

- Layer dependency violations MUST be detected by CI
- Each module MUST have unit tests
- Cross-layer integration MUST be tested

### 8.3 Analyzer Plugin Tests

- Analyzer registration MUST be tested
- Analyzer execution MUST be tested
- Error handling in analyzers MUST be tested
- Execution order MUST be validated

### 8.4 Contract Enforcement Tests

- All invariant validations MUST be tested
- Error envelope generation MUST be tested
- Contract violation detection MUST be tested

### 8.5 Degraded Mode Tests

- Degradation triggers MUST be tested
- Degraded envelope structure MUST be validated
- Degraded mode determinism MUST be tested

---

## 9. Cross-References

- [Contract Layer Spec](./21-contract-layer-spec-v1.md) — Data schemas and interfaces
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) — File structure
- [Phase 1 Implementation Plan](./23-phase1-implementation-plan.md) — Build order
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements
- [Invariants Contract](./06-invariants-contract.md) — Invariant definitions
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md) — Boundary contract
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md) — Snapshot schema
- [Envelope Format Spec](./15-envelope-format-spec.md) — Envelope schema

---

## 10. Change Log (Append-Only)

**v1.0.0** — Initial wedge architecture specification

- Engine pipeline architecture defined
- Module layering established
- Analyzer plugin system specified
- Internal type signatures defined
- Error handling rules established
- Contract enforcement architecture defined
- Degraded mode architecture specified

