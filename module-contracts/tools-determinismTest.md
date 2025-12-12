# Module Contract — src/tools/determinismTest.ts (V1 FINAL LOCKED)

**Layer:** tools

**Responsibility:** Testing and validation tool that validates ArcSight Wedge determinism by running the same input multiple times and comparing outputs byte-for-byte, testing filesystem determinism, canonicalization invariance, and alias map stability.

This module produces:

- Determinism validation reports
- Repeated-run consistency validation
- Filesystem determinism validation
- Canonicalization invariance validation
- Alias map stability validation
- Mismatch detection and reporting

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

**Scope:** This module operates on test scenarios to validate ArcSight Wedge determinism guarantees.

**Function:** Runs the same input multiple times, compares outputs byte-for-byte, tests filesystem determinism with randomized file ordering, and validates canonicalization invariance and alias map stability.

**This module is used for pre-commit validation and CI enforcement of determinism requirements.**

**Tool-Specific Exceptions:**

- **Console output allowed:** Tooling exception to core no-logging rule
- **File I/O allowed:** Can read test configs, write reports, access git
- **API calls allowed:** Can fetch test data from APIs (for tooling purposes)
- **Side effects allowed:** Can write validation reports, update test files

**Boundaries:**

- Does NOT modify core analysis modules
- Does NOT affect production PR processing
- Does NOT introduce non-determinism in analysis (validation must be deterministic)
- MUST validate determinism through repeated runs
- MUST compare outputs byte-for-byte
- MUST report validation results

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface DeterminismTestScenario {
  repoPath: string;
  testType: 'commit' | 'pr';
  prData?: {
    baseSha: string;
    headSha: string;
    changedFiles: string[];
  };
}

interface DeterminismTestConfig {
  scenarios: DeterminismTestScenario[];
  numRuns?: number;  // Default: 3
  testFilesystemDeterminism?: boolean;  // Test with randomized file ordering
  testCanonicalization?: boolean;  // Validate canonicalization invariance
  testAliasStability?: boolean;  // Validate alias map stability
}

interface AnalyzeCyclesForCommitFunction {
  (repoPath: string): Promise<{
    canonicalCycles: CanonicalCycle[];
    importGraph: ImportGraphList;
    confidence: number;
  }>;
}

interface AnalyzeCyclesForPRFunction {
  (baseSha: string, headSha: string, changedFiles: string[], repoPath: string): Promise<PRCycleAnalysis>;
}

function runDeterminismTests(
  config: DeterminismTestConfig,
  analyzeCyclesForCommit: AnalyzeCyclesForCommitFunction,
  analyzeCyclesForPR: AnalyzeCyclesForPRFunction
): Promise<DeterminismTestReport>
```

**Input assumptions:**

- `config.scenarios`: Array of test scenarios to validate
  - Each scenario contains `repoPath` and `testType`
  - `prData` is required if `testType === 'pr'`
- `config.numRuns`: Number of times to run each scenario (default: 3)
- `config.testFilesystemDeterminism`: If true, tests with randomized file ordering
- `config.testCanonicalization`: If true, validates canonicalization invariance
- `config.testAliasStability`: If true, validates alias map stability
- `analyzeCyclesForCommit`: Function reference to public API entry point
- `analyzeCyclesForPR`: Function reference to public API entry point

**Input validation:**

- If `config` is missing or `null`/`undefined` → throw Error
- If `config.scenarios` is missing or not an array → throw Error
- If `config.scenarios` is empty → return empty report (no error, but no validation)
- If `testType === 'pr'` and `prData` is missing → throw Error
- If `analyzeCyclesForCommit` is missing or not a function → throw Error
- If `analyzeCyclesForPR` is missing or not a function → throw Error

### 2.2 Output

**Function signature:**

```ts
interface DeterminismTestReport {
  totalScenarios: number;
  testedScenarios: number;
  failedScenarios: number;
  violations: Array<{
    scenarioIndex: number;
    scenario: DeterminismTestScenario;
    violationType: 'repeated_run_mismatch' | 'filesystem_ordering_mismatch' | 'canonicalization_mismatch' | 'alias_stability_mismatch';
    details: string;
    runOutputs: unknown[];  // Outputs from each run for comparison
  }>;
  passed: boolean;  // Overall validation pass/fail
  errors: Array<{
    scenarioIndex: number;
    error: string;
  }>;
}

function runDeterminismTests(
  config: DeterminismTestConfig,
  analyzeCyclesForCommit: AnalyzeCyclesForCommitFunction,
  analyzeCyclesForPR: AnalyzeCyclesForPRFunction
): Promise<DeterminismTestReport>
```

**Return types:**

- `Promise<DeterminismTestReport>`: Resolves with validation report

**Output semantics:**

- `totalScenarios`: Total number of scenarios in input
- `testedScenarios`: Number of scenarios successfully tested
- `failedScenarios`: Number of scenarios that failed analysis
- `violations`: Array of determinism violations detected
  - `violationType`: Type of determinism violation
  - `details`: Human-readable description of the violation
  - `runOutputs`: Outputs from each run for comparison
- `passed`: Overall validation result (true if no violations detected)
- `errors`: Array of errors encountered during testing

**Output rules:**

- MUST return structured report (JSON-serializable)
- MUST report all violations with details
- MUST indicate pass/fail status
- MUST include error details for failed scenarios
- Console output is allowed (tooling exception)

---

## 3. Validation Criteria

### 3.1 Repeated-Run Consistency Validation

**Rule:** Same input → same output across repeated runs (byte-for-byte identical).

**Validation logic:**

- For each scenario:
  - Run analysis function `numRuns` times (default: 3)
  - Serialize each output to JSON string (deterministic serialization)
  - Compare JSON strings byte-for-byte
  - If any outputs differ → determinism violation
- Count scenarios with violations
- If any violations detected → `passed: false`

**Deterministic comparison:**

- Serialize outputs using `JSON.stringify` with sorted keys
- Compare strings byte-for-byte (exact match, no tolerance)
- Report which fields differ if mismatch detected

### 3.2 Filesystem Determinism Validation

**Rule:** Filesystem traversal ordering must not affect results.

**Validation logic (if `testFilesystemDeterminism === true`):

- For each scenario:
  - Run analysis with original file ordering
  - Run analysis with randomized file ordering (if possible)
  - Compare outputs byte-for-byte
  - If outputs differ → filesystem determinism violation
- Count scenarios with violations
- If any violations detected → `passed: false`

**Deterministic randomization:**

- Use fixed seed for randomization (if needed)
- Test with multiple different orderings (if possible)
- Compare all orderings to baseline

**Note:** Filesystem determinism testing may require mocking or special test setup. If not possible, this test may be skipped or marked as "not applicable".

### 3.3 Canonicalization Invariance Validation

**Rule:** Same cycles must produce same canonical representation.

**Validation logic (if `testCanonicalization === true`):

- For each scenario:
  - Extract canonical cycles from analysis output
  - Validate canonical cycle format (POSIX, lowercase, sorted, " → " separator)
  - Run analysis multiple times, compare canonical cycles
  - If canonical cycles differ across runs → canonicalization violation
- Count scenarios with violations
- If any violations detected → `passed: false`

**Deterministic validation:**

- Compare canonical cycles as sorted arrays (exact match)
- Validate format compliance (POSIX paths, lowercase, proper separator)
- Report format violations separately from consistency violations

### 3.4 Alias Map Stability Validation

**Rule:** Same alias map must produce same resolution.

**Validation logic (if `testAliasStability === true`):

- For each scenario:
  - Extract alias resolution results from analysis output
  - Run analysis multiple times with same alias map
  - Compare resolved paths
  - If resolutions differ across runs → alias stability violation
- Count scenarios with violations
- If any violations detected → `passed: false`

**Deterministic validation:**

- Compare resolved paths as sorted arrays (exact match)
- Validate that same alias maps produce same resolutions
- Report ambiguous resolutions separately

---

## 4. Tool-Specific Behavior Rules

`determinismTest.ts` MUST:

- Produce deterministic validation results (same input → same report)
- Use deterministic comparison logic (sorted JSON serialization, byte-for-byte matching)
- Report all violations with details
- Handle errors gracefully (continue processing other scenarios)
- Provide clear violation details
- Support batch processing (multiple scenarios)

`determinismTest.ts` MAY (Tool-Specific Exceptions):

- Output to console (tooling exception to no-logging rule)
- Read files (test configs, expected results)
- Write files (validation reports)
- Access git (checkout commits, read diffs)
- Make API calls (fetch test data)
- Use timestamps for performance measurement
- Have side effects (writing reports, updating files)

`determinismTest.ts` MUST NOT:

- Modify core analysis behavior
- Affect production PR processing
- Introduce non-determinism in analysis (validation logic must be deterministic)
- Skip validation criteria
- Produce false validation results
- Modify input test data
- Cache analysis results (must call analysis functions fresh each time)

---

## 5. Error Handling Rules

`determinismTest.ts` MUST:

- Continue processing other scenarios if one fails
- Record errors in report (with scenario index and error message)
- Distinguish between analysis failures and determinism violations
- Provide clear error messages
- Never crash on individual scenario failures

**Error categories:**

- **Analysis failures:** Analysis function throws or returns error
  - Record in `errors` array
  - Increment `failedScenarios`
  - Continue with next scenario
- **Determinism violations:** Analysis succeeds but outputs differ across runs
  - Record in `violations` array
  - Continue with next scenario
- **Input errors:** Invalid test data or configuration
  - Throw Error immediately (stop processing)

**Fail-safe principle:**

- When in doubt, record as violation and continue
- Never skip validation due to errors
- Always report accurate violation counts

---

## 6. Integration with Core Modules

**Dependencies:**

- MUST call `analyzeCyclesForCommit` or `analyzeCyclesForPR` (public API entry points)
- MUST NOT modify core analysis modules
- MUST NOT access internal module functions directly
- MUST use dependency injection for analysis functions (testability)

**Integration flow:**

1. Load test scenarios from input (file or function parameters)
2. For each scenario:
   - Call appropriate analysis function (`analyzeCyclesForCommit` or `analyzeCyclesForPR`)
   - Run multiple times (default: 3)
   - Compare outputs byte-for-byte
   - Record violations if any
3. Generate validation report
4. Output report (console and/or file)

**This module does NOT:**

- Modify core analysis logic
- Access internal module functions
- Bypass public API
- Cache analysis results
- Affect production behavior

---

## 7. Forbidden Behaviors

`determinismTest.ts` MUST NOT:

- Modify core analysis behavior
- Affect production PR processing
- Introduce non-determinism in analysis (validation must be deterministic)
- Skip validation criteria
- Produce false validation results
- Modify input test data
- Cache analysis results
- Access internal module functions directly
- Bypass public API
- Modify git repositories (read-only access)
- Delete or modify files (read-only access, except for report writing)

---

## 8. Edge Cases

### 8.1 Empty scenario list

**Input:** `config.scenarios = []`

**Output:** Empty report with `totalScenarios: 0`, `passed: true`

**Behavior:** Return empty report, no error

### 8.2 Analysis failure

**Input:** Scenario that causes analysis function to throw

**Output:** Error recorded in `errors` array, `failedScenarios++`, continue with next scenario

**Behavior:** Continue processing, don't stop on individual failures

### 8.3 Missing PR data

**Input:** `testType === 'pr'` but `prData` is missing

**Output:** Error thrown immediately

**Behavior:** Stop processing, throw Error

### 8.4 Determinism violation detected

**Input:** Scenario that produces different outputs across runs

**Output:** Violation recorded in `violations` array, `passed: false`, continue

**Behavior:** Record violation, continue processing

### 8.5 Very large scenario list

**Input:** Hundreds or thousands of scenarios

**Output:** Full report with all violations

**Behavior:** Process all scenarios (may be slow, but must complete)

### 8.6 Invalid repo path

**Input:** Scenario with invalid or inaccessible `repoPath`

**Output:** Error recorded, `failedScenarios++`, continue

**Behavior:** Record error, continue with next scenario

### 8.7 Filesystem determinism test not applicable

**Input:** Scenario where filesystem ordering cannot be randomized

**Output:** Test skipped, no violation recorded

**Behavior:** Skip test gracefully, continue with other validations

---

## 9. Performance Considerations

**Tool performance:**

- Tool itself may be slow for large scenario lists (acceptable for tooling)
- Must not timeout or crash on large inputs
- Should provide progress indicators (console output allowed)

**Analysis performance:**

- Each scenario analysis must meet runtime SLO/SLA (validated separately)
- Tool measures determinism, doesn't enforce performance
- Tool may be slower than production (acceptable for validation)

---

## 10. Boundary Constraints

This module MUST depend on:

- Public API (`analyzeCyclesForCommit` and `analyzeCyclesForPR` functions)
- Node.js `fs` module (for reading test configs, writing reports)
- Node.js `path` module (for path manipulation)

This module MAY depend on:

- Git libraries (for checking out commits, reading diffs)
- HTTP libraries (for fetching test data from APIs)
- JSON parsing (for reading test configs)

This module MUST NOT depend on:

- Internal analysis modules (must use public API only)
- Internal diff modules
- Internal PR modules
- Internal safety modules
- Internal snapshot modules

This module MUST remain a pure validation tool with no impact on core functionality.

---

## 11. Summary Checklist

`determinismTest.ts` MUST:

✔ accept `DeterminismTestConfig` and analysis function inputs  
✔ validate test scenario structure  
✔ run each scenario multiple times (default: 3)  
✔ compare outputs byte-for-byte (exact match)  
✔ validate repeated-run consistency  
✔ validate filesystem determinism (if enabled)  
✔ validate canonicalization invariance (if enabled)  
✔ validate alias map stability (if enabled)  
✔ report all violations with details  
✔ handle errors gracefully (continue processing)  
✔ produce deterministic validation results  
✔ output structured report (JSON-serializable)  
✔ support tool-specific exceptions (console output, file I/O)  
✔ use dependency injection for analysis functions  
✔ never modify core analysis behavior  
✔ never affect production PR processing  

`determinismTest.ts` MUST NOT:

❌ modify core analysis behavior  
❌ affect production PR processing  
❌ introduce non-determinism in analysis  
❌ skip validation criteria  
❌ produce false validation results  
❌ modify input test data  
❌ cache analysis results  
❌ access internal module functions  
❌ bypass public API  

---

## 12. Authority

This contract is authoritative over all implementation of `src/tools/determinismTest.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract (Section 1.1: analyzeCyclesForCommit, Section 1.2: analyzeCyclesForPR)
- Determinism Requirements (Section 11: Repeated-Run Consistency Rule)
- Testing Philosophy (Section 2.2: Determinism Tests)
- Local Dev Guide (Section 3.1: Determinism Suite)
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

