# Module Contract — src/confidence/computeConfidence.ts (V1 FINAL LOCKED)

**Layer:** confidence

**Responsibility:** Deterministic confidence score computation from segmentation quality metrics, evaluating repo quality without inspecting cycles, imports, or diff outputs.

This module produces:

- A deterministic confidence score in [0, 1]
- A High/Low bucket classification based on confidence threshold

This module MUST NOT:

- inspect cycles
- inspect imports
- inspect root-cause edges
- reference diff outputs
- depend on canonicalization
- access PR logic
- modify outputs
- log anything
- infer architecture
- add new signals
- maintain state across calls

It is a pure scoring module with hard isolation from cycle detection.

---

## 1. Module Purpose & Responsibilities

**Location:** `confidence/`

**Scope:** This module operates on segmentation quality metrics to compute confidence scores.

**Function:** Evaluates repo quality indicators (file coverage, alias resolution, monorepo status, import graph stability) and computes a confidence score that determines whether ArcSight should speak.

**This module implements the confidence gating rule: confidence ≥ 0.8 → High, otherwise Low (silent).**

**Hard Isolation Rule (from Module Boundaries):**

The confidence module MUST NOT:

- inspect cycles
- inspect imports
- inspect root-cause edges
- reference diff outputs
- depend on canonicalization

Confidence MAY ONLY depend on:

- segmentation quality
- file-count statistics
- alias resolution success/failure

**Boundaries:**

- Does NOT inspect cycle structure (hard isolation)
- Does NOT access PR logic directly
- Does NOT modify outputs (read-only computation)
- Does NOT detect cycles or perform diffing
- MUST compute confidence from segmentation quality only
- MUST be a pure function (no state)

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface SegmentationQuality {
  fileCount: number;
  analyzedFileCount: number;
  analyzedFileCoverage: number;
  aliasStatus: "ok" | "uncertain";
  isMonorepo: boolean;
  importGraphStable: boolean;
  unresolvedImportRatio: number;
}
```

**Input assumptions:**

- `fileCount`: Total number of JS/TS source files considered (non-negative integer)
- `analyzedFileCount`: Number of JS/TS files successfully analyzed (non-negative integer, ≤ fileCount)
- `analyzedFileCoverage`: Ratio (0..1) of analyzed files to total files (computed or provided)
- `aliasStatus`: "ok" if alias resolution unambiguous, "uncertain" if ambiguous
- `isMonorepo`: true if monorepo heuristics match, false otherwise
- `importGraphStable`: true if import graph is stable across repeated runs, false otherwise
- `unresolvedImportRatio`: Ratio (0..1) of unresolved imports (0 = all resolved, 1 = none resolved)

**Input validation:**

- If `q` is missing or `null`/`undefined` → return `0` (fail-safe)
- If any required field is missing or invalid type → return `0` (fail-safe)
- If `fileCount < 0` OR `analyzedFileCount < 0` → return `0` (fail-safe)
- If `analyzedFileCount > fileCount` → return `0` (fail-safe, invalid state)
- If `analyzedFileCoverage < 0` OR `analyzedFileCoverage > 1` → return `0` (fail-safe)
- If `unresolvedImportRatio < 0` OR `unresolvedImportRatio > 1` → return `0` (fail-safe)
- If `aliasStatus` is not "ok" or "uncertain" → return `0` (fail-safe)

### 2.2 Output

**Function signature:**

```ts
function computeConfidence(q: SegmentationQuality): number
function bucketConfidence(score: number): "High" | "Low"
```

**Return types:**

- `computeConfidence`: `number` in [0, 1]
- `bucketConfidence`: `"High" | "Low"`

**Output semantics:**

- `computeConfidence`: Returns confidence score in [0, 1]
  - `0`: Low confidence (silent mode)
  - `1`: Maximum confidence
  - `≥ 0.8`: High confidence (may speak)
  - `< 0.8`: Low confidence (silent)
- `bucketConfidence`: Returns "High" if `score >= 0.8`, "Low" otherwise

**Output rules:**

- `0` is the safe default (prefer silence over incorrect output)
- Score MUST be in [0, 1] (clamped if formula exceeds bounds)
- Must be deterministic (same input → same output)
- Never throws exceptions (returns 0 on errors)

---

## 3. Confidence Computation Rules

### 3.1 Zero-Return Conditions (MANDATORY)

`computeConfidence` MUST return `0` immediately if ANY of the following are true:

1. **File count too low:**
   - `fileCount < 10`

2. **Alias ambiguity:**
   - `aliasStatus === "uncertain"`

3. **Monorepo detected:**
   - `isMonorepo === true`

4. **Import graph unstable:**
   - `importGraphStable === false`

5. **Input validation failure:**
   - Context is invalid or malformed
   - Required fields missing or invalid types
   - Invalid numeric ranges

**Evaluation order (deterministic):**

1. Input validation (return 0 if invalid)
2. Check `fileCount < 10` (return 0 if true)
3. Check `aliasStatus === "uncertain"` (return 0 if true)
4. Check `isMonorepo === true` (return 0 if true)
5. Check `importGraphStable === false` (return 0 if true)
6. Compute formula (if all checks pass)
7. Clamp result to [0, 1]
8. Return score

This order MUST be fixed to ensure deterministic evaluation.

### 3.2 Confidence Formula (When All Zero-Return Conditions Pass)

**Formula (from v1-design-doc.md Section 8.1):**

```
confidence = 0.4 * analyzedFileCoverage + 0.3 * (1 - unresolvedImportRatio) + 0.3 * (importGraphStable ? 1 : 0)
```

**Formula breakdown:**

- `0.4 * analyzedFileCoverage`: Weighted file coverage (40% weight)
- `0.3 * (1 - unresolvedImportRatio)`: Weighted import resolution success (30% weight)
- `0.3 * (importGraphStable ? 1 : 0)`: Weighted graph stability (30% weight)

**Formula rules:**

- Use exact floating-point arithmetic (no rounding unless necessary for clamping)
- Clamp final result to [0, 1] (if formula exceeds bounds, clamp to nearest bound)
- Use deterministic arithmetic operations only
- Never use approximations or heuristics

**Note:** Since `importGraphStable` is already checked in zero-return conditions, if we reach the formula, `importGraphStable === true`, so the third term is always `0.3 * 1 = 0.3`.

### 3.3 Bucket Classification

**Function:** `bucketConfidence(score: number): "High" | "Low"`

**Rules:**

- If `score >= 0.8` → return `"High"`
- Otherwise → return `"Low"`

**Input validation:**

- If `score` is missing, `null`, `undefined`, or `NaN` → return `"Low"` (fail-safe)
- If `score < 0` → return `"Low"` (fail-safe)
- If `score > 1` → treat as `1.0` → return `"High"` (if >= 0.8) or `"Low"` (if < 0.8)

**Threshold:**

- Threshold is **inclusive** for "High": `score >= 0.8` → "High"
- Threshold is **exclusive** for "Low": `score < 0.8` → "Low"

---

## 4. Deterministic Behavior Rules

`computeConfidence.ts` MUST:

- Produce identical output for identical input
- Use deterministic arithmetic (standard floating-point, no rounding unless for clamping)
- Apply zero-return conditions deterministically (fixed evaluation order)
- Never use timestamps or dynamic content in decision logic
- Never depend on environment or runtime state
- Never maintain state across calls (pure function)
- Clamp formula results to [0, 1] deterministically

`computeConfidence.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Store confidence scores in module-level variables
- Cache results across calls
- Round intermediate calculations (only clamp final result if needed)

**Arithmetic determinism:**

- Use standard JavaScript floating-point arithmetic
- No rounding of intermediate values
- Only clamp final result to [0, 1] if formula exceeds bounds
- Same input MUST produce same output across machines and Node versions

**Evaluation order (deterministic):**

1. Input validation
2. Zero-return condition checks (in fixed order)
3. Formula computation
4. Result clamping
5. Return score

This order MUST be fixed to ensure deterministic evaluation.

---

## 5. Silent Mode Rules

`computeConfidence.ts` MUST return `0` (silent mode) when:

- ANY zero-return condition is met
- Input validation fails
- Any field is invalid or malformed
- Any uncertainty is present

**Silent mode behavior:**

- Return `0` immediately on any zero-return condition (short-circuit evaluation)
- Do not attempt partial computation
- Do not log errors
- Do not modify input
- Prefer false positives (err on side of silence)

`computeConfidence.ts` MUST return positive score (≥ 0) when:

- All zero-return conditions are absent
- All validations pass
- Input is valid and complete
- Formula can be computed

---

## 6. False-Positive Preference Rules

`computeConfidence.ts` MUST:

- Prefer returning `0` (silence) over allowing potentially incorrect scores
- NEVER allow invalid inputs to produce positive scores
- Treat ANY ambiguity as a zero-return trigger
- Return `0` on partial failures or inconsistencies
- Return `0` when context is invalid or incomplete

**If computation is ambiguous:**

- Return `0` (silent mode)
- Do not attempt to "fix" or interpret input
- Prefer false positive (incorrect silence) over false negative (incorrect score)

**Fail-safe principle:**

- When in doubt, return 0
- Invalid input → 0
- Missing data → 0
- Ambiguous indicators → 0

---

## 7. Forbidden Behaviors

`computeConfidence.ts` MUST NOT:

- Inspect cycles (hard isolation rule)
- Inspect imports (hard isolation rule)
- Inspect root-cause edges (hard isolation rule)
- Reference diff outputs (hard isolation rule)
- Depend on canonicalization (hard isolation rule)
- Access PR logic directly
- Modify outputs beyond computation
- Detect cycles or perform diffing
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps or dynamic content
- Depend on environment variables
- Throw exceptions (return 0 instead)
- Use nondeterministic APIs
- Access filesystem or external services
- Maintain state across calls (must be pure function)
- Store confidence scores in module-level variables
- Cache results across calls
- Round intermediate calculations (only clamp final result)
- Use approximations or heuristics in formula

---

## 8. Edge Cases

### 8.1 File count < 10

**Input:** `fileCount = 5`, all other fields valid

**Output:** `0` (silent mode)

**Behavior:** Too few files → return 0 immediately

### 8.2 Alias uncertainty

**Input:** `aliasStatus = "uncertain"`, all other fields valid

**Output:** `0` (silent mode)

**Behavior:** Alias ambiguity → return 0 immediately

### 8.3 Monorepo detected

**Input:** `isMonorepo = true`, all other fields valid

**Output:** `0` (silent mode)

**Behavior:** Monorepo → return 0 immediately

### 8.4 Import graph unstable

**Input:** `importGraphStable = false`, all other fields valid

**Output:** `0` (silent mode)

**Behavior:** Unstable graph → return 0 immediately

### 8.5 All conditions pass, formula computation

**Input:** All zero-return conditions pass, valid `SegmentationQuality`

**Output:** Score in [0, 1] from formula

**Behavior:** Compute formula, clamp to [0, 1], return score

### 8.6 Formula result > 1

**Input:** Formula produces value > 1 (edge case)

**Output:** `1.0` (clamped)

**Behavior:** Clamp to upper bound

### 8.7 Formula result < 0

**Input:** Formula produces value < 0 (edge case)

**Output:** `0.0` (clamped)

**Behavior:** Clamp to lower bound

### 8.8 Invalid input context

**Input:** Missing `SegmentationQuality` or invalid types

**Output:** `0` (silent mode)

**Behavior:** Invalid input → fail-safe return 0

### 8.9 Invalid numeric ranges

**Input:** `analyzedFileCount > fileCount` or `analyzedFileCoverage > 1`

**Output:** `0` (silent mode)

**Behavior:** Invalid state → fail-safe return 0

### 8.10 bucketConfidence: score >= 0.8

**Input:** `score = 0.8` or `score = 0.9`

**Output:** `"High"`

**Behavior:** Threshold is inclusive → "High"

### 8.11 bucketConfidence: score < 0.8

**Input:** `score = 0.79` or `score = 0.0`

**Output:** `"Low"`

**Behavior:** Below threshold → "Low"

### 8.12 bucketConfidence: invalid score

**Input:** `score = NaN` or `score = null`

**Output:** `"Low"` (fail-safe)

**Behavior:** Invalid score → fail-safe "Low"

---

## 9. Integration with Other Modules

**Caller responsibilities:**

- Caller must provide `SegmentationQuality` from analysis results
- Caller must check confidence score against threshold (≥ 0.8)
- Caller must activate silent mode if confidence < 0.8
- Caller may use `bucketConfidence` for classification

**Integration flow:**

1. Analysis modules produce `SegmentationQuality` (from `imports.ts` fileStats and other sources)
2. Caller calls `computeConfidence` with `SegmentationQuality`
3. If confidence < 0.8 → caller activates silent mode
4. If confidence ≥ 0.8 → caller may proceed (subject to other gates)

**This module does NOT:**

- Read from snapshots directly
- Access PR API directly
- Inspect cycles or imports
- Modify data structures
- Compute cycles or diffs

---

## 10. Boundary Constraints

This module MUST depend on:

- Type definitions (`SegmentationQuality` from `types.md`)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- safety modules (`safety/safetySwitch.ts`, etc.)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- analysis modules directly (`analysis/imports.ts`, etc.) - except for type imports
- other confidence modules (none exist)

This module MUST remain a pure scoring layer with hard isolation.

---

## 11. Summary Checklist

`computeConfidence.ts` MUST:

✔ accept `SegmentationQuality` input structure  
✔ validate input structure deterministically  
✔ check zero-return conditions (fileCount < 10, aliasStatus uncertain, isMonorepo, importGraphStable false)  
✔ compute formula when all conditions pass  
✔ clamp formula result to [0, 1]  
✔ return `0` on ANY zero-return condition  
✔ return `0` on validation failure (fail-safe)  
✔ use deterministic evaluation order  
✔ prefer false positives (err on side of silence)  
✔ treat invalid input as zero-return trigger  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic arithmetic operations only  
✔ never log anything  
✔ never access filesystem or external services  
✔ never modify input  
✔ never maintain state across calls  
✔ never inspect cycles, imports, or root-cause edges  
✔ implement `bucketConfidence` with threshold >= 0.8  

`computeConfidence.ts` MUST NOT:

❌ inspect cycles (hard isolation)  
❌ inspect imports (hard isolation)  
❌ inspect root-cause edges (hard isolation)  
❌ reference diff outputs (hard isolation)  
❌ depend on canonicalization (hard isolation)  
❌ access PR logic directly  
❌ modify outputs beyond computation  
❌ detect cycles or perform diffing  
❌ log or print anything  
❌ infer architecture  
❌ add new signals  
❌ use timestamps or dynamic content  
❌ depend on environment variables  
❌ throw exceptions  
❌ use nondeterministic APIs  
❌ maintain state across calls  
❌ store confidence scores in module-level variables  
❌ cache results across calls  
❌ round intermediate calculations  
❌ use approximations or heuristics  

---

## 12. Authority

This contract is authoritative over all implementation of `src/confidence/computeConfidence.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract (Section 3: Confidence Gating)
- API Contract (Section 2.3: computeConfidence & bucketConfidence)
- Module Boundaries (Section 2.3: confidence/ with Hard Isolation Rule)
- Types (`types.md` Section 4: SegmentationQuality)
- Determinism Requirements
- ADRs
- v1-design-doc.md (Section 8.1: Confidence Scoring & Segmentation Quality)

Any deviation from this contract MUST be approved via ADR and SSOT update.

