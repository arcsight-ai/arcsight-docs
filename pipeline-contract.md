# ArcSight Wedge — V1 FINAL LOCKED Pipeline Contract

**Document:** `pipeline-contract.md`

**Scope:** End-to-end orchestration for ArcSight Wedge V1

**Layer:** Orchestration / index-level (logical contract for `src/index.ts`, `analyzeCyclesForCommit`, `analyzeCyclesForPR`, and their callers)

This contract defines the only allowed behavior of the ArcSight pipeline from repo input through to PR surface and snapshot.

It does not redefine module internals. Instead, it specifies:

- Exact entrypoints
- Exact module order
- Exact data flow
- Pipeline-level immutability and invariants
- Pipeline-level silent-mode rules
- Pipeline-level safety precedence
- Pipeline-level snapshot semantics
- What the pipeline layer is forbidden to do

All rules here are binding for:

- `src/index.ts`
- `src/pr/prHandler.ts` (where it calls into the pipeline)
- `src/tools/replayPR.ts`
- `src/tools/determinismTest.ts`

---

## 1. Responsibilities & Non-Responsibilities

### 1.1 What the Pipeline MUST Do

For a given repo + commit(s) / PR, the pipeline MUST:

- Build a deterministic import graph (`imports.ts`)
- Detect import cycles (`cycles.ts`)
- Diff cycles between base and head (`cycleDiff.ts`)
- Compute root-cause edges for new cycles (`rootCause.ts`)
- Compute a confidence score (`computeConfidence.ts`)
- Validate invariants (`invariants.ts`)
- Apply safety rules (`safetySwitch.ts`, `cooldown.ts`, `errorPath.ts`)
- Produce:
  - For commit analysis: a structured `CommitAnalysisResult`
  - For PR analysis: a structured `PRAnalysisResult` and (optionally) a PR comment payload
- Write a snapshot via `snapshotWriter.ts`

Do all of the above deterministically, with:

- No logging
- No global mutable state
- No nondeterministic behavior

### 1.2 What the Pipeline MUST NOT Do

The pipeline MUST NOT:

- Re-implement any module's internal logic
- Re-parse imports, re-detect cycles, or re-diff cycles
- Infer architecture, "governance," "fragility," "risk," or other semantics
- Log anything (no console, no side-channel logging)
- Introduce new signals beyond:
  - canonical cycles
  - root-cause edges
  - confidence
  - invariants result
  - safety decision
  - PR comment text
- Mutate module outputs (see Immutability Rule)
- Depend on environment-specific state (timezone, locale, random seeds)
- Call external services except:
  - Git provider APIs in `prHandler.ts`
  - filesystem where explicitly allowed in `snapshotWriter.ts`

The pipeline is a pure orchestration layer.

---

## 2. Pipeline Entry Points

There are exactly two top-level programmatic entrypoints.

### 2.1 Commit Analysis Entry

```ts
// src/index.ts

export interface CommitAnalysisArgs {
  repoPath: string;    // repo root
  headCommit: string;  // commit SHA or ref (e.g. "HEAD")
  baseCommit?: string; // optional, for base/head diff mode
}

export interface CommitAnalysisResult {
  importAnalysis: ImportAnalysisResult | null;
  cycleDetection: CycleDetectionResult | null;
  cycleDiff: CycleDiffResult | null;
  rootCause: RootCauseResult | null;
  confidence: number | null;
  invariants: InvariantsResult | null;
  safety: SafetyDecision | null;
  snapshotWritten: boolean;
  errorDetected: boolean; // true if pipeline could not guarantee correctness
}

export function analyzeCyclesForCommit(
  args: CommitAnalysisArgs
): Promise<CommitAnalysisResult>;
```

### 2.2 PR Analysis Entry

```ts
// src/index.ts

export interface PRAnalysisArgs {
  repoPath: string;
  baseSha: string;
  headSha: string;
  prNumber: number;
}

export interface PRAnalysisResult {
  importAnalysis: ImportAnalysisResult | null;
  cycleDetection: CycleDetectionResult | null;
  cycleDiff: CycleDiffResult | null;
  rootCause: RootCauseResult | null;
  confidence: number | null;
  invariants: InvariantsResult | null;
  safety: SafetyDecision | null;
  relevantCycles: CanonicalCycle[];
  rootCauseEdges: RootCauseEdge[];
  snapshotWritten: boolean;
  errorDetected: boolean;
}

export function analyzeCyclesForPR(
  args: PRAnalysisArgs
): Promise<PRAnalysisResult>;
```

`SafetyDecision` and other referenced types are defined in their respective module contracts. The pipeline MUST NOT alter or extend their meaning.

---

## 3. Canonical Module Order

The only allowed module execution order is:

1. `analysis/imports.ts`
2. `analysis/cycles.ts`
3. `diff/cycleDiff.ts`
4. `diff/rootCause.ts`
5. `confidence/computeConfidence.ts`
6. `safety/invariants.ts`
7. `safety/safetySwitch.ts`
8. `safety/cooldown.ts`
9. `safety/errorPath.ts`
10. `snapshot/snapshotWriter.ts`
11. `pr/prSurface.ts` (PR-only, formatting)
12. `pr/prHandler.ts` (GitHub PR API integration)

**Mandatory rule:**

- The pipeline MUST NOT skip any stage except where silent mode explicitly requires bailout (see Section 5).
- The pipeline MUST NOT reorder stages or insert new analysis stages between them.

---

## 4. Pipeline Data Flow & Immutability

### 4.0 Pipeline Context Immutability Rule (MANDATORY)

Once any stage produces its output, the pipeline MUST treat that output as immutable:

- MUST NOT mutate arrays in-place (push, pop, splice, sort on the original, etc.)
- MUST NOT reassign or mutate object properties on module outputs
- MUST NOT modify strings; all concatenation must produce new strings

If the pipeline needs to transform or filter any structure, it MUST:

- Work on a copy (e.g., `const copy = original.slice()`), not the original

In determinism tests, any detected mutation of module outputs MUST be treated as a pipeline violation and cause a determinism test failure. In production runs, such a violation MUST be caught by invariants/safety and result in silent mode (see Sections 5–6).

### 4.1 Cross-Stage Invariant Enforcement (MANDATORY)

After each stage, the pipeline MUST validate that the stage output satisfies that module's public invariants, including but not limited to:

- Correct types and shapes (arrays, objects, numbers in valid ranges)
- Deterministic ordering where required:
  - e.g. `canonicalCycles` sorted and deduped
  - e.g. `newCycles` sorted and deduped
- Canonical formatting where required:
  - normalized paths (POSIX, lowercase)
  - canonical cycle strings with `" → "` separator
- Set semantics where required:
  - arrays representing sets MUST contain no duplicates

If any invariant check fails for a stage:

- Pipeline MUST set `pipeline.errorDetected = true`
- Pipeline MUST enter silent mode for PR surface (Section 5)
- Safety layer MUST treat this as non-surfaceable

These checks are lightweight and structural; they do not re-run the full internal logic of modules.

### 4.2 Stage 1 — Imports

**Input:**

- `repoPath` from pipeline arguments
- Commit/checkout context provided by caller (working tree corresponds to target commit)

**Call:**

- `imports.ts` → `analyzeImports(repoPath)`

**Output:**

- `ImportAnalysisResult`

**Pipeline rules:**

- If `analyzeImports` fails internally, it MUST return a valid `ImportAnalysisResult` per its contract; the pipeline MUST NOT see exceptions from it.
- Pipeline MUST:
  - Verify that `importGraph` is an array with normalized, sorted, deduped entries
  - Treat `importGraph` as immutable thereafter

### 4.3 Stage 2 — Cycles

**Input:**

- `importGraph: ImportGraphList` from Stage 1

**Call:**

- `cycles.ts` → `detectCycles(importGraph)`

**Output:**

- `CycleDetectionResult`:
  - `canonicalCycles`
  - `rawCycles`
  - `errorDetected`

**Pipeline rules:**

After calling `detectCycles`, pipeline MUST:

- Verify `canonicalCycles` are normalized, sorted, deduped
- Verify `errorDetected` is boolean
- If `errorDetected === true`:
  - Pipeline MUST set `pipeline.errorDetected = true`
  - Pipeline MUST enter silent mode for PR surface

### 4.4 Stage 3 — Cycle Diff

**Commit analysis:**

- If `baseCommit` is not provided, pipeline MAY skip diff:
  - `cycleDiff` MUST be `null` in `CommitAnalysisResult`.
- Otherwise, pipeline MUST analyze both base and head and call `cycleDiff.ts`.

**PR analysis:**

**Input:**

- `baseCycles.canonicalCycles`
- `headCycles.canonicalCycles`

**Call:**

- `cycleDiff.ts` → `diffCycles(base, head)`

**Output:**

- `CycleDiffResult`:
  - `newCycles`
  - `removedCycles`
  - `errorDetected`

**Pipeline rules:**

After calling `diffCycles`, pipeline MUST:

- Verify `newCycles` and `removedCycles` are sorted and deduped
- If `errorDetected === true`:
  - Pipeline MUST set `pipeline.errorDetected = true`
  - Pipeline MUST enter silent mode

Only `newCycles` are used for V1 root-cause detection.

### 4.5 Stage 4 — Root-Cause Detection

**Input:**

- `newCycles: CanonicalCycle[]` from Stage 3
- `headImportGraph: ImportGraphList` from Stage 1 (head)
- `baseImportGraph: ImportGraphList` from base (for comparison as needed)
- `changedFiles`, `diffHunks` from PR/commit context

**Call:**

- `rootCause.ts` → `detectRootCauses(newCycles, changedFiles, diffHunks, baseImportGraph, headImportGraph)`

**Output:**

- `RootCauseResult`:
  - `rootCauseEdges: RootCauseEdge[]`
  - `errorDetected: boolean`

**Pipeline rules:**

After calling `rootCause.ts`, pipeline MUST:

- Verify `rootCauseEdges` are stable and deterministically ordered (sorted by `canonicalCycle`/`from`/`to` per contract)
- Verify all `canonicalCycle` references correspond to cycles in `newCycles`
- If `errorDetected === true`:
  - Pipeline MUST set `pipeline.errorDetected = true`
  - Pipeline MUST enter silent mode

**Relevant cycles for PR path:**

```ts
relevantCycles = uniqueSortedCanonicalCycles(
  rootCauseEdges.map(edge => edge.canonicalCycle)
);
```

- MUST be sorted and deduped
- MUST be treated as immutable thereafter

Cycles without root-cause edges are non-attributable and excluded in V1.

### 4.6 Stage 5 — Confidence

**Input:**

- All analysis context required by `computeConfidence.ts`:
  - `importAnalysis`
  - `cycleDetection`
  - `cycleDiff`
  - `rootCause`
  - any other fields defined in its contract

**Call:**

- `computeConfidence.ts` → `computeConfidence(context)`

**Output:**

- `confidence: number` in [0, 1]

**Pipeline rules:**

Pipeline MUST validate:

- `confidence` is a finite number
- `0 ≤ confidence ≤ 1`

If confidence is outside this range:

- Pipeline MUST set `pipeline.errorDetected = true`
- Pipeline MUST treat the run as unsafe to surface

**PR path gating rule:**

- If `confidence < 0.8`, pipeline MUST NOT surface a PR comment, regardless of cycles.

### 4.7 Stage 6 — Invariants

**Input:**

- `canonicalCycles`
- `importGraph`
- `rootCauseEdges`
- Any additional fields per `invariants.ts` contract

**Call:**

- `invariants.ts` → `validateInvariants(context)`

**Output:**

- `InvariantsResult`:
  - `allInvariantsSatisfied: boolean`
  - `violations: InvariantViolation[]`

**Pipeline rules:**

After calling `validateInvariants`, pipeline MUST:

- Verify `violations` is deterministically ordered
- If `allInvariantsSatisfied === false`:
  - Pipeline MUST set `pipeline.errorDetected = true`
  - Pipeline MUST treat the run as unsafe to surface
  - PR surface MUST NOT be invoked

### 4.8 Stage 7 — Safety (SafetySwitch, Cooldown, ErrorPath)

**Input:**

- `confidence`
- `InvariantsResult`
- Prior module-level `errorDetected` flags
- Historical error / cooldown data (per safety module contracts)

**Calls (in order):**

- `safetySwitch.ts` → high-level allow/deny decision
- `cooldown.ts` → rate-limiting / throttling
- `errorPath.ts` → error-path tracking and extended silence rules

**Output:**

- `SafetyDecision` (e.g., `{ allowSurface: boolean; reason?: string }`)

**Pipeline rules:**

- If `pipeline.errorDetected === true` from any prior stage:
  - Safety layer MUST enforce `allowSurface = false`.
- Safety modules MAY only restrict surface behavior; they MUST NOT override earlier failures.

### 4.9 Stage 8 — Snapshot (with Atomicity)

**Input:**

- Final pipeline state, including:
  - `ImportAnalysisResult`
  - `CycleDetectionResult`
  - `CycleDiffResult`
  - `RootCauseResult`
  - `confidence`
  - `InvariantsResult`
  - `SafetyDecision`
  - For PR: `relevantCycles`, `rootCauseEdges`
  - `pipeline.errorDetected`

**Call:**

- `snapshotWriter.ts` → `writeSnapshot(pipelineState)`

**Output:**

- Snapshot result per its contract (e.g., `{ snapshotWritten: boolean, errorDetected: boolean }`)

**Snapshot Atomicity Rule (MANDATORY):**

`SnapshotWriter` MUST:

- Execute after all safety decisions are made
- Reflect the final pipeline state (safe or silent)
- Perform a single, logically atomic write per pipeline run
- NOT be retried on failure
- NOT influence PR-surface behavior for that run

If snapshot writing fails:

- Pipeline MAY set `pipeline.errorDetected = true` in results
- But the decision to surface or not for this run MUST NOT be retroactively changed due to snapshot failure.

Snapshot failure is an observability issue, not a safety issue.

### 4.10 Stage 9 — PR Surface (PR Path Only)

**Input:**

- `relevantCycles`
- `rootCauseEdges`
- `SafetyDecision`
- Constant fingerprint string

**Precondition:**

PR surface MUST NOT be invoked if:

- `pipeline.errorDetected === true`, OR
- `SafetyDecision.allowSurface === false`, OR
- `confidence < 0.8`, OR
- `InvariantsResult.allInvariantsSatisfied === false`

**Call:**

- `prSurface.ts` → `buildPRComment(cyclesWithRootCause, fingerprint)`

**Output:**

- `commentBody: string` (possibly empty)

If input structure is invalid or arrays are empty, `buildPRComment` MUST return `""`, and the pipeline MUST treat that as "no comment".

### 4.11 Stage 10 — PR Handling (Integration Layer)

`prHandler.ts` orchestrates PR API calls around the pipeline:

- Receives `PullRequestEvent`
- Calls `analyzeCyclesForPR(...)`
- Fetches existing PR comments via API client
- Uses `prIgnore.ts` to detect ignore commands
- Applies:
  - Single-comment invariant
  - Update-in-place invariant
- Creates/updates/deletes the ArcSight PR comment based on:
  - `commentBody` from `prSurface.ts`
  - Ignore result
  - Safety & confidence decisions from the pipeline

**Pipeline contract constraints:**

- `prHandler.ts` MUST treat `analyzeCyclesForPR` as the only wedge engine.
- It MUST NOT run internal analysis modules directly.
- It MUST NOT apply additional heuristics on cycles/root-cause/confidence beyond what is defined in module + pipeline contracts.

---

## 5. Silent Mode & Bailout Rules

The pipeline MUST enter silent mode (no PR comment) if ANY of the following hold:

- `CycleDetectionResult.errorDetected === true`
- `CycleDiffResult.errorDetected === true`
- `RootCauseResult.errorDetected === true`
- `InvariantsResult.allInvariantsSatisfied === false`
- pipeline-level invariants (Section 4.1) fail after any stage
- `confidence < 0.8` (PR path)
- `SafetyDecision.allowSurface === false`
- An unexpected exception is thrown in orchestration code

In silent mode:

- `CommitAnalysisResult.errorDetected` MUST be `true` iff the failure is correctness-related (not just "no cycles")
- `PRAnalysisResult.errorDetected` MUST be `true` iff the failure is correctness-related
- PR surface MUST NOT be invoked
- `SnapshotWriter` MUST still be called with the best-available pipeline state

---

## 6. Safety Precedence Rule (MANDATORY)

If multiple safety-related conditions apply, they MUST be enforced in the following strict priority order:

1. Module-level `errorDetected` flags
   (imports, cycles, cycleDiff, rootCause, snapshot if it impacts correctness)
2. Invariant failures (`InvariantsResult.allInvariantsSatisfied === false`)
3. Confidence threshold (`confidence < 0.8` in PR path)
4. SafetySwitch decision (higher-level safety policy)
5. Cooldown throttling (rate limits)
6. PR ignore command (developer's explicit request to silence)

**Rules:**

- A downstream safety layer MAY silence output further but MAY NOT override an upstream failure.
- For example:
  - Ignore command MUST NOT resurrect an unsafe or low-confidence signal.
  - Cooldown MUST NOT surface something invariants marked as unsafe.

**Result:** The PR comment surface is exposed only if all upstream safety gates are satisfied.

---

## 7. Determinism Requirements (End-to-End)

For fixed:

- `repoPath`
- Commits/SHAs
- PR number
- Filesystem contents

The pipeline MUST produce:

- Identical `CommitAnalysisResult`
- Identical `PRAnalysisResult`
- Identical snapshots (for identical inputs)
- Identical PR comment body (if any)

The pipeline MUST NOT:

- Use randomness
- Use time or clock-based decisions
- Depend on unspecified iteration order (e.g. Map/Set)
- Use environment-specific ordering (locale-sensitive sort) for canonical sorting
- Maintain mutable shared state across calls

Determinism tests (`determinismTest.ts`) MUST treat any deviation as a failure. Immutability violations detected in testing MUST fail the test suite.

---

## 8. Snapshot Semantics (Summary)

Snapshots are append-only records of:

- Commit/PR identity
- Import / cycle / diff / root-cause summary
- Confidence
- Invariants result
- Safety decision
- "Did we surface a comment?" flag (not necessarily the full body)

Snapshots MUST be:

- Deterministic for identical inputs
- Written once per pipeline run
- Independent of PR behavior decisions (failure to write MUST NOT change whether a comment was posted for that run)

---

## 9. Tooling: Replay & Determinism

`replayPR.ts` and `determinismTest.ts` MUST:

- Call `analyzeCyclesForCommit` / `analyzeCyclesForPR` as black-box engines
- NEVER call internal analysis modules directly
- NEVER monkey-patch module implementations
- Use pipeline outputs only for:
  - Determinism checks
  - Replay analyses
  - Metrics computation

Outside-wedge tooling MAY log, visualize, and persist extra debug info, but MUST NOT influence wedge behavior.

---

## 10. Forbidden Pipeline-Layer Behaviors (Summary)

The pipeline (including `index.ts` and its orchestrated flows):

**MUST NOT:**

- Reinterpret or weaken module contracts
- Add new signals
- Introduce new analysis heuristics
- Log from inside wedge modules
- Derive architecture-level conclusions
- Bypass safety or invariants
- Rely on mutable shared state
- Use any nondeterministic API

**MUST:**

- Compose existing modules in the canonical order
- Treat all module outputs as immutable
- Validate invariants between stages
- Fail closed (silent) on ambiguity
- Be fully deterministic

---

## 11. Authority

This contract is authoritative over all pipeline orchestration in ArcSight Wedge V1.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Individual Module Contracts
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

---

## 12. Contract Immutability

This contract is **V1 FINAL LOCKED**. Any change to pipeline behavior MUST be made by:

1. Updating this document first
2. Then updating relevant module contracts
3. Only then updating the orchestrator implementation

This ensures contract-first development and prevents drift.

