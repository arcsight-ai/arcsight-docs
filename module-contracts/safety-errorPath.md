# Module Contract — src/safety/errorPath.ts (V1 FINAL LOCKED)

**Layer:** safety

**Responsibility:** Deterministic error handling utilities that catch exceptions from module calls and convert them to silent mode, ensuring all errors result in silent behavior without exceptions propagating.

This module produces:

- Error handling wrappers that catch exceptions
- Deterministic error flags indicating silent mode should be activated
- Safe function execution that never throws exceptions

This module MUST NOT:

- inspect cycle structure
- access PR logic directly
- modify outputs beyond error handling
- detect cycles or perform diffing
- compute confidence
- log anything
- infer architecture
- add new signals
- maintain state across calls

It is a pure error handling utility module.

---

## 1. Module Purpose & Responsibilities

**Location:** `safety/`

**Scope:** This module operates on functions and error conditions to provide deterministic error handling.

**Function:** Provides error handling utilities that catch exceptions, convert them to silent mode indicators, and ensure no exceptions propagate to callers.

**This module ensures all errors result in silent mode without exceptions.**

**Boundaries:**

- Does NOT inspect cycle structure
- Does NOT access PR logic directly
- Does NOT modify outputs beyond error handling (read-only error detection)
- Does NOT detect cycles or perform diffing
- Does NOT compute confidence (delegated to confidence module)
- MUST catch all exceptions
- MUST convert exceptions to silent mode indicators
- MUST be a pure function (no state)

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface ErrorPathContext {
  error: Error | null;
  errorDetected?: boolean;
  result?: unknown;
}

// OR for function wrapping:
type SafeFunction<T> = () => T;
```

**Input assumptions:**

- `error`: Optional Error object or null
  - May be `null` (no error)
  - May be an Error instance (error occurred)
- `errorDetected`: Optional boolean flag indicating error state
  - May be `undefined` (not provided)
  - May be `true` (error detected)
  - May be `false` (no error)
- `result`: Optional result from a function call
  - May be `undefined` (not provided)
  - May be any value (function result)

**Input validation:**

- If `context` is missing or `null`/`undefined` → return `shouldSilence: true` (fail-safe)
- If `error` is provided and not null → return `shouldSilence: true`
- If `errorDetected === true` → return `shouldSilence: true`
- If all indicators are absent or negative → return `shouldSilence: false`

### 2.2 Output

**Function signature (Option 1 - Error evaluation):**

```ts
function shouldSilenceOnError(context: ErrorPathContext): boolean
```

**Function signature (Option 2 - Function wrapper):**

```ts
function executeSafely<T>(fn: SafeFunction<T>): T | null
```

**Return type:** `boolean` (for error evaluation) OR `T | null` (for function wrapper)

**Output semantics:**

- `shouldSilenceOnError`: Returns `true` if silent mode should be activated, `false` otherwise
- `executeSafely`: Returns function result on success, `null` on error (triggers silent mode)

**Output rules:**

- `true` is the safe default (prefer silence over incorrect output)
- `false` only when no errors are detected
- Must be deterministic (same context → same result)
- Never throws exceptions

---

## 3. Error Handling Rules

### 3.1 Error Detection

**Silent mode MUST be activated (`true`) when:**

1. **Exception thrown:**
   - `error` is provided and not `null`
   - Any exception is caught during function execution

2. **Error flag set:**
   - `errorDetected === true`
   - Any module returns `errorDetected: true`

3. **Invalid context:**
   - Context is missing, null, or undefined
   - Required fields are missing or invalid types

**Silent mode MUST NOT be activated (`false`) when:**

- `error === null` AND `errorDetected === false` (or undefined)
- No exceptions thrown
- All error indicators are negative
- Context is valid and complete

### 3.2 Function Wrapping

**When wrapping functions:**

- Catch ALL exceptions (try-catch around function execution)
- Convert exceptions to `null` return value
- Never re-throw exceptions
- Never log exceptions
- Return `null` immediately on any exception (fail-fast)

**Function execution rules:**

- Execute function synchronously (if synchronous function)
- Return function result on success
- Return `null` on any exception
- Do not attempt partial execution
- Do not retry on failure

---

## 4. Deterministic Behavior Rules

`errorPath.ts` MUST:

- Produce identical output for identical input context
- Use deterministic error detection logic
- Apply error handling rules deterministically (fixed evaluation order)
- Never use timestamps or dynamic content in decision logic
- Never depend on environment or runtime state
- Never maintain state across calls (pure function)
- Never throw exceptions (catch all, return flags)

`errorPath.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Store error state in module-level variables
- Cache results across calls
- Re-throw exceptions
- Log errors or exceptions

**Evaluation order (deterministic):**

1. Input validation (context structure)
2. Check for Error object (if provided)
3. Check for errorDetected flag (if provided)
4. Return boolean result

This order MUST be fixed to ensure deterministic evaluation.

---

## 5. Silent Mode Rules

`errorPath.ts` MUST return `true` (silent mode) when:

- ANY error is detected
- ANY exception is thrown
- Context is invalid or malformed
- Error flag is set
- Any uncertainty is present

**Silent mode behavior:**

- Return `true` immediately on any error (short-circuit evaluation)
- Do not attempt partial evaluation
- Do not log errors
- Do not modify context
- Prefer false positives (err on side of silence)

`errorPath.ts` MUST return `false` (proceed) when:

- No errors detected
- No exceptions thrown
- All error indicators are negative
- Context is valid and complete

---

## 6. False-Positive Preference Rules

`errorPath.ts` MUST:

- Prefer activating silent mode (return `true`) over allowing potentially incorrect output
- NEVER allow output when uncertain
- Treat ANY error as a trigger
- Activate on partial failures or inconsistencies
- Return `true` when context is invalid or incomplete

**If evaluation is ambiguous:**

- Return `true` (silent mode)
- Do not attempt to "fix" or interpret context
- Prefer false positive (incorrect silence) over false negative (incorrect output)

**Fail-safe principle:**

- When in doubt, silence
- Invalid context → silence
- Missing data → silence (if error indicators are missing, assume error)
- Ambiguous indicators → silence

---

## 7. Forbidden Behaviors

`errorPath.ts` MUST NOT:

- Inspect cycle structure
- Access PR logic directly
- Modify outputs beyond error handling
- Detect cycles or perform diffing
- Compute confidence (delegated to confidence module)
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps or dynamic content
- Depend on environment variables
- Throw exceptions (catch all, return flags)
- Use nondeterministic APIs
- Access filesystem or external services
- Maintain state across calls (must be pure function)
- Store error state in module-level variables
- Cache results across calls
- Re-throw exceptions
- Log errors or exceptions
- Attempt to recover from errors
- Retry failed operations

---

## 8. Edge Cases

### 8.1 No error

**Input:** `error = null`, `errorDetected = false` (or undefined)

**Output:** `false` (proceed)

**Behavior:** No errors → proceed normally

### 8.2 Exception thrown

**Input:** `error = Error("...")` OR exception caught during function execution

**Output:** `true` (silent mode)

**Behavior:** Exception detected → silent mode

### 8.3 Error flag set

**Input:** `errorDetected = true`

**Output:** `true` (silent mode)

**Behavior:** Error flag → silent mode

### 8.4 Invalid context

**Input:** `context = null` or `undefined`

**Output:** `true` (silent mode)

**Behavior:** Invalid context → fail-safe silence

### 8.5 Function returns null on error

**Input:** Function throws exception

**Output:** `null` (for function wrapper)

**Behavior:** Exception caught → return null (caller interprets as silent mode)

### 8.6 Function succeeds

**Input:** Function executes successfully

**Output:** Function result (for function wrapper)

**Behavior:** No exception → return result

### 8.7 Multiple error indicators

**Input:** Both `error` and `errorDetected` indicate errors

**Output:** `true` (silent mode)

**Behavior:** Any error indicator → silent mode

### 8.8 Missing error indicators

**Input:** All error indicators are `undefined` or missing

**Output:** `true` (silent mode, fail-safe)

**Behavior:** Missing indicators → assume error (fail-safe)

### 8.9 Error object but errorDetected false

**Input:** `error = Error("...")`, `errorDetected = false`

**Output:** `true` (silent mode)

**Behavior:** Error object takes precedence → silent mode

### 8.10 No error object but errorDetected true

**Input:** `error = null`, `errorDetected = true`

**Output:** `true` (silent mode)

**Behavior:** Error flag takes precedence → silent mode

---

## 9. Integration with Other Modules

**Caller responsibilities:**

- Caller may wrap function calls with `executeSafely` to catch exceptions
- Caller may check `shouldSilenceOnError` after module calls
- Caller must interpret `null` return from `executeSafely` as silent mode trigger
- Caller must handle error flags from module results

**Integration flow:**

1. Caller calls analysis/diff/PR modules
2. If using `executeSafely` wrapper:
   - Wraps function call
   - Catches exceptions
   - Returns result or `null`
3. If checking error flags:
   - Calls `shouldSilenceOnError` with module result
   - Checks return value
   - Activates silent mode if `true`
4. Caller proceeds with filtered/validated results

**This module does NOT:**

- Read from snapshots directly
- Access PR API directly
- Modify data structures
- Fix errors automatically

---

## 10. Boundary Constraints

This module MUST depend on:

- Type definitions (if shared types are needed)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- confidence modules (`confidence/computeConfidence.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- analysis modules directly (`analysis/imports.ts`, etc.)
- other safety modules (to avoid circular dependencies)

This module MUST remain a pure error handling utility layer.

---

## 11. Summary Checklist

`errorPath.ts` MUST:

✔ accept `ErrorPathContext` input structure OR function to wrap  
✔ validate context structure deterministically  
✔ check for Error objects  
✔ check for errorDetected flags  
✔ catch all exceptions (if wrapping functions)  
✔ return `true` on ANY error  
✔ return `false` only when no errors detected  
✔ return `null` on exceptions (for function wrapper)  
✔ use deterministic evaluation order  
✔ prefer false positives (err on side of silence)  
✔ treat invalid context as error trigger  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic comparison operations only  
✔ never log anything  
✔ never access filesystem or external services  
✔ never modify context beyond error handling  
✔ never maintain state across calls  
✔ never re-throw exceptions  

`errorPath.ts` MUST NOT:

❌ inspect cycle structure  
❌ access PR logic directly  
❌ modify outputs beyond error handling  
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
❌ store error state in module-level variables  
❌ cache results across calls  
❌ re-throw exceptions  
❌ attempt to recover from errors  
❌ retry failed operations  

---

## 12. Authority

This contract is authoritative over all implementation of `src/safety/errorPath.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)
- Determinism Requirements
- ADRs (especially ADR 0003: Silent on Error)

Any deviation from this contract MUST be approved via ADR and SSOT update.

