# Module Contract — src/safety/cooldown.ts (V1 FINAL LOCKED)

**Layer:** safety

**Responsibility:** Deterministic cooldown filtering to prevent repeated warnings for the same cycle across PR pushes, implementing the "Repeated Cycle Handling" rule from the Safety Wedge Contract.

This module produces:

- A deterministic boolean for each cycle indicating whether it should be warned about
- Filtered cycle list that respects cooldown rules

This module MUST NOT:

- inspect cycle structure beyond canonical comparison
- access PR logic directly
- modify outputs
- detect cycles or perform diffing
- compute confidence
- log anything
- infer architecture
- add new signals
- maintain state across calls

It is a pure filtering module.

---

## 1. Module Purpose & Responsibilities

**Location:** `safety/`

**Scope:** This module operates on canonical cycles and root-cause edges to determine if a cycle should be warned about based on previous warnings.

**Function:** Filters cycles to prevent repeated warnings for the same cycle across multiple PR pushes, unless the cycle disappeared and reappeared or the root-cause edge changed.

**This module implements the "Repeated Cycle Handling" rule from the Safety Wedge Contract Section 1.**

**Boundaries:**

- Does NOT inspect cycle structure beyond canonical string comparison
- Does NOT access PR logic directly (receives previous cycles as input)
- Does NOT modify outputs (read-only evaluation)
- Does NOT detect cycles or perform diffing
- Does NOT compute confidence (delegated to confidence module)
- MUST compare canonical cycles deterministically
- MUST compare root-cause edges deterministically
- MUST be a pure function (no state)

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface CooldownContext {
  canonicalCycle: CanonicalCycle;
  rootCauseEdge: RootCauseEdge;
  previousCycles: CanonicalCycle[];
  previousRootCauses: RootCauseEdge[];
}
```

**Input assumptions:**

- `canonicalCycle`: Current canonical cycle string (normalized, lowercase, " → " joined)
- `rootCauseEdge`: Current root-cause edge for this cycle
  - Must have `canonicalCycle` field matching `canonicalCycle` input
  - Must have `from` and `to` fields (normalized paths)
  - May have `lineNumber` and `importLine` fields
- `previousCycles`: Array of canonical cycles from previous PR push or snapshot
  - Empty array `[]` if no previous state exists (first push)
  - Must be sorted alphabetically (deterministic)
- `previousRootCauses`: Array of root-cause edges from previous PR push or snapshot
  - Empty array `[]` if no previous state exists (first push)
  - Must have matching `canonicalCycle` field for each corresponding cycle in `previousCycles`

**Input validation:**

- If `canonicalCycle` is missing, empty string, or invalid → return `false` (suppress, fail-safe)
- If `rootCauseEdge` is missing or invalid → return `false` (suppress, fail-safe)
- If `rootCauseEdge.canonicalCycle !== canonicalCycle` → return `false` (suppress, mismatch)
- If `previousCycles` is missing → treat as empty array `[]` (first push)
- If `previousRootCauses` is missing → treat as empty array `[]` (first push)
- If `previousCycles` and `previousRootCauses` lengths don't match → treat as empty arrays (invalid state)

### 2.2 Output

**Function signature:**

```ts
function shouldWarnCycle(context: CooldownContext): boolean
```

**Return type:** `boolean`

**Output semantics:**

- `true`: Cycle should be warned about (new cycle, or cycle disappeared and reappeared, or root-cause edge changed)
- `false`: Cycle should be suppressed (same cycle with same root-cause edge from previous push)

**Output rules:**

- `false` is the safe default (prefer suppression over incorrect warnings)
- `true` only when cycle is new OR cycle disappeared and reappeared OR root-cause edge changed
- Must be deterministic (same context → same result)

---

## 3. Cooldown Rules (From Safety Wedge Contract)

**Cycle should be warned about (`true`) when:**

1. **Cycle is new:**
   - `canonicalCycle` is not present in `previousCycles`
   - First PR push (empty `previousCycles`)

2. **Cycle disappeared and reappeared:**
   - `canonicalCycle` was in `previousCycles` but was removed (not present in current analysis)
   - Then `canonicalCycle` reappears in current analysis
   - **Note:** This module receives only current cycle, so this case is handled by caller comparing current vs previous cycles

3. **Root-cause edge changed:**
   - `canonicalCycle` is present in `previousCycles`
   - Current `rootCauseEdge` differs from previous `rootCauseEdge` for the same cycle
   - Comparison: `rootCauseEdge.from !== previousRootCause.from` OR `rootCauseEdge.to !== previousRootCause.to` OR `rootCauseEdge.lineNumber !== previousRootCause.lineNumber` OR `rootCauseEdge.importLine !== previousRootCause.importLine`

**Cycle should be suppressed (`false`) when:**

- `canonicalCycle` is present in `previousCycles`
- Current `rootCauseEdge` matches previous `rootCauseEdge` for the same cycle
- Cycle has not disappeared and reappeared (handled by caller)

**Evaluation algorithm:**

1. Validate inputs (return `false` if invalid)
2. Check if cycle is new (not in `previousCycles`) → return `true`
3. Find matching cycle in `previousCycles` (exact string match)
4. Find corresponding root-cause edge in `previousRootCauses`
5. Compare current `rootCauseEdge` with previous `rootCauseEdge`
6. If root-cause edge changed → return `true`
7. If root-cause edge unchanged → return `false` (suppress)

---

## 4. Deterministic Behavior Rules

`cooldown.ts` MUST:

- Produce identical output for identical input context
- Use deterministic comparison logic (exact string equality for cycles, exact field comparison for root-cause edges)
- Apply cooldown rules deterministically (fixed evaluation order)
- Never use timestamps or dynamic content in decision logic
- Never depend on environment or runtime state
- Never maintain state across calls (pure function)

`cooldown.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets (must sort inputs)
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Store previous cycles in module-level state
- Cache results across calls

**Comparison rules (deterministic):**

- Canonical cycles: Exact string equality (case-sensitive, whitespace-sensitive)
- Root-cause edges: Compare all fields deterministically:
  - `from`: Exact string equality
  - `to`: Exact string equality
  - `lineNumber`: Exact numeric equality (or both `undefined`/`null`)
  - `importLine`: Exact string equality (or both `undefined`/`null`/empty)
  - `canonicalCycle`: Must match (already validated in input)

**Evaluation order (deterministic):**

1. Input validation
2. Check if cycle is new (not in previous cycles)
3. Find matching previous cycle (exact string match)
4. Find corresponding previous root-cause edge
5. Compare root-cause edges (all fields)
6. Return boolean result

This order MUST be fixed to ensure deterministic evaluation.

---

## 5. Root-Cause Edge Comparison Rules

**Root-cause edges are considered "changed" if ANY of the following differ:**

1. `from` field: Current `rootCauseEdge.from !== previousRootCause.from`
2. `to` field: Current `rootCauseEdge.to !== previousRootCause.to`
3. `lineNumber` field:
   - Current `lineNumber` is `undefined`/`null` AND previous `lineNumber` is not `undefined`/`null` → changed
   - Current `lineNumber` is not `undefined`/`null` AND previous `lineNumber` is `undefined`/`null` → changed
   - Both are defined: `current !== previous` → changed
   - Both are `undefined`/`null` → unchanged (for this field)
4. `importLine` field:
   - Current `importLine` is `undefined`/`null`/empty AND previous `importLine` is not `undefined`/`null`/empty → changed
   - Current `importLine` is not `undefined`/`null`/empty AND previous `importLine` is `undefined`/`null`/empty → changed
   - Both are defined: `current !== previous` → changed
   - Both are `undefined`/`null`/empty → unchanged (for this field)

**Root-cause edges are considered "unchanged" only when:**

- `from` matches
- `to` matches
- `lineNumber` matches (both defined and equal, or both `undefined`/`null`)
- `importLine` matches (both defined and equal, or both `undefined`/`null`/empty)

**Comparison is case-sensitive and whitespace-sensitive for string fields.**

---

## 6. Silent Mode Rules

`cooldown.ts` MUST return `false` (suppress) when:

- Input validation fails
- Cycle is invalid or malformed
- Root-cause edge is invalid or malformed
- Previous state is invalid or malformed
- Any uncertainty is present

**Suppression behavior:**

- Return `false` immediately on any validation failure (short-circuit evaluation)
- Do not attempt partial evaluation
- Do not log errors
- Do not modify context
- Prefer false positives (err on side of suppression)

`cooldown.ts` MUST return `true` (warn) when:

- Cycle is new (not in previous cycles)
- Root-cause edge changed for existing cycle
- All validations pass and cycle should be warned about

---

## 7. False-Positive Preference Rules

`cooldown.ts` MUST:

- Prefer suppressing warnings (return `false`) over allowing potentially incorrect warnings
- NEVER allow warning when uncertain
- Treat ANY ambiguity as suppression trigger
- Suppress on partial failures or inconsistencies
- Return `false` when context is invalid or incomplete

**If evaluation is ambiguous:**

- Return `false` (suppress)
- Do not attempt to "fix" or interpret context
- Prefer false positive (incorrect suppression) over false negative (incorrect warning)

**Fail-safe principle:**

- When in doubt, suppress
- Invalid context → suppress
- Missing data → suppress
- Ambiguous indicators → suppress

---

## 8. Forbidden Behaviors

`cooldown.ts` MUST NOT:

- Inspect cycle structure beyond canonical string comparison
- Access PR logic directly (receives previous cycles as input)
- Modify outputs (read-only evaluation)
- Detect cycles or perform diffing
- Compute confidence (delegated to confidence module)
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps or dynamic content
- Depend on environment variables
- Throw exceptions (return boolean instead)
- Use nondeterministic APIs
- Access filesystem or external services
- Maintain state across calls (must be pure function)
- Store previous cycles in module-level variables
- Cache results across calls
- Read from snapshots directly (receives previous cycles as input)
- Access PR API directly (receives previous cycles as input)

---

## 9. Edge Cases

### 9.1 First PR push (no previous state)

**Input:** `previousCycles = []`, `previousRootCauses = []`

**Output:** `true` (warn, cycle is new)

**Behavior:** Empty previous state → all cycles are new → warn about all

### 9.2 Cycle is new

**Input:** `canonicalCycle` not in `previousCycles`

**Output:** `true` (warn)

**Behavior:** New cycle → always warn

### 9.3 Cycle exists with same root-cause edge

**Input:** `canonicalCycle` in `previousCycles`, matching `rootCauseEdge` in `previousRootCauses`

**Output:** `false` (suppress)

**Behavior:** Same cycle, same root-cause → suppress (cooldown active)

### 9.4 Cycle exists with different root-cause edge

**Input:** `canonicalCycle` in `previousCycles`, different `rootCauseEdge` from `previousRootCauses`

**Output:** `true` (warn)

**Behavior:** Same cycle, different root-cause → warn (cooldown lifted)

### 9.5 Root-cause edge: `from` changed

**Input:** Same cycle, `rootCauseEdge.from !== previousRootCause.from`

**Output:** `true` (warn)

**Behavior:** Root-cause changed → warn

### 9.6 Root-cause edge: `to` changed

**Input:** Same cycle, `rootCauseEdge.to !== previousRootCause.to`

**Output:** `true` (warn)

**Behavior:** Root-cause changed → warn

### 9.7 Root-cause edge: `lineNumber` changed

**Input:** Same cycle, `rootCauseEdge.lineNumber !== previousRootCause.lineNumber`

**Output:** `true` (warn)

**Behavior:** Root-cause changed → warn

### 9.8 Root-cause edge: `importLine` changed

**Input:** Same cycle, `rootCauseEdge.importLine !== previousRootCause.importLine`

**Output:** `true` (warn)

**Behavior:** Root-cause changed → warn

### 9.9 Missing previous root-cause edge

**Input:** `canonicalCycle` in `previousCycles`, but no matching `rootCauseEdge` in `previousRootCauses`

**Output:** `true` (warn, treat as new root-cause)

**Behavior:** Previous state incomplete → warn (fail-safe: prefer warning over suppression)

### 9.10 Invalid input context

**Input:** Missing `canonicalCycle` or `rootCauseEdge`, or invalid types

**Output:** `false` (suppress)

**Behavior:** Invalid input → fail-safe suppression

### 9.11 Cycle disappeared and reappeared

**Input:** Cycle was in `previousCycles`, then removed, then reappears in current analysis

**Output:** `true` (warn)

**Behavior:** **Note:** This module receives only current cycle. The caller must detect disappearance by comparing current cycles with previous cycles. If caller determines cycle disappeared and reappeared, it should call this function with the cycle, and this function will return `true` because the cycle is effectively "new" in the current context.

**Alternative interpretation:** If the caller provides `previousCycles` that does not include the cycle, but the cycle was previously warned about, the caller should handle this case. This module focuses on comparing current cycle with previous cycles provided as input.

---

## 10. Integration with Other Modules

**Caller responsibilities:**

- Caller must provide `previousCycles` and `previousRootCauses` from:
  - Snapshot data (read from snapshot module)
  - Previous PR analysis results
  - PR comment history (extracted from previous comment)
- Caller must detect "cycle disappeared and reappeared" by comparing current cycles with previous cycles
- Caller must call this function for each cycle individually
- Caller must filter cycles based on return value

**Integration flow:**

1. Caller analyzes current PR → gets current cycles and root-cause edges
2. Caller reads previous state (snapshot or previous analysis) → gets previous cycles and root-cause edges
3. For each current cycle:
   - Caller calls `shouldWarnCycle` with current cycle, current root-cause, previous cycles, previous root-causes
   - If `true` → include cycle in warning
   - If `false` → exclude cycle from warning (cooldown active)
4. Caller proceeds with filtered cycles

**This module does NOT:**

- Read from snapshots directly
- Access PR API directly
- Compare current cycles with previous cycles (caller does this)
- Filter cycles automatically (caller does this based on return value)

---

## 11. Boundary Constraints

This module MUST depend on:

- Type definitions (`CanonicalCycle`, `RootCauseEdge` from `types.md` or imported modules)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- confidence modules (`confidence/computeConfidence.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- analysis modules directly (`analysis/imports.ts`, etc.)
- other safety modules (to avoid circular dependencies)

This module MUST remain a pure filtering layer.

---

## 12. Summary Checklist

`cooldown.ts` MUST:

✔ accept `CooldownContext` input structure  
✔ validate context structure deterministically  
✔ check if cycle is new (not in previous cycles)  
✔ find matching previous cycle (exact string match)  
✔ find corresponding previous root-cause edge  
✔ compare root-cause edges deterministically (all fields)  
✔ return `true` for new cycles  
✔ return `true` for cycles with changed root-cause edges  
✔ return `false` for cycles with unchanged root-cause edges  
✔ return `false` on validation failure (fail-safe suppression)  
✔ use deterministic evaluation order  
✔ prefer false positives (err on side of suppression)  
✔ treat invalid context as suppression trigger  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic comparison operations only  
✔ never log anything  
✔ never access filesystem or external services  
✔ never modify context  
✔ never maintain state across calls  
✔ never inspect cycle structure beyond canonical comparison  
✔ never access PR logic directly  

`cooldown.ts` MUST NOT:

❌ inspect cycle structure beyond canonical comparison  
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
❌ store previous cycles in module-level variables  
❌ cache results across calls  
❌ read from snapshots directly  
❌ access PR API directly  

---

## 13. Authority

This contract is authoritative over all implementation of `src/safety/cooldown.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract (Section 1: Repeated Cycle Handling)
- API Contract
- Module Boundaries
- Types (`types.md`)
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

