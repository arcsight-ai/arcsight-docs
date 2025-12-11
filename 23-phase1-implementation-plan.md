# Phase 1 Implementation Plan

## 1. Purpose

This document defines the milestone-level implementation plan for Phase 1, including:

- Build order and dependencies
- Test harness requirements
- Freeze criteria
- Wedge versioning strategy
- Acceptance conditions

This plan ensures Phase 1 stays deterministic, small, and on track. It provides the execution roadmap that must be followed to achieve zero-drift, golden-stable engine output before Phase 2 migration.

---

## 2. Scope

This document governs:

### Included

- Phase 1 build order and milestones
- Implementation dependencies
- Test harness architecture and requirements
- Freeze criteria for each milestone
- Versioning strategy (analyzerVersion, schemaVersion)
- Acceptance conditions for Phase 1 completion
- Integration points between components

### Not Included

- Detailed implementation specifications (see [Wedge Architecture Spec](./22-wedge-architecture-spec-v1.md))
- Data schemas (see [Contract Layer Spec](./21-contract-layer-spec-v1.md))
- Folder structure details (see [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md))
- Runtime implementation details (see [Full System Roadmap](./03-full-system-roadmap.md))

---

## 3. Definitions & Terms

**Milestone**  
A discrete phase of implementation with defined deliverables and acceptance criteria.

**Freeze Criteria**  
Conditions that must be met before a milestone is considered complete and development may proceed.

**Test Harness**  
The infrastructure for running golden tests, unit tests, and integration tests.

**Golden Baseline**  
The canonical, expected output for a golden test repository. Must be byte-for-byte stable.

**Zero Drift**  
The condition where identical inputs produce identical outputs across all environments and time.

---

## 4. Rules / Contract

### 4.1 Build Order

Phase 1 implementation MUST follow this exact order:

#### Milestone 1: Foundation & Canonicalization

**Deliverables:**
- Utils module (deepSort, stableStringify, UUIDv5)
- Canonicalization module (path, newline, Unicode normalization)
- RepoSnapshot type definitions
- Basic unit tests

**Dependencies:**
- None (foundation layer)

**Freeze Criteria:**
- ✅ All canonicalization functions produce deterministic output
- ✅ Unit tests pass with 100% deterministic output validation
- ✅ Path normalization handles all edge cases (`.`, `..`, repeated slashes)
- ✅ Unicode normalization uses NFC form correctly
- ✅ No filesystem or environment dependencies

**Version:** Not applicable (no analyzer yet)

---

#### Milestone 2: RepoSnapshot Builder (Temporary)

**Deliverables:**
- Snapshot builder module (temporary location in engine)
- RepoSnapshot construction from filesystem/tarball
- Snapshot validation
- Snapshot serialization/deserialization tests

**Dependencies:**
- Milestone 1 (canonicalization)

**Freeze Criteria:**
- ✅ Snapshot builder produces identical snapshots for identical inputs
- ✅ Snapshot validation enforces all RepoSnapshot Contract rules
- ✅ Serialization round-trips correctly (`stableParse(stableStringify(snapshot)) === snapshot`)
- ✅ Binary file handling is deterministic
- ✅ Snapshot metadata (repo_fingerprint, file_count, total_bytes) is correct

**Version:** Not applicable (no analyzer yet)

**Note:** Snapshot builder will migrate to runtime in Phase 2. It lives in engine temporarily for Phase 1.

---

#### Milestone 3: Graph Building

**Deliverables:**
- Graph type definitions
- Graph builder module
- Graph statistics computation
- Graph validation

**Dependencies:**
- Milestone 1 (canonicalization)
- Milestone 2 (snapshot)

**Freeze Criteria:**
- ✅ Graph structure is deterministic for identical snapshots
- ✅ Graph statistics are accurate and deterministic
- ✅ Graph validation enforces structural invariants
- ✅ Edge ordering is stable (sorted adjacency lists)
- ✅ Node ordering is stable (sorted by canonical path)

**Version:** Not applicable (no analyzer yet)

---

#### Milestone 4: Cycle Detection (Core Rulepack)

**Deliverables:**
- Cycle detection algorithm
- Cycle canonicalization (rooting + sorting)
- Cycles rulepack (built-in core rulepack)
- Cycle violation generation

**Dependencies:**
- Milestone 3 (graph building)

**Freeze Criteria:**
- ✅ Cycle detection produces identical results for identical graphs
- ✅ Cycle canonicalization follows determinism rules (lexicographically smallest node as root)
- ✅ Cycles are sorted deterministically (length, then lexicographic order)
- ✅ Cycle violations conform to Violation Schema v1
- ✅ No duplicate cycles in output
- ✅ Cycle detection handles edge cases (self-loops, disconnected cycles)

**Version:** Not applicable (no analyzer yet, but rulepack structure established)

---

#### Milestone 5: Analyzer Plugin System

**Deliverables:**
- Rulepack registry
- Analyzer executor
- Analyzer interface implementation
- Plugin system tests

**Dependencies:**
- Milestone 4 (cycle detection rulepack)
- [Contract Layer Spec](./21-contract-layer-spec-v1.md) must be finalized

**Freeze Criteria:**
- ✅ Rulepack registration works correctly
- ✅ Analyzer execution is deterministic
- ✅ Analyzer error handling doesn't crash engine
- ✅ Execution order is deterministic (sorted by namespace)
- ✅ Violation aggregation is correct
- ✅ Extension data aggregation is correct

**Version:** `analyzerVersion: "0.1.0"` (initial development version)

---

#### Milestone 6: Invariant System & Contract Enforcement

**Deliverables:**
- Invariant validation module
- Contract enforcer
- Error code system
- Invariant tests

**Dependencies:**
- Milestone 5 (analyzer system)
- [Invariants Contract](./06-invariants-contract.md) must be finalized

**Freeze Criteria:**
- ✅ All invariants from Invariants Contract are validated
- ✅ Contract violations produce deterministic error envelopes
- ✅ Error codes are canonical and stable
- ✅ Invariant validation is fast (doesn't slow down pipeline)
- ✅ Validation covers graph, violations, and envelope

**Version:** `analyzerVersion: "0.2.0"`

---

#### Milestone 7: Envelope Construction

**Deliverables:**
- Core envelope builder
- Extension attachment
- Signature computation
- Envelope validation

**Dependencies:**
- Milestone 6 (invariants)
- [Envelope Format Spec](./15-envelope-format-spec.md) must be finalized

**Freeze Criteria:**
- ✅ Envelope structure matches Envelope Format Spec exactly
- ✅ Signature computation is deterministic
- ✅ Extension attachment preserves determinism
- ✅ Envelope validation catches all structural errors
- ✅ Identity fields are preserved correctly

**Version:** `analyzerVersion: "0.3.0"`, `schemaVersion: 1`

---

#### Milestone 8: Degraded Mode

**Deliverables:**
- Degraded mode handlers
- Limit enforcement
- Complexity detection
- Degraded envelope generation

**Dependencies:**
- Milestone 7 (envelope construction)
- [Deterministic Config Contract](./20-deterministic-config-contract.md) must be finalized

**Freeze Criteria:**
- ✅ Degraded envelopes are structurally valid
- ✅ Degradation reasons are deterministic
- ✅ Limits are enforced deterministically
- ✅ Degraded mode doesn't break envelope signatures
- ✅ All degraded envelopes pass validation

**Version:** `analyzerVersion: "0.4.0"`

---

#### Milestone 9: Pipeline Integration

**Deliverables:**
- Complete pipeline assembly
- Public `analyze()` API
- End-to-end integration tests
- Pipeline error handling

**Dependencies:**
- All previous milestones
- [Wedge Architecture Spec](./22-wedge-architecture-spec-v1.md) must be finalized

**Freeze Criteria:**
- ✅ Pipeline produces deterministic envelopes for identical snapshots
- ✅ All pipeline stages execute in correct order
- ✅ Error handling works at all stages
- ✅ Integration tests pass
- ✅ Pipeline performance is acceptable (no timeouts on golden repos)

**Version:** `analyzerVersion: "0.5.0"`

---

#### Milestone 10: Golden Test Infrastructure

**Deliverables:**
- Golden test harness
- Golden test repositories
- Baseline generation tooling
- Comparison utilities

**Dependencies:**
- Milestone 9 (pipeline integration)
- [Golden Test Governance](./09-golden-test-governance.md) must be finalized

**Freeze Criteria:**
- ✅ Golden test harness runs deterministically
- ✅ Baseline generation produces stable outputs
- ✅ Comparison utilities detect differences correctly
- ✅ Golden repos cover key scenarios with specific coverage requirements:
  - **Small repos:** Basic cycle detection, simple dependencies
  - **Medium repos:** Cross-module cycles, multiple rulepack outputs
  - **Complex repos:** Large dependency graphs, nested cycles, boundary violations
  - **Edge cases:** Invalid paths, binary files, Unicode edge cases, simple drift cases
- ✅ All golden tests pass consistently

**Golden Test Coverage Requirements:**

Golden test repositories MUST include:

1. **Cross-Module Cycles:**
   - Cycles spanning multiple modules/packages
   - Nested cycles (cycles within cycles)
   - Indirect cycle dependencies

2. **Boundary Violations:**
   - Path escape attempts (`../` escaping repo root)
   - Invalid path characters
   - Symlink handling (rejection cases)
   - Size limit boundary conditions

3. **Simple Drift Cases:**
   - Basic drift detection scenarios
   - Structural changes that should be detected
   - Cases that validate drift analyzer determinism

**Version:** `analyzerVersion: "0.6.0"`

**Important:** Cycle detection MUST be complete and stable before generating golden baselines. See [Full System Roadmap](./03-full-system-roadmap.md) section on "Cycle-Complete Before Golden Tests."

---

#### Milestone 11: Zero Drift & Stability

**Deliverables:**
- Zero drift across all golden repos
- Cross-environment validation (CI)
- Repeat-run validation
- Stability validation over time

**Dependencies:**
- Milestone 10 (golden test infrastructure)

**Freeze Criteria:**
- ✅ Zero drift across all golden repos (identical output for identical input)
- ✅ Golden tests pass on all CI environments (Linux, macOS, Windows if supported)
- ✅ Repeat-run tests pass (run analyzer twice → identical output)
- ✅ No nondeterministic behavior detected
- ✅ All invariants validated
- ✅ All contracts enforced

**Version:** `analyzerVersion: "1.0.0"` (Phase 1 release candidate)

---

#### Milestone 12: Phase 1 Freeze

**Deliverables:**
- Finalized `analyzerVersion: "1.0.0"`
- Frozen golden baselines
- Complete documentation
- Acceptance sign-off

**Dependencies:**
- Milestone 11 (zero drift)

**Freeze Criteria:**
- ✅ `analyzerVersion` frozen at `1.0.0`
- ✅ All golden baselines locked and validated
- ✅ Zero drift confirmed across all test scenarios
- ✅ All contracts implemented and validated
- ✅ Documentation complete
- ✅ Ready for Phase 2 migration

**Version:** `analyzerVersion: "1.0.0"`, `schemaVersion: 1` (FROZEN)

---

### 4.2 Test Harness Requirements

**Test Harness Architecture:**

```
test-harness/
  unit/
    canonicalization/
    graph/
    rulepacks/
    envelope/
    invariants/
  
  integration/
    pipeline/
    analyzer-execution/
    degraded-mode/
  
  golden/
    repos/
      small/
      medium/
      complex/
      edge-cases/
    expected/
      small/
      medium/
      complex/
      edge-cases/
    fixtures/
      snapshots/
      configs/
  
  utils/
    baseline-generator.ts
    comparator.ts
    deterministic-validator.ts
```

**Test Harness Requirements:**

1. **Deterministic Execution:**
   - Tests MUST run deterministically
   - No timestamps in test outputs
   - No randomness in test execution
   - Stable file ordering

2. **Baseline Generation:**
   - MUST use `stableStringify` for baseline generation
   - MUST validate baselines are deterministic before committing
   - MUST include envelope signature in baselines

3. **Comparison Utilities:**
   - MUST compare deep structure, not just JSON strings
   - MUST detect ordering differences
   - MUST report differences clearly

4. **Golden Test Execution:**
   - MUST run on every commit
   - MUST validate zero drift
   - MUST fail CI if drift detected

5. **Cross-Environment Validation:**
   - MUST run on multiple OS environments
   - MUST validate Node version compatibility
   - MUST catch OS-specific nondeterminism

6. **CLI & GitHub App Equivalence Tests (MANDATORY CI):**
   - MUST validate that CLI and GitHub App produce identical envelope output for identical snapshots
   - MUST run as mandatory CI step on every commit
   - MUST fail CI if CLI and GitHub App outputs diverge
   - MUST prevent wedge output drift between CLI and runtime execution paths
   - MUST validate that snapshot builder produces identical snapshots in both contexts

---

### 4.3 Freeze Criteria Summary

**Per-Milestone Freeze:**

- Each milestone MUST meet its freeze criteria before proceeding
- No milestone may be skipped
- Freeze criteria violations MUST block progression

**Phase 1 Final Freeze:**

- ✅ Zero drift across all golden repos
- ✅ `analyzerVersion: "1.0.0"` frozen
- ✅ All contracts implemented
- ✅ All invariants validated
- ✅ Documentation complete
- ✅ Ready for Phase 2 migration

---

### 4.4 Wedge Versioning Strategy

**Version Numbering:**

- `analyzerVersion`: Semver (e.g., "1.0.0")
- `schemaVersion`: Integer (e.g., 1, 2, 3)

**Version Bump Rules:**

- **Milestone 5-9:** Development versions (`0.1.0` → `0.9.0`)
- **Milestone 11:** Release candidate (`1.0.0-rc.1`)
- **Milestone 12:** Phase 1 freeze (`1.0.0`)

**Version Bump Triggers:**

- Structural changes → `analyzerVersion++`
- Schema changes → `schemaVersion++` and `analyzerVersion++`
- Behavior changes → `analyzerVersion++`
- Bug fixes (if output changes) → `analyzerVersion++`
- Performance improvements (no output change) → no version bump

**Version Freeze:**

- Once `analyzerVersion: "1.0.0"` is reached and validated, it is FROZEN
- No changes allowed that would require version bump
- All changes must be backward-compatible or deferred to Phase 2

---

### 4.5 Acceptance Conditions

**Phase 1 Acceptance Criteria:**

1. **Determinism:**
   - ✅ Zero drift across all golden repos
   - ✅ Identical output for identical input (repeat-run validation)
   - ✅ Cross-environment stability (CI validation)

2. **Functionality:**
   - ✅ Core pipeline complete and working
   - ✅ Cycle detection rulepack working
   - ✅ Envelope generation correct
   - ✅ Degraded mode working

3. **Quality:**
   - ✅ All contracts implemented
   - ✅ All invariants validated
   - ✅ All tests passing
   - ✅ No known bugs

4. **Documentation:**
   - ✅ All contract documents complete
   - ✅ Architecture documented
   - ✅ Implementation plan followed

5. **Readiness:**
   - ✅ Ready for Phase 2 migration
   - ✅ Snapshot builder ready to migrate to runtime
   - ✅ Golden baselines stable
   - ✅ Version frozen at `1.0.0`

**Acceptance Sign-off:**

Phase 1 is complete when all acceptance criteria are met and sign-off is obtained. No Phase 2 work may begin until Phase 1 is accepted.

---

## 5. Determinism Impact

This implementation plan enforces determinism at every milestone:

- Each milestone includes determinism validation
- Test harness validates determinism
- Freeze criteria include determinism checks
- Zero drift is a final acceptance criterion

Violating determinism at any milestone blocks progression.

---

## 6. Runtime / Engine Boundary Impact

**Engine Development (Phase 1):**

- Engine is developed independently
- Snapshot builder lives temporarily in engine
- Runtime development may proceed in parallel (separate package)
- Integration testing validates boundary contracts

**Integration Points:**

- Runtime must provide RepoSnapshots conforming to contract
- Engine must produce Envelopes conforming to contract
- Boundary contracts must be validated

---

## 7. Versioning Impact

**Development Versions:**

- `0.1.0` → `0.9.0`: Development milestones
- Frequent version bumps allowed during development
- Version stability not required until Milestone 11

**Release Versions:**

- `1.0.0-rc.1`: Release candidate (Milestone 11)
- `1.0.0`: Phase 1 freeze (Milestone 12)
- Version frozen at `1.0.0` for Phase 1

**Version Lock:**

- Once `1.0.0` is frozen, no version bumps allowed
- All Phase 1 work must be backward-compatible with `1.0.0`
- Phase 2 may introduce `2.0.0` with proper migration

---

## 8. Testing Requirements

### 8.1 Unit Tests

- Each module MUST have unit tests
- Tests MUST validate determinism
- Tests MUST cover edge cases
- Test coverage target: 90%+

### 8.2 Integration Tests

- Pipeline integration tests MUST validate end-to-end flow
- Analyzer execution tests MUST validate plugin system
- Error handling tests MUST validate degraded mode

### 8.3 Golden Tests

- Golden tests MUST cover diverse scenarios
- Golden baselines MUST be byte-for-byte stable
- Golden tests MUST run on every commit
- Zero drift MUST be validated

### 8.4 Determinism Tests

- Repeat-run tests (run twice → identical output)
- Cross-environment tests (Linux, macOS, etc.)
- Time-stability tests (run over time → no drift)

### 8.5 Contract Tests

- All contracts MUST have validation tests
- Invariant tests MUST cover all invariants
- Schema validation tests MUST cover all schemas

### 8.6 CLI & GitHub App Equivalence Tests (MANDATORY)

**MANDATORY CI Requirements:**

- CLI and GitHub App MUST produce identical envelope output for identical RepoSnapshots
- Equivalence tests MUST run on every commit as a mandatory CI step
- CI MUST fail if CLI and GitHub App outputs diverge
- Tests MUST validate that snapshot builder produces identical snapshots in both execution contexts
- Tests MUST prevent wedge output drift between CLI and runtime execution paths

**Test Structure:**

```
integration/
  cli-runtime-equivalence/
    same-snapshot-different-paths.ts  // CLI vs runtime snapshot builder
    envelope-equivalence.ts           // Envelope output comparison
    deterministic-execution.ts        // Repeat-run validation for both paths
```

This ensures that the temporary snapshot builder location (engine in Phase 1) produces identical results whether called from CLI or runtime, preventing any divergence that would break determinism guarantees.

---

## 9. Cross-References

- [Contract Layer Spec](./21-contract-layer-spec-v1.md) — Data schemas
- [Wedge Architecture Spec](./22-wedge-architecture-spec-v1.md) — Engine architecture
- [Phase 1 Folder Scaffold](./02-phase1-folder-scaffold.md) — File structure
- [Full System Roadmap](./03-full-system-roadmap.md) — Overall system plan
- [Golden Test Governance](./09-golden-test-governance.md) — Golden test rules
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements
- [Invariants Contract](./06-invariants-contract.md) — Invariant definitions

---

## 10. Change Log (Append-Only)

**v1.1.0** — Enhanced test coverage and clarity

- Added explicit golden test coverage requirements (cross-module cycles, boundary violations, simple drift cases)
- Added CLI & GitHub App equivalence tests as mandatory CI step
- Clarified test harness requirements for equivalence validation
- Enhanced Milestone 10 freeze criteria with specific coverage requirements

**v1.0.0** — Initial Phase 1 implementation plan

- Build order and milestones defined
- Test harness requirements specified
- Freeze criteria established
- Versioning strategy defined
- Acceptance conditions specified

