# Module Contract — src/index.ts (V1 FINAL LOCKED)

**Layer:** Public API

**Responsibility:** Orchestrate the analysis pipeline and expose the two public entry points required by the API Contract.

This module produces:

- `analyzeCyclesForCommit`: Pure structural analysis for a single commit
- `analyzeCyclesForPR`: PR-specific analysis comparing base and head commits

This module MUST NOT:

- modify core analysis modules
- introduce non-determinism
- log anything
- infer architecture
- cache results across calls
- maintain global state

It is the public API orchestration layer.

---

## 1. Module Purpose & Responsibilities

**Location:** `src/index.ts`

**Scope:** This module is the ONLY public API entry point for ArcSight Wedge V1. It exposes exactly two functions as required by the API Contract.

**Function:** Orchestrates the complete analysis pipeline by calling internal modules in the correct sequence, handling git operations for PR analysis, and applying safety checks.

**This module is the bridge between external callers (PR bots, tools) and internal analysis modules.**

**Boundaries:**

- MUST expose only the two public functions defined in API Contract
- MUST NOT expose internal module functions
- MUST orchestrate the pipeline deterministically
- MUST handle errors gracefully (silent mode)
- MAY use git commands via `child_process` for PR analysis
- MAY read alias configuration files (tsconfig.json/jsconfig.json)

---

## 2. Inputs & Outputs

### 2.1 Function 1: `analyzeCyclesForCommit`

**Function signature:**

```ts
export async function analyzeCyclesForCommit(
  repoPath: string
): Promise<{
  canonicalCycles: CanonicalCycle[];
  importGraph: ImportGraphList;
  confidence: number;
}>
```

**Input:**

- `repoPath: string` — Local filesystem path to the repository to analyze
  - Repository MUST be at a valid commit state
  - Path MUST be valid and accessible
  - Repository MUST contain JS/TS source files

**Output:**

- `canonicalCycles: CanonicalCycle[]` — All canonical cycles present at this commit (sorted alphabetically)
- `importGraph: ImportGraphList` — Resolved internal import graph (JS/TS only)
- `confidence: number` — Confidence score in [0, 1] for this snapshot

**Output rules:**

- If any step fails → return empty results (silent mode):
  - `canonicalCycles: []`
  - `importGraph: []`
  - `confidence: 0`
- MUST NOT throw exceptions to caller
- MUST return deterministic results (same input → same output)

### 2.2 Function 2: `analyzeCyclesForPR`

**Function signature:**

```ts
export async function analyzeCyclesForPR(
  baseSha: string,
  headSha: string,
  changedFiles: string[],
  repoPath: string
): Promise<PRCycleAnalysis>
```

**Input:**

- `baseSha: string` — Base commit SHA (must exist in repository)
- `headSha: string` — Head commit SHA (must exist in repository)
- `changedFiles: string[]` — List of POSIX-normalized, lowercased file paths changed in the PR
- `repoPath: string` — Local filesystem path to the repository

**Output:**

- `PRCycleAnalysis` (see `types.md`):
  - `relevantCycles: CanonicalCycle[]` — Canonical cycles that are new in this PR and satisfy all V1 filters
  - `rootCauses: RootCauseEdge[]` — Root-cause edges for those cycles
  - `confidence: number` — Repo confidence score for this analysis

**Output rules:**

- If any step fails → return empty results (silent mode):
  - `relevantCycles: []`
  - `rootCauses: []`
  - `confidence: 0`
- MUST NOT throw exceptions to caller
- MUST return deterministic results (same input → same output)

---

## 3. Pipeline Orchestration

### 3.1 `analyzeCyclesForCommit` Pipeline

**Step 1: Read Alias Configuration (Optional)**

- Attempt to read `tsconfig.json` or `jsconfig.json` from `repoPath`
- Extract `compilerOptions.paths` or `paths` mapping
- Normalize alias map keys and values (POSIX, lowercase)
- If file not found or invalid → use `null` for alias resolver (acceptable, not an error)
- If parsing fails → use `null` for alias resolver (acceptable, not an error)

**Step 2: Build Import Graph**

- Call: `buildImportGraph(repoPath, aliasResolver, aliasMap)`
- Receive: `ImportAnalysisResult` with `importGraph` and `fileStats`
- If call fails or returns error → return empty results (silent mode)

**Step 3: Detect Cycles**

- Call: `detectCycles(importGraph)`
- Receive: `CycleDetectionResult` with `canonicalCycles` and `errorDetected`
- If `errorDetected === true` → return empty results (silent mode)
- If call fails → return empty results (silent mode)

**Step 4: Compute Confidence**

- Build `SegmentationQuality` from `fileStats`:
  - `fileCount`: from `fileStats.fileCount`
  - `analyzedFileCount`: from `fileStats.analyzedFileCount`
  - `analyzedFileCoverage`: `analyzedFileCount / fileCount` (clamped to [0, 1])
  - `aliasStatus`: `fileStats.aliasAmbiguityDetected ? 'uncertain' : 'ok'`
  - `isMonorepo`: `false` (V1 does not detect monorepos)
  - `importGraphStable`: `true` (assumed stable for commit analysis)
  - `unresolvedImportRatio`: `fileStats.unresolvedImportCount / fileStats.totalImportCount` (clamped to [0, 1], handle division by zero)
- Call: `computeConfidence(segmentationQuality)`
- Receive: `confidence: number` in [0, 1]
- If call fails → return `confidence: 0`

**Step 5: Write Snapshot (Optional)**

- If `ARC_SIGHT_SNAPSHOT_PATH` environment variable is set:
  - Build `CycleSnapshot`:
    - `repoId`: Extract from `repoPath` (deterministic identifier)
    - `commitSha`: Get from `git rev-parse HEAD` (if git available)
    - `timestamp`: Current UTC timestamp in ISO-8601 format (second precision, no milliseconds)
    - `canonicalCycles`: from Step 3
    - `confidence`: from Step 4
  - Call: `writeSnapshot(snapshot, storagePath)`
  - If call fails → ignore error (snapshot writing is optional, does not affect return value)

**Step 6: Return Results**

- Return: `{ canonicalCycles, importGraph, confidence }`
- All arrays MUST be sorted (already sorted by called modules)

### 3.2 `analyzeCyclesForPR` Pipeline

**Step 1: Validate Inputs**

- Validate `baseSha` and `headSha` are non-empty strings
- Validate `changedFiles` is an array of strings
- Validate `repoPath` is a non-empty string
- If any validation fails → return empty results (silent mode)

**Step 2: Checkout Base Commit**

- Use `child_process.execSync('git checkout <baseSha>', { cwd: repoPath, stdio: 'ignore' })`
- If git command fails → return empty results (silent mode)
- If baseSha does not exist → return empty results (silent mode)

**Step 3: Analyze Base Commit**

- Call: `analyzeCyclesForCommit(repoPath)`
- Receive: `baseResult` with `canonicalCycles`, `importGraph`, `confidence`
- If call fails → return empty results (silent mode)
- Store `baseResult` for later use

**Step 4: Checkout Head Commit**

- Use `child_process.execSync('git checkout <headSha>', { cwd: repoPath, stdio: 'ignore' })`
- If git command fails → return empty results (silent mode)
- If headSha does not exist → return empty results (silent mode)

**Step 5: Analyze Head Commit**

- Call: `analyzeCyclesForCommit(repoPath)`
- Receive: `headResult` with `canonicalCycles`, `importGraph`, `confidence`
- If call fails → return empty results (silent mode)
- Store `headResult` for later use

**Step 6: Diff Cycles**

- Call: `diffCycles(baseResult.canonicalCycles, headResult.canonicalCycles)`
- Receive: `CycleDiffResult` with `newCycles`, `removedCycles`, `errorDetected`
- If `errorDetected === true` → return empty results (silent mode)
- If call fails → return empty results (silent mode)

**Step 7: Filter Cycles by Size (2–5 nodes)**

- For each cycle in `newCycles`:
  - Parse cycle: `cycle.split(' → ')`
  - Count nodes (length of split array)
  - If node count is 2, 3, 4, or 5 → include in `filteredCycles`
  - Otherwise → exclude
- Result: `filteredCycles: CanonicalCycle[]` (sorted alphabetically)

**Step 8: Filter Cycles by PR-Changed Files**

- For each cycle in `filteredCycles`:
  - Parse cycle: `cycle.split(' → ')`
  - Check if ANY node (file path) is in `changedFiles` set
  - If at least one node matches → include in `relevantCycles`
  - Otherwise → exclude
- Result: `relevantCycles: CanonicalCycle[]` (sorted alphabetically)

**Step 9: Generate Diff Hunks**

- Use `child_process.execSync('git diff <baseSha>..<headSha>', { cwd: repoPath, encoding: 'utf8' })`
- Parse git diff output to extract:
  - File paths (normalized, POSIX, lowercase)
  - Added lines with line numbers
- Build `DiffHunk[]`:
  ```ts
  interface DiffHunk {
    filePath: string;  // POSIX-normalized, lowercased
    addedLines: Array<{
      lineNumber: number;  // 1-based line number in head file
      content: string;      // Full line content
    }>;
  }
  ```
- If git diff fails → return empty results (silent mode)
- If parsing fails → return empty results (silent mode)

**Step 10: Find Root Causes**

- Call: `findRootCauses(relevantCycles, changedFiles, diffHunks, baseResult.importGraph, headResult.importGraph)`
- Receive: `RootCauseResult` with `rootCauseEdges` and `errorDetected`
- If `errorDetected === true` → return empty results (silent mode)
- If call fails → return empty results (silent mode)

**Step 11: Filter Cycles by Root Cause Availability**

- For each cycle in `relevantCycles`:
  - Check if `rootCauseEdges` contains an edge with `canonicalCycle === cycle`
  - If root cause exists → include in final `relevantCycles`
  - Otherwise → exclude (cycle is non-attributable for V1)
- Filter `rootCauseEdges` to only include edges for cycles in final `relevantCycles`
- Result: `relevantCycles` and `rootCauses` are paired (1:1 relationship)

**Step 12: Build Safety Switch Context**

- Build `SafetySwitchContext`:
  - `determinism.repeatedRunConsistent`: `true` (assumed for V1, validated by determinism test tool)
  - `determinism.canonicalizationStable`: `true` (validated by cycle detection)
  - `determinism.importGraphStable`: `true` (assumed for V1)
  - `performance.runtimeSeconds`: Measure runtime (optional, for future use)
  - `quality.aliasAmbiguityDetected`: from `baseResult` or `headResult` fileStats
  - `quality.importGraphComplete`: `true` if no errors in import graph building
  - `quality.rootCauseDetectionStable`: `true` if root cause detection succeeded
  - `errors.analysisFailed`: `false` if both base and head analysis succeeded
  - `errors.canonicalizationFailed`: `false` if cycle detection succeeded
  - `errors.rootCauseDetectionFailed`: `false` if root cause detection succeeded

**Step 13: Check Safety Switch**

- Call: `shouldSilence(safetySwitchContext)`
- If `shouldSilence === true` → return empty results (silent mode)
- If call fails → return empty results (silent mode)

**Step 14: Return Results**

- Return: `{ relevantCycles, rootCauses, confidence: headResult.confidence }`
- All arrays MUST be sorted (already sorted by called modules)

---

## 4. Error Handling Rules

`index.ts` MUST:

- Catch ALL exceptions from internal module calls
- Convert ALL errors to silent mode (empty results)
- NEVER throw exceptions to caller
- NEVER log errors or warnings
- Use `executeSafely` for operations that may throw
- Use `shouldSilenceOnError` for final validation if needed

**Error categories:**

- **Git command failures:** Return empty results (silent mode)
- **Module call failures:** Return empty results (silent mode)
- **Input validation failures:** Return empty results (silent mode)
- **Parsing failures:** Return empty results (silent mode)
- **Safety switch activation:** Return empty results (silent mode)

**Fail-safe principle:**

- When in doubt, return empty results
- Never return partial or speculative data
- Silent mode is always the safe default

---

## 5. Git Operations

`index.ts` MAY use git commands via Node.js `child_process` module:

- `git checkout <sha>` — Checkout specific commit
- `git diff <baseSha>..<headSha>` — Get diff between commits
- `git rev-parse HEAD` — Get current commit SHA

**Git operation rules:**

- MUST use `execSync` (synchronous, deterministic)
- MUST set `cwd: repoPath` for all commands
- MUST handle git command failures gracefully (return empty results)
- MUST NOT assume git is available (handle gracefully if not)
- MUST NOT modify repository state beyond checkout (read-only analysis)

**Git diff parsing:**

- MUST parse unified diff format
- MUST extract file paths (normalize to POSIX, lowercase)
- MUST extract added lines with line numbers (1-based)
- MUST handle binary files gracefully (skip)
- MUST handle rename/move operations (treat as delete + add for V1)

---

## 6. Alias Configuration Reading

`index.ts` MAY read alias configuration from:

- `tsconfig.json` — TypeScript configuration (look for `compilerOptions.paths`)
- `jsconfig.json` — JavaScript configuration (look for `paths`)

**Alias reading rules:**

- MUST attempt to read from `repoPath` root
- MUST handle missing files gracefully (use `null` for alias resolver)
- MUST handle invalid JSON gracefully (use `null` for alias resolver)
- MUST normalize alias map keys and values (POSIX, lowercase)
- MUST NOT throw if alias config is missing or invalid

**Alias map structure:**

```ts
Record<string, string>  // alias → resolved path
```

Example: `{ "@/*": "src/*", "~/*": "src/*" }`

---

## 7. Snapshot Writing (Optional)

`analyzeCyclesForCommit` MAY write snapshots if:

- `ARC_SIGHT_SNAPSHOT_PATH` environment variable is set
- Git is available to get commit SHA
- All previous steps succeeded

**Snapshot writing rules:**

- MUST be append-only (no read API in V1)
- MUST use deterministic timestamp (ISO-8601 UTC, second precision)
- MUST handle write failures gracefully (ignore, do not affect return value)
- MUST NOT block analysis pipeline (async write, do not await)

---

## 8. Cycle Filtering Rules

### 8.1 Size Filtering (2–5 nodes)

- Parse cycle: `cycle.split(' → ')`
- Count nodes: `nodes.length`
- Include if: `nodes.length >= 2 && nodes.length <= 5`
- Exclude otherwise

### 8.2 PR-Changed Files Filtering

- Parse cycle: `cycle.split(' → ')`
- Check if ANY node is in `changedFiles` set (case-insensitive, POSIX-normalized comparison)
- Include if: at least one node matches
- Exclude otherwise

### 8.3 Root Cause Availability Filtering

- For each cycle, check if `rootCauseEdges` contains an edge with matching `canonicalCycle`
- Include if: root cause edge exists
- Exclude otherwise (non-attributable cycle)

---

## 9. Determinism Guarantees

`index.ts` MUST:

- Produce identical output for identical input
- Use deterministic git operations (no timestamps in commands)
- Sort all arrays before returning
- Normalize all paths consistently (POSIX, lowercase)
- Use deterministic alias resolution
- Avoid any randomness or non-deterministic operations

**Determinism validation:**

- Repeated calls with same input MUST produce byte-for-byte identical output
- Git checkout operations MUST be deterministic (same commit → same state)
- Diff parsing MUST be deterministic (same diff → same hunks)

---

## 10. Boundary Constraints

This module MUST depend on:

- `analysis/imports.ts` — `buildImportGraph`
- `analysis/cycles.ts` — `detectCycles`
- `diff/cycleDiff.ts` — `diffCycles`
- `diff/rootCause.ts` — `findRootCauses`, `DiffHunk`
- `confidence/computeConfidence.ts` — `computeConfidence`
- `safety/safetySwitch.ts` — `shouldSilence`
- `safety/errorPath.ts` — `executeSafely`, `shouldSilenceOnError`
- `snapshot/snapshotWriter.ts` — `writeSnapshot` (optional)
- Node.js `child_process` — Git operations
- Node.js `fs` — Alias config reading

This module MUST NOT depend on:

- `pr/` modules (PR handling is separate)
- Internal implementation details of analysis modules
- External libraries beyond Node.js standard library

This module MUST remain a pure orchestration layer with no business logic.

---

## 11. Edge Cases

### 11.1 Empty repository

**Input:** Repository with no JS/TS source files

**Output:** `{ canonicalCycles: [], importGraph: [], confidence: 0 }`

**Behavior:** Return empty results (not an error)

### 11.2 Git not available

**Input:** `analyzeCyclesForPR` called but git is not installed

**Output:** Empty results (silent mode)

**Behavior:** Return empty results, do not throw

### 11.3 Invalid commit SHA

**Input:** `baseSha` or `headSha` does not exist in repository

**Output:** Empty results (silent mode)

**Behavior:** Return empty results, do not throw

### 11.4 No cycles found

**Input:** Repository with no dependency cycles

**Output:** `{ canonicalCycles: [], importGraph: [...], confidence: <value> }`

**Behavior:** Return empty cycles array (not an error)

### 11.5 No new cycles in PR

**Input:** PR that does not introduce new cycles

**Output:** `{ relevantCycles: [], rootCauses: [], confidence: <value> }`

**Behavior:** Return empty results (not an error)

### 11.6 Alias config missing

**Input:** Repository without `tsconfig.json` or `jsconfig.json`

**Output:** Normal results (alias resolution skipped)

**Behavior:** Continue without alias resolution (acceptable)

### 11.7 Snapshot write failure

**Input:** Snapshot writing fails (permissions, disk full, etc.)

**Output:** Normal results (snapshot write ignored)

**Behavior:** Continue without snapshot (optional operation)

---

## 12. Forbidden Behaviors

`index.ts` MUST NOT:

- Modify core analysis modules
- Introduce non-determinism
- Log anything
- Infer architecture
- Cache results across calls
- Maintain global state
- Throw exceptions to caller
- Modify repository state (beyond checkout)
- Access internal module functions directly
- Bypass public API contracts
- Add new public functions beyond the two required

---

## 13. Summary Checklist

`index.ts` MUST:

✔ Expose exactly two public functions (`analyzeCyclesForCommit`, `analyzeCyclesForPR`)  
✔ Orchestrate the complete analysis pipeline  
✔ Handle git operations for PR analysis  
✔ Read alias configuration (if available)  
✔ Apply cycle filtering (size 2–5, PR-changed files, root cause availability)  
✔ Apply safety switch checks  
✔ Handle all errors gracefully (silent mode)  
✔ Return deterministic results  
✔ Write snapshots optionally (if configured)  
✔ Never throw exceptions to caller  
✔ Never log anything  

`index.ts` MUST NOT:

❌ Modify core analysis modules  
❌ Introduce non-determinism  
❌ Log anything  
❌ Cache results  
❌ Maintain global state  
❌ Throw exceptions  
❌ Add new public functions  

---

## 14. Authority

This contract is authoritative over all implementation of `src/index.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract (Section 1: Public Entry Points)
- Module Boundaries
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

