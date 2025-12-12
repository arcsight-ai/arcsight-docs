# Module Contract — src/diff/cycleDiff.ts (V1 FINAL LOCKED)

**Layer:** diff

**Responsibility:** Deterministic set-based diffing of canonical cycles.

This module produces:

- A deterministic diff of canonical cycles (new vs removed)
- Error detection flag for nondeterminism

This module MUST NOT:

- re-canonicalize cycles
- attempt to "repair" malformed cycles
- infer architecture
- look at PR diffs or changed files
- compute root-cause edges
- call `imports.ts`, `cycles.ts`, or any other module
- read filesystem or Git data
- log anything
- introduce new signals
- mutate inputs
- return partial or speculative output

It is a pure diffing module.

---

## 1. Module Purpose & Responsibilities

**Location:** `diff/`

**Scope:** This module operates ONLY on canonical cycles.

**Function:** Performs ONLY a set-diff between two arrays of canonical cycle strings.

**Boundaries:**

- Does NOT touch PR logic
- Does NOT perform graph analysis
- Does NOT perform canonicalization
- MUST be a pure function

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
base: CanonicalCycle[]
head: CanonicalCycle[]
```

**Input assumptions:**

- Inputs are arrays of canonical cycle strings
- Cycles are already canonicalized (as defined in `types.md`)
- Cycles are already normalized (POSIX, lowercase)
- Cycles MAY be unsorted (module MUST sort deterministically)
- Cycles MAY contain duplicates (module MUST treat them as sets)
- Input strings MUST NOT be modified

### 2.2 Output

```ts
interface CycleDiffResult {
  newCycles: CanonicalCycle[];
  removedCycles: CanonicalCycle[];
  errorDetected: boolean;
}
```

**newCycles:**

- Cycles present in `head` but not in `base`
- MUST be sorted alphabetically
- MUST be deduplicated (set semantics)
- MUST contain only canonical cycle strings

**removedCycles:**

- Cycles present in `base` but not in `head`
- MUST be sorted alphabetically
- MUST be deduplicated (set semantics)
- MUST contain only canonical cycle strings

**errorDetected:**

- `true` ONLY when `cycleDiff.ts` cannot guarantee deterministic correctness
- `false` when operating normally, INCLUDING "no cycles found" and "empty inputs"

---

## 3. Deterministic Behavior Rules

`cycleDiff.ts` MUST:

- Copy and sort `base`/`head` canonically before diffing
- Treat both arrays as mathematical sets
- Compare cycles using exact string equality (byte-for-byte)
- Produce output identical across runs
- Ensure:
  - `newCycles` sorted alphabetically
  - `removedCycles` sorted alphabetically
  - No duplicates in either output
- Be stable under input order permutations
- NEVER modify input arrays

**Forbidden:**

- Trimming cycle strings
- Re-canonicalizing cycles
- Case-changing
- Path normalization (already done upstream)

---

## 4. Set Semantics & Equality Rules

**Equality definition:**

Two cycles are equal if and only if:

```ts
base[i] === head[j]  // strict byte-for-byte equality
```

**Rules:**

- Strict byte-for-byte equality
- No transformations allowed
- No normalization attempts
- No case-insensitive comparison

**Malformed input strings:**

- MUST trigger silent mode (`errorDetected = true`)
- MUST NOT be "fixed" or repaired
- MUST NOT be normalized or transformed

---

## 5. Silent Mode Conditions (MANDATORY)

`cycleDiff.ts` MUST return:

```ts
{
  newCycles: [],
  removedCycles: [],
  errorDetected: true
}
```

ONLY if:

### 5.1 Inputs are not arrays

- `base` is not an array
- `head` is not an array

### 5.2 Inputs contain non-strings

- Any element in `base` is not a string
- Any element in `head` is not a string
- Any element is `null` or `undefined`

### 5.3 Duplicate canonical cycles violate equality invariants

- Internal set operations produce inconsistent results
- Deduplication fails deterministically

### 5.4 Internal invariants break

- Sorting throws an exception
- Set operations produce nondeterministic results

### 5.5 CanonicalCycle values violate format

- Cycle strings do not match the canonicalization format defined in `types.md`
- Format validation fails

**Corrected rules (NOT silent mode):**

`cycleDiff.ts` MUST NOT enter silent mode for:

- Empty input arrays (`base = []`, `head = []`)
- No cycles detected (identical sets)
- Large but valid sets
- Empty graphs

**Hard rule:**

`cycleDiff.ts` MUST NOT set `errorDetected = true` solely because:

- No cycles were found
- Input arrays are empty
- Sets are identical

"Zero cycles" and "empty inputs" are valid, non-error outcomes.

Empty = Fine.  
Quiet = Fine.  
`errorDetected` must remain `false`.

---

## 6. False-Negative Preference Rules

`cycleDiff.ts` MUST:

- Prefer missing a cycle (false negative) over introducing a wrong diff (false positive)
- If equality of two cycles is ambiguous:
  - MUST treat them as unequal
  - MUST NOT attempt normalization
  - MUST NOT guess

Producing no diff is ALWAYS safer than producing a wrong diff.

---

## 7. Internal Data Constraints

`cycleDiff.ts` MUST:

- Use ONLY local variables
- Avoid global state, static caches, memoization
- Avoid mutation of `base`/`head` arrays
- Avoid side effects
- Perform deterministic array sorting using ASCII order

**Forbidden:**

- Maps keyed by objects
- Nondeterministic iteration
- Sorting based on locale
- Global or module-level mutable state

---

## 8. Edge Cases (Required Behaviors)

### 8.1 Empty inputs

`base = []`, `head = []`

- `newCycles = []`
- `removedCycles = []`
- `errorDetected = false`

### 8.2 New cycle in head

`base = []`, `head = ['A → B → A']`

- `newCycles = ['A → B → A']`
- `removedCycles = []`
- `errorDetected = false`

### 8.3 Removed cycle from base

`base = ['A → B → A']`, `head = []`

- `newCycles = []`
- `removedCycles = ['A → B → A']`
- `errorDetected = false`

### 8.4 Unsorted inputs

`base` unsorted, `head` unsorted

- Diff result MUST be identical to sorted input
- Order invariance MUST be guaranteed

### 8.5 Duplicates in inputs

Duplicates in `base`/`head`

- MUST be ignored (set semantics)
- Output MUST contain no duplicates

### 8.6 Identical sets

`base = ['A → B → A']`, `head = ['A → B → A']`

- `newCycles = []`
- `removedCycles = []`
- `errorDetected = false`

### 8.7 Malformed canonical strings

Malformed canonical cycle strings in input

- MUST trigger silent mode
- `errorDetected = true`
- Empty outputs

### 8.8 Large valid sets

Thousands of cycles in `base`/`head`

- MUST remain deterministic
- MUST complete within performance constraints
- `errorDetected = false` if inputs are valid

---

## 9. Boundary Constraints

This module MUST NOT depend on:

- filesystem modules (`fs`)
- import parsing modules (`imports.ts`)
- cycle detection modules (`cycles.ts`)
- PR modules (`prHandler.ts`, `prSurface.ts`, `prIgnore.ts`)
- safety modules (`safetySwitch.ts`, `cooldown.ts`, `invariants.ts`, `errorPath.ts`)
- snapshot modules (`snapshotWriter.ts`)
- confidence module (`computeConfidence.ts`)

It MAY be used by:

- `rootCause.ts`
- `prHandler.ts`

It MUST remain a pure diffing module with no PR/build/runtime semantics.

---

## 10. Summary

`cycleDiff.ts` MUST:

✔ be deterministic  
✔ be stable under input order permutations  
✔ treat inputs as mathematical sets  
✔ sort outputs alphabetically  
✔ dedupe outputs  
✔ set `errorDetected = false` for all valid cases (even empty)  
✔ set `errorDetected = true` ONLY for malformed or nondeterministic cases  
✔ use exact string equality for comparison  
✔ copy inputs before processing (never mutate)  

`cycleDiff.ts` MUST NOT:

❌ re-canonicalize cycles  
❌ modify input arrays  
❌ infer architecture  
❌ use PR semantics  
❌ log anything  
❌ add new signals  
❌ attempt to repair malformed input  
❌ normalize or transform cycle strings  

This module is the foundation upon which:

- root-cause detection
- PR cycle filtering
- cycle change tracking

all depend.

