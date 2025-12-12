# Module Contract — src/safety/invariants.ts (V1 FINAL LOCKED)

**Layer:** safety

**Responsibility:** Deterministic validation of structural invariants for data structures (canonical cycles, import graphs, etc.) to ensure contract compliance and detect violations that should trigger silent mode.

This module produces:

- A deterministic boolean or error flags indicating whether invariants are satisfied
- Validation results for structural integrity checks

This module MUST NOT:

- inspect cycle structure beyond format validation
- access PR logic directly
- modify outputs
- detect cycles or perform diffing
- compute confidence
- log anything
- infer architecture
- add new signals
- maintain state across calls

It is a pure validation module.

---

## 1. Module Purpose & Responsibilities

**Location:** `safety/`

**Scope:** This module operates on data structures (canonical cycles, import graphs, etc.) to validate structural invariants.

**Function:** Validates that data structures conform to contract requirements (format, structure, consistency) and detects violations that should trigger silent mode.

**This module ensures data structure integrity before they are used by other modules.**

**Boundaries:**

- Does NOT inspect cycle structure beyond format validation
- Does NOT access PR logic directly
- Does NOT modify outputs (read-only validation)
- Does NOT detect cycles or perform diffing
- Does NOT compute confidence (delegated to confidence module)
- MUST validate format compliance
- MUST check structural consistency
- MUST be a pure function (no state)

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface InvariantsContext {
  canonicalCycles?: CanonicalCycle[];
  importGraph?: ImportGraphList;
  rootCauseEdges?: RootCauseEdge[];
}

interface InvariantsResult {
  allInvariantsSatisfied: boolean;
  violations: InvariantViolation[];
}

interface InvariantViolation {
  type: 'canonical_cycle_format' | 'duplicate_cycles' | 'import_graph_structure' | 'root_cause_structure' | 'determinism';
  message: string;
}
```

**Input assumptions:**

- `canonicalCycles`: Optional array of canonical cycle strings
  - May be empty array `[]`
  - May be `undefined` (not provided)
- `importGraph`: Optional ImportGraphList structure
  - May be empty array `[]`
  - May be `undefined` (not provided)
- `rootCauseEdges`: Optional array of root-cause edges
  - May be empty array `[]`
  - May be `undefined` (not provided)

**Input validation:**

- If `context` is missing or `null`/`undefined` → return `allInvariantsSatisfied: false` with violation
- If all fields are `undefined` or empty → return `allInvariantsSatisfied: true` (no data to validate)
- If any field is provided, validate it according to its invariant rules

### 2.2 Output

**Function signature:**

```ts
function validateInvariants(context: InvariantsContext): InvariantsResult
```

**Return type:** `InvariantsResult`

**Output semantics:**

- `allInvariantsSatisfied: true`: All provided data structures satisfy their invariants
- `allInvariantsSatisfied: false`: At least one invariant violation detected
- `violations`: Array of specific violations found (empty if all satisfied)

**Output rules:**

- `false` is the safe default (prefer detecting violations over missing them)
- `true` only when all provided data structures satisfy all applicable invariants
- Must be deterministic (same context → same result)
- Violations array must be sorted deterministically (by violation type, then message)

---

## 3. Invariant Rules

### 3.1 Canonical Cycle Format Invariants

**When `canonicalCycles` is provided:**

Each canonical cycle string MUST:

1. **Be a non-empty string**
   - Violation if: `typeof cycle !== 'string'` OR `cycle === ''`

2. **Use POSIX path separators (`/`)**
   - Violation if: cycle contains backslashes (`\`) that are not escaped
   - Note: This is a format check, not a path validation

3. **Use lowercase paths**
   - Violation if: cycle contains uppercase letters (except in string literals, but cycles should not contain string literals)
   - Note: This is a heuristic check; exact validation may be complex

4. **Use `" → "` as join separator**
   - Violation if: cycle does not contain `" → "` separator (at least once for multi-node cycles)
   - Exception: Single-node cycles may not have separator (but should be represented as `"node → node"`)

5. **Have valid structure**
   - Violation if: cycle cannot be split into at least 2 nodes using `" → "` separator
   - Minimum: `"node1 → node2"` (2 nodes)

**Duplicate cycles check:**

- Violation if: `canonicalCycles` array contains duplicate strings (same cycle appears multiple times)
- Comparison: Exact string equality (byte-for-byte)
- Must check after sorting (deterministic order)

### 3.2 ImportGraph Structure Invariants

**When `importGraph` is provided:**

Each ImportGraph entry MUST:

1. **Have `filePath` field**
   - Violation if: entry missing `filePath` OR `typeof filePath !== 'string'` OR `filePath === ''`

2. **Have `imports` field**
   - Violation if: entry missing `imports` OR `!Array.isArray(imports)`

3. **Have normalized `filePath`**
   - Violation if: `filePath` contains backslashes (`\`) OR contains uppercase letters
   - Note: This is a format check, not a full normalization validation

4. **Have normalized `imports`**
   - Violation if: any import in `imports` array contains backslashes (`\`) OR contains uppercase letters
   - Violation if: any import is not a string OR is empty string

5. **Have sorted `imports`**
   - Violation if: `imports` array is not sorted alphabetically (deterministic check)
   - Comparison: `imports[i] > imports[i+1]` for any `i`

6. **Have deduplicated `imports`**
   - Violation if: `imports` array contains duplicate strings
   - Comparison: Exact string equality

**Array-level invariants:**

- Violation if: `importGraph` contains duplicate `filePath` entries
- Comparison: Exact string equality on `filePath` field

### 3.3 Root-Cause Edge Structure Invariants

**When `rootCauseEdges` is provided:**

Each RootCauseEdge entry MUST:

1. **Have `from` field**
   - Violation if: entry missing `from` OR `typeof from !== 'string'` OR `from === ''`

2. **Have `to` field**
   - Violation if: entry missing `to` OR `typeof to !== 'string'` OR `to === ''`

3. **Have `canonicalCycle` field**
   - Violation if: entry missing `canonicalCycle` OR `typeof canonicalCycle !== 'string'` OR `canonicalCycle === ''`

4. **Have normalized `from` and `to`**
   - Violation if: `from` or `to` contains backslashes (`\`) OR contains uppercase letters

5. **Have valid `lineNumber` (if present)**
   - Violation if: `lineNumber` is present AND (`typeof lineNumber !== 'number'` OR `!isFinite(lineNumber)` OR `lineNumber < 1`)

6. **Have valid `importLine` (if present)**
   - Violation if: `importLine` is present AND `typeof importLine !== 'string'`

**Array-level invariants:**

- Violation if: `rootCauseEdges` contains duplicate entries (all fields match)
- Comparison: Compare `from`, `to`, `canonicalCycle`, `lineNumber`, `importLine` fields

### 3.4 Determinism Invariants

**When multiple data structures are provided:**

- Violation if: `canonicalCycles` and `rootCauseEdges` are both provided AND any `rootCauseEdge.canonicalCycle` does not exist in `canonicalCycles`
- This ensures consistency between cycles and root-cause edges

---

## 4. Deterministic Behavior Rules

`invariants.ts` MUST:

- Produce identical output for identical input context
- Use deterministic comparison logic (exact string equality, exact field comparison)
- Apply invariant checks deterministically (fixed evaluation order)
- Never use timestamps or dynamic content in decision logic
- Never depend on environment or runtime state
- Never maintain state across calls (pure function)
- Sort violations deterministically (by type, then message)

`invariants.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets (must sort before checking)
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Store validation results in module-level state
- Cache results across calls

**Evaluation order (deterministic):**

1. Input validation (context structure)
2. Canonical cycle format checks (if provided)
3. Canonical cycle duplicate checks (if provided)
4. ImportGraph structure checks (if provided)
5. Root-cause edge structure checks (if provided)
6. Determinism consistency checks (if multiple structures provided)
7. Sort violations deterministically
8. Return result

This order MUST be fixed to ensure deterministic evaluation.

---

## 5. Silent Mode Rules

`invariants.ts` MUST return `allInvariantsSatisfied: false` when:

- ANY invariant violation is detected
- Context structure is invalid
- Any data structure is malformed
- Any consistency check fails

**Violation behavior:**

- Return `allInvariantsSatisfied: false` immediately on first violation (but continue checking all invariants to collect all violations)
- Do not attempt to "fix" or repair violations
- Do not log errors
- Do not modify context
- Prefer detecting violations (err on side of caution)

`invariants.ts` MUST return `allInvariantsSatisfied: true` when:

- All provided data structures satisfy all applicable invariants
- No violations detected
- Context is valid (even if empty)

---

## 6. False-Positive Preference Rules

`invariants.ts` MUST:

- Prefer detecting violations (return `false`) over missing violations
- NEVER allow invalid data structures to pass validation
- Treat ANY ambiguity as a violation trigger
- Detect violations on partial failures or inconsistencies
- Return `false` when context is invalid or incomplete

**If validation is ambiguous:**

- Return `false` (violations detected)
- Do not attempt to "fix" or interpret context
- Prefer false positive (incorrect violation detection) over false negative (missing violation)

**Fail-safe principle:**

- When in doubt, detect violation
- Invalid context → violation
- Missing data → no violation (if field not provided, skip its checks)
- Ambiguous indicators → violation

---

## 7. Forbidden Behaviors

`invariants.ts` MUST NOT:

- Inspect cycle structure beyond format validation
- Access PR logic directly
- Modify outputs (read-only validation)
- Detect cycles or perform diffing
- Compute confidence (delegated to confidence module)
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps or dynamic content
- Depend on environment variables
- Throw exceptions (return result with violations instead)
- Use nondeterministic APIs
- Access filesystem or external services
- Maintain state across calls (must be pure function)
- Store validation results in module-level variables
- Cache results across calls
- Attempt to repair or fix violations
- Normalize or transform data (only validate format)

---

## 8. Edge Cases

### 8.1 Empty context

**Input:** `context = {}` or all fields `undefined`

**Output:** `{ allInvariantsSatisfied: true, violations: [] }`

**Behavior:** No data to validate → all invariants satisfied

### 8.2 Empty arrays

**Input:** `canonicalCycles = []`, `importGraph = []`, `rootCauseEdges = []`

**Output:** `{ allInvariantsSatisfied: true, violations: [] }`

**Behavior:** Empty arrays are valid → all invariants satisfied

### 8.3 Single canonical cycle

**Input:** `canonicalCycles = ["src/a.ts → src/b.ts → src/a.ts"]`

**Output:** `{ allInvariantsSatisfied: true, violations: [] }` (if format is valid)

**Behavior:** Single cycle → validate format only

### 8.4 Duplicate canonical cycles

**Input:** `canonicalCycles = ["a → b → a", "a → b → a"]`

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'duplicate_cycles', message: '...' }] }`

**Behavior:** Duplicates detected → violation

### 8.5 Invalid canonical cycle format

**Input:** `canonicalCycles = ["invalid format"]`

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'canonical_cycle_format', message: '...' }] }`

**Behavior:** Invalid format → violation

### 8.6 Invalid ImportGraph structure

**Input:** `importGraph = [{ filePath: "src/a.ts", imports: ["src/b.ts", "src/a.ts"] }]` (not sorted)

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'import_graph_structure', message: '...' }] }`

**Behavior:** Unsorted imports → violation

### 8.7 Invalid root-cause edge structure

**Input:** `rootCauseEdges = [{ from: "src/a.ts", to: "src/b.ts" }]` (missing `canonicalCycle`)

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'root_cause_structure', message: '...' }] }`

**Behavior:** Missing required field → violation

### 8.8 Inconsistent cycles and root-cause edges

**Input:** `canonicalCycles = ["a → b → a"]`, `rootCauseEdges = [{ canonicalCycle: "x → y → x", ... }]`

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'determinism', message: '...' }] }`

**Behavior:** Root-cause edge references non-existent cycle → violation

### 8.9 Multiple violations

**Input:** Multiple violations in different data structures

**Output:** `{ allInvariantsSatisfied: false, violations: [...] }` (all violations collected)

**Behavior:** All violations detected and reported

### 8.10 Null or undefined context

**Input:** `context = null` or `context = undefined`

**Output:** `{ allInvariantsSatisfied: false, violations: [{ type: 'canonical_cycle_format', message: 'Context is null or undefined' }] }`

**Behavior:** Invalid context → violation

---

## 9. Integration with Other Modules

**Caller responsibilities:**

- Caller must provide data structures to validate
- Caller must check `allInvariantsSatisfied` flag
- Caller must handle violations appropriately (typically trigger silent mode)
- Caller may use `violations` array for diagnostics (but not in production)

**Integration flow:**

1. Analysis modules produce data structures (canonical cycles, import graphs, etc.)
2. Before using data structures, caller calls `validateInvariants`
3. If `allInvariantsSatisfied === false` → trigger silent mode (do not proceed)
4. If `allInvariantsSatisfied === true` → proceed with data structures

**This module does NOT:**

- Read from snapshots directly
- Access PR API directly
- Modify data structures
- Fix violations automatically

---

## 10. Boundary Constraints

This module MUST depend on:

- Type definitions (`CanonicalCycle`, `ImportGraphList`, `RootCauseEdge` from `types.md` or imported modules)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- confidence modules (`confidence/computeConfidence.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- analysis modules directly (`analysis/imports.ts`, etc.)
- other safety modules (to avoid circular dependencies)

This module MUST remain a pure validation layer.

---

## 11. Summary Checklist

`invariants.ts` MUST:

✔ accept `InvariantsContext` input structure  
✔ validate context structure deterministically  
✔ check canonical cycle format invariants  
✔ check for duplicate canonical cycles  
✔ check ImportGraph structure invariants  
✔ check root-cause edge structure invariants  
✔ check determinism consistency invariants  
✔ return `allInvariantsSatisfied: false` on ANY violation  
✔ return `allInvariantsSatisfied: true` only when all invariants satisfied  
✔ collect all violations (not just first)  
✔ sort violations deterministically  
✔ use deterministic evaluation order  
✔ prefer detecting violations (err on side of caution)  
✔ treat invalid context as violation trigger  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic comparison operations only  
✔ never log anything  
✔ never access filesystem or external services  
✔ never modify context  
✔ never maintain state across calls  
✔ never attempt to repair violations  

`invariants.ts` MUST NOT:

❌ inspect cycle structure beyond format validation  
❌ access PR logic directly  
❌ modify outputs  
❌ detect cycles or perform diffing  
❌ compute confidence  
❌ log or print anything  
❌ infer architecture  
❌ add new signals  
❌ use timestamps or dynamic content  
❌ depend on environment variables  
❌ throw exceptions  
❌ use nondeterministic APIs  
❌ maintain state across calls  
❌ store validation results in module-level variables  
❌ cache results across calls  
❌ attempt to repair or fix violations  
❌ normalize or transform data  

---

## 12. Authority

This contract is authoritative over all implementation of `src/safety/invariants.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

