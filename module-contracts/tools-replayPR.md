# Module Contract — src/tools/replayPR.ts (V1 FINAL LOCKED)

**Layer:** tools

**Responsibility:** Testing and validation tool that replays historical PRs through the ArcSight Wedge analysis pipeline to validate zero false positives, measure comment rates, and verify runtime performance.

This module produces:

- Validation reports for historical PR replays
- False positive detection and counting
- Comment rate statistics
- Runtime performance metrics
- Determinism validation results

This module MUST NOT:

- modify core analysis behavior
- affect production PR processing
- introduce non-determinism in analysis (validation logic must be deterministic)
- skip validation criteria
- produce false validation results

It is a testing and validation tool module with tool-specific exceptions to core constraints.

---

## 1. Module Purpose & Responsibilities

**Location:** `tools/`

**Scope:** This module operates on historical PR data to validate ArcSight Wedge behavior and performance.

**Function:** Replays historical PRs through the analysis pipeline, validates results against expected outcomes, measures performance metrics, and reports validation statistics.

**This module is used for pre-release validation and design partner readiness.**

**Tool-Specific Exceptions:**

- **Console output allowed:** Tooling exception to core no-logging rule
- **File I/O allowed:** Can read PR lists, write reports, access git
- **API calls allowed:** Can fetch PR data from APIs (for tooling purposes)
- **Side effects allowed:** Can write validation reports, update test files

**Boundaries:**

- Does NOT modify core analysis modules
- Does NOT affect production PR processing
- Does NOT introduce non-determinism in analysis (validation must be deterministic)
- MUST validate against expected outcomes
- MUST measure performance metrics
- MUST report validation results

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface PRReplayEntry {
  repoPath: string;
  baseSha: string;
  headSha: string;
  changedFiles: string[];
  expectedCycles?: CanonicalCycle[];  // Optional: for false positive validation
  expectedComment?: boolean;  // Optional: whether comment should be created
}

interface ReplayConfig {
  prs: PRReplayEntry[];
  validateFalsePositives: boolean;  // If true, requires expectedCycles
  validateCommentRate: boolean;
  validateRuntime: boolean;
  validateDeterminism: boolean;  // Run each PR multiple times, compare outputs
}

function replayPRs(
  config: ReplayConfig,
  analyzeCyclesForPR: AnalyzeCyclesForPRFunction
): Promise<ReplayReport>
```

**Input assumptions:**

- `config.prs`: Array of PR entries to replay
  - Each entry contains PR metadata (baseSha, headSha, changedFiles, repoPath)
  - `expectedCycles` is optional but required if `validateFalsePositives === true`
  - `expectedComment` is optional but used for comment rate validation
- `config.validateFalsePositives`: If true, validates that no unexpected cycles are detected
- `config.validateCommentRate`: If true, measures and validates comment rate ≤ 20%
- `config.validateRuntime`: If true, measures and validates runtime p95 < 3s
- `config.validateDeterminism`: If true, runs each PR multiple times and compares outputs
- `analyzeCyclesForPR`: Function reference to the public API entry point

**Input validation:**

- If `config` is missing or `null`/`undefined` → throw Error
- If `config.prs` is missing or not an array → throw Error
- If `config.prs` is empty → return empty report (no error, but no validation)
- If `validateFalsePositives === true` and any PR missing `expectedCycles` → throw Error
- If `analyzeCyclesForPR` is missing or not a function → throw Error

### 2.2 Output

**Function signature:**

```ts
interface ReplayReport {
  totalPRs: number;
  analyzedPRs: number;
  failedPRs: number;
  falsePositives: number;  // Cycles detected that were not expected
  falseNegatives: number;  // Expected cycles that were not detected
  commentRate: number;  // Comments created / PRs analyzed (0-1)
  runtimeStats: {
    p50: number;
    p95: number;
    p99: number;
    max: number;
  };
  determinismViolations: number;  // PRs with non-deterministic outputs
  passed: boolean;  // Overall validation pass/fail
  errors: Array<{
    prIndex: number;
    error: string;
  }>;
}

function replayPRs(
  config: ReplayConfig,
  analyzeCyclesForPR: AnalyzeCyclesForPRFunction
): Promise<ReplayReport>
```

**Return types:**

- `Promise<ReplayReport>`: Resolves with validation report

**Output semantics:**

- `totalPRs`: Total number of PRs in input
- `analyzedPRs`: Number of PRs successfully analyzed
- `failedPRs`: Number of PRs that failed analysis
- `falsePositives`: Number of unexpected cycles detected (must be 0 for pass)
- `falseNegatives`: Number of expected cycles not detected (acceptable, but reported)
- `commentRate`: Ratio of PRs that would receive comments (must be ≤ 0.2 for pass)
- `runtimeStats`: Performance statistics in seconds
- `determinismViolations`: Number of PRs with non-deterministic outputs (must be 0 for pass)
- `passed`: Overall validation result (true if all criteria pass)
- `errors`: Array of errors encountered during replay

**Output rules:**

- MUST return structured report (JSON-serializable)
- MUST report all validation metrics
- MUST indicate pass/fail status
- MUST include error details for failed PRs
- Console output is allowed (tooling exception)

---

## 3. Validation Criteria

### 3.1 False Positive Validation

**Rule:** Zero false positives required for pass.

**Validation logic:**

- For each PR with `expectedCycles`:
  - Call `analyzeCyclesForPR` to get actual results
  - Compare `relevantCycles` to `expectedCycles`
  - Count cycles in `relevantCycles` that are NOT in `expectedCycles` as false positives
  - Count cycles in `expectedCycles` that are NOT in `relevantCycles` as false negatives (acceptable)
- If any false positives detected → `passed: false`

**Deterministic comparison:**

- Compare canonical cycles as strings (exact match)
- Sort both arrays before comparison (deterministic)
- Case-sensitive, exact string matching

### 3.2 Comment Rate Validation

**Rule:** Comment rate must be ≤ 20% (0.2) for pass.

**Validation logic:**

- For each PR:
  - Call `analyzeCyclesForPR` to get results
  - Determine if comment would be created:
    - `confidence >= 0.8` AND
    - `relevantCycles.length > 0` AND
    - All cycles have root-cause edges
  - Count PRs that would receive comments
- Calculate: `commentRate = commentsCreated / analyzedPRs`
- If `commentRate > 0.2` → `passed: false`

**Deterministic calculation:**

- Use exact same logic as production PR handler
- Count based on analysis results, not actual API calls
- Round to 4 decimal places for reporting

### 3.3 Runtime Performance Validation

**Rule:** Runtime p95 must be < 3 seconds for pass.

**Validation logic:**

- For each PR:
  - Measure runtime of `analyzeCyclesForPR` call
  - Record runtime in milliseconds (convert to seconds)
- Calculate percentiles: p50, p95, p99, max
- If `p95 >= 3.0` → `passed: false`

**Deterministic measurement:**

- Use `performance.now()` or `Date.now()` for timing (tooling exception)
- Measure wall-clock time (not CPU time)
- Round to 3 decimal places for reporting

### 3.4 Determinism Validation

**Rule:** Zero determinism violations required for pass.

**Validation logic (if `validateDeterminism === true`):

- For each PR:
  - Run `analyzeCyclesForPR` multiple times (e.g., 3 times)
  - Compare outputs (canonical cycles, root causes, confidence)
  - If any outputs differ → determinism violation
- Count PRs with determinism violations
- If any violations detected → `passed: false`

**Deterministic comparison:**

- Compare canonical cycles arrays (sorted, exact match)
- Compare root-cause edges arrays (sorted by canonicalCycle, then by from/to)
- Compare confidence scores (exact match, no tolerance)
- All comparisons must be byte-for-byte identical

---

## 4. Tool-Specific Behavior Rules

`replayPR.ts` MUST:

- Produce deterministic validation results (same input → same report)
- Use deterministic comparison logic (sorted arrays, exact matching)
- Report all validation metrics accurately
- Handle errors gracefully (continue processing other PRs)
- Provide clear error messages for failed PRs
- Support batch processing (multiple PRs)

`replayPR.ts` MAY (Tool-Specific Exceptions):

- Output to console (tooling exception to no-logging rule)
- Read files (PR lists, expected results)
- Write files (validation reports)
- Access git (checkout commits, read diffs)
- Make API calls (fetch PR data)
- Use timestamps for performance measurement
- Have side effects (writing reports, updating files)

`replayPR.ts` MUST NOT:

- Modify core analysis behavior
- Affect production PR processing
- Introduce non-determinism in analysis (validation logic must be deterministic)
- Skip validation criteria
- Produce false validation results
- Modify input PR data
- Cache analysis results (must call `analyzeCyclesForPR` fresh each time)

---

## 5. Error Handling Rules

`replayPR.ts` MUST:

- Continue processing other PRs if one fails
- Record errors in report (with PR index and error message)
- Distinguish between analysis failures and validation failures
- Provide clear error messages
- Never crash on individual PR failures

**Error categories:**

- **Analysis failures:** `analyzeCyclesForPR` throws or returns error
  - Record in `errors` array
  - Increment `failedPRs`
  - Continue with next PR
- **Validation failures:** Analysis succeeds but validation fails
  - Record in appropriate metric (falsePositives, commentRate, etc.)
  - Continue with next PR
- **Input errors:** Invalid PR data or configuration
  - Throw Error immediately (stop processing)

**Fail-safe principle:**

- When in doubt, record as error and continue
- Never skip validation due to errors
- Always report accurate error counts

---

## 6. Integration with Core Modules

**Dependencies:**

- MUST call `analyzeCyclesForPR` (public API entry point)
- MUST NOT modify core analysis modules
- MUST NOT access internal module functions directly
- MUST use dependency injection for `analyzeCyclesForPR` (testability)

**Integration flow:**

1. Load PR list from input (file or command-line)
2. For each PR:
   - Call `analyzeCyclesForPR(baseSha, headSha, changedFiles, repoPath)`
   - Measure runtime
   - Validate results against expected outcomes
   - Record metrics
3. Calculate aggregate statistics
4. Generate validation report
5. Output report (console and/or file)

**This module does NOT:**

- Modify core analysis logic
- Access internal module functions
- Bypass public API
- Cache analysis results
- Affect production behavior

---

## 7. Forbidden Behaviors

`replayPR.ts` MUST NOT:

- Modify core analysis behavior
- Affect production PR processing
- Introduce non-determinism in analysis (validation must be deterministic)
- Skip validation criteria
- Produce false validation results
- Modify input PR data
- Cache analysis results
- Access internal module functions directly
- Bypass public API
- Modify git repositories (read-only access)
- Delete or modify files (read-only access, except for report writing)

---

## 8. Edge Cases

### 8.1 Empty PR list

**Input:** `config.prs = []`

**Output:** Empty report with `totalPRs: 0`, `passed: true`

**Behavior:** Return empty report, no error

### 8.2 PR analysis failure

**Input:** PR that causes `analyzeCyclesForPR` to throw

**Output:** Error recorded in `errors` array, `failedPRs++`, continue with next PR

**Behavior:** Continue processing, don't stop on individual failures

### 8.3 Missing expected cycles

**Input:** `validateFalsePositives === true` but PR missing `expectedCycles`

**Output:** Error thrown immediately

**Behavior:** Stop processing, throw Error

### 8.4 Determinism validation with non-deterministic PR

**Input:** PR that produces different outputs across runs

**Output:** `determinismViolations++`, `passed: false`

**Behavior:** Record violation, continue processing

### 8.5 Very large PR list

**Input:** Hundreds or thousands of PRs

**Output:** Full report with all metrics

**Behavior:** Process all PRs (may be slow, but must complete)

### 8.6 Invalid repo path

**Input:** PR with invalid or inaccessible `repoPath`

**Output:** Error recorded, `failedPRs++`, continue

**Behavior:** Record error, continue with next PR

### 8.7 Missing git commits

**Input:** PR with `baseSha` or `headSha` that doesn't exist

**Output:** Error recorded, `failedPRs++`, continue

**Behavior:** Record error, continue with next PR

---

## 9. Performance Considerations

**Tool performance:**

- Tool itself may be slow for large PR lists (acceptable for tooling)
- Must not timeout or crash on large inputs
- Should provide progress indicators (console output allowed)

**Analysis performance:**

- Each PR analysis must meet runtime SLO/SLA (validated separately)
- Tool measures and reports performance, doesn't enforce it
- Tool may be slower than production (acceptable for validation)

---

## 10. Boundary Constraints

This module MUST depend on:

- Public API (`analyzeCyclesForPR` function)
- Node.js `fs` module (for reading PR lists, writing reports)
- Node.js `path` module (for path manipulation)

This module MAY depend on:

- Git libraries (for checking out commits, reading diffs)
- HTTP libraries (for fetching PR data from APIs)
- JSON parsing (for reading PR lists)

This module MUST NOT depend on:

- Internal analysis modules (must use public API only)
- Internal diff modules
- Internal PR modules
- Internal safety modules
- Internal snapshot modules

This module MUST remain a pure validation tool with no impact on core functionality.

---

## 11. Summary Checklist

`replayPR.ts` MUST:

✔ accept `ReplayConfig` and `analyzeCyclesForPR` inputs  
✔ validate PR list structure  
✔ replay each PR through `analyzeCyclesForPR`  
✔ validate false positives (zero required)  
✔ validate comment rate (≤ 20% required)  
✔ validate runtime performance (p95 < 3s required)  
✔ validate determinism (zero violations required, if enabled)  
✔ measure and report all metrics accurately  
✔ handle errors gracefully (continue processing)  
✔ produce deterministic validation results  
✔ output structured report (JSON-serializable)  
✔ support tool-specific exceptions (console output, file I/O)  
✔ use dependency injection for `analyzeCyclesForPR`  
✔ never modify core analysis behavior  
✔ never affect production PR processing  

`replayPR.ts` MUST NOT:

❌ modify core analysis behavior  
❌ affect production PR processing  
❌ introduce non-determinism in analysis  
❌ skip validation criteria  
❌ produce false validation results  
❌ modify input PR data  
❌ cache analysis results  
❌ access internal module functions  
❌ bypass public API  

---

## 12. Authority

This contract is authoritative over all implementation of `src/tools/replayPR.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract (Section 1.2: analyzeCyclesForPR)
- Testing Philosophy (Section 2.4: Replay Tests)
- Production Guide (Section 8: Release Gates)
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

