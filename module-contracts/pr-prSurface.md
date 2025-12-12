# Module Contract — src/pr/prSurface.ts (V1 FINAL LOCKED)

**Layer:** pr

**Responsibility:** Deterministic PR comment body generation from cycle analysis results.

This module produces:

- A single PR comment body string (markdown format)
- Deterministic formatting of cycles and root-cause edges
- Fingerprint marker for comment detection

This module MUST NOT:

- detect cycles
- perform diffing
- compute confidence
- access filesystem or Git
- interact with PR APIs directly
- compare old vs new comments
- extract fingerprints from existing comments
- handle "ignore" replies
- modify global state
- log anything
- infer architecture

It is a pure formatting module.

---

## 1. Module Purpose & Responsibilities

**Location:** `pr/`

**Scope:** This module operates ONLY on cycle analysis results (cycles with root-cause edges).

**Function:** Generates a single PR comment body string from validated cycle data.

**Boundaries:**

- Does NOT touch PR API calls (handled by `prHandler.ts`)
- Does NOT perform comment comparison (handled by `prHandler.ts`)
- Does NOT extract fingerprints (handled by `prHandler.ts`)
- Does NOT handle "ignore" commands (handled by `prIgnore.ts`)
- MUST be a pure function
- MUST return deterministic output

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface CycleWithRootCause {
  canonicalCycle: CanonicalCycle;
  rootCause: RootCauseEdge;
}

function buildPRComment(
  cyclesWithRootCause: CycleWithRootCause[],
  fingerprint: string
): string
```

**Input assumptions:**

- `cyclesWithRootCause`: Array of cycle-root-cause pairs
  - Each entry MUST have a `canonicalCycle` (string, already canonicalized)
  - Each entry MUST have a `rootCause` (RootCauseEdge)
  - `rootCause.canonicalCycle` MUST match `canonicalCycle` in the same entry
  - Array MAY be unsorted (module MUST sort deterministically)
  - Array MAY be empty (module MUST return empty string)
  - Input array MUST NOT be modified

- `fingerprint`: Exact string `"<!-- arcsight:pr-comment:v1:cycle-only -->"`
  - MUST be passed as-is
  - MUST be inserted into the comment body
  - MUST NOT be validated or modified

**Input validation:**

- If `cyclesWithRootCause` is not an array → return `""`
- If any entry is missing `canonicalCycle` or `rootCause` → return `""`
- If any `rootCause.canonicalCycle` does not match the paired `canonicalCycle` → return `""`
- If `fingerprint` is not a string → return `""`
- If `cyclesWithRootCause.length === 0` → return `""`

### 2.2 Output

**Return type:** `string`

**Output semantics:**

- Non-empty string: Valid PR comment body (markdown format)
- Empty string `""`: Silent mode (no comment should be posted)
  - Returned when input validation fails
  - Returned when `cyclesWithRootCause` is empty
  - Returned when any structural error is detected

**Output structure:**

- MUST include the exact markdown template (see Section 5)
- MUST include the fingerprint at the end (on its own line)
- MUST format all cycles deterministically
- MUST be identical for identical inputs

---

## 3. Deterministic Behavior Rules

`prSurface.ts` MUST:

- Sort `cyclesWithRootCause` alphabetically by `canonicalCycle` before formatting
- Format cycles in deterministic order (sorted order)
- Format root-cause edges deterministically (use exact field values)
- Use deterministic string concatenation (no randomness, no timestamps)
- Produce identical output for identical inputs across all runs

`prSurface.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets
- Include timestamps or dynamic content in the comment
- Vary formatting based on environment or runtime state
- Use nondeterministic string operations

**Normalization note:** Inputs are already normalized (POSIX paths, lowercase, canonicalized). This module formats them only.

---

## 4. Sorting Rules

**Primary sort:** Alphabetical by `canonicalCycle` string (using `localeCompare` or equivalent deterministic comparison).

**Sorting requirements:**

- MUST sort `cyclesWithRootCause` array before formatting
- Sorting MUST be stable (identical inputs produce identical sort order)
- Sorting MUST be case-sensitive (canonical cycles are already lowercase)
- Sorting MUST use deterministic comparison (no locale-dependent behavior that varies across environments)

**After sorting:**

- Format cycles in sorted order
- Each cycle is formatted with its paired root-cause edge
- Order of cycles in the comment MUST match sorted order

---

## 5. Fingerprint Placement Rules

**Fingerprint string (exact):**

```
<!-- arcsight:pr-comment:v1:cycle-only -->
```

**Placement requirements:**

- MUST be inserted on its own line at the very end of the comment body
- MUST be preceded by a newline character
- MUST be the last line of the returned string
- MUST NOT be modified or validated (insert as-is)

**Format:**

```
<comment body content>

<!-- arcsight:pr-comment:v1:cycle-only -->
```

**When to include:**

- Include fingerprint ONLY when returning a non-empty string
- If returning `""`, do not include fingerprint

**Responsibility:**

- `prSurface.ts`: Renders the fingerprint into the comment body
- `prHandler.ts`: Uses the fingerprint to locate existing comments

---

## 6. Exact Markdown Template

**Complete template structure:**

```
⚠️ ArcSight Safety Check

This PR introduces new dependency cycles.

Below are the affected files and the imports that created each cycle:

<per-cycle sections>

Reply "ignore" to silence ArcSight for this PR.

<!-- arcsight:pr-comment:v1:cycle-only -->
```

**Per-cycle section format:**

For each cycle in `cyclesWithRootCause` (sorted alphabetically):

```
**Cycle:**

`<canonicalCycle>`

**New import creating the cycle:**

- From: `<rootCause.from>`
- To: `<rootCause.to>`
- Line: `<rootCause.lineNumber>` (ONLY if `rootCause.lineNumber` is provided)
- Import: `<rootCause.importLine>` (ONLY if `rootCause.importLine` is provided, as inline code with backticks)

```

**Per-cycle section rules:**

- Each cycle section MUST be separated by a blank line (empty line between sections)
- Canonical cycle MUST be formatted as inline code (backticks: `` `cycle` ``)
- Root-cause `from` and `to` paths MUST be displayed as plain text (already normalized)
- `lineNumber` MUST be displayed ONLY if `rootCause.lineNumber` is defined (not undefined, not null)
- `importLine` MUST be displayed ONLY if `rootCause.importLine` is defined (not undefined, not null, not empty string)
- `importLine` MUST be formatted as inline code (backticks: `` `import line` ``)
- If both `lineNumber` and `importLine` are missing, display only "From:" and "To:" lines

**Single cycle example:**

```
⚠️ ArcSight Safety Check

This PR introduces new dependency cycles.

Below are the affected files and the imports that created each cycle:

**Cycle:**

`src/auth/index.ts → src/session/store.ts → src/auth/index.ts`

**New import creating the cycle:**

- From: src/auth/index.ts
- To: src/session/store.ts
- Line: 42
- Import: `import { SessionStore } from './session/store'`

Reply "ignore" to silence ArcSight for this PR.

<!-- arcsight:pr-comment:v1:cycle-only -->
```

**Multiple cycles example:**

```
⚠️ ArcSight Safety Check

This PR introduces new dependency cycles.

Below are the affected files and the imports that created each cycle:

**Cycle:**

`src/auth/index.ts → src/session/store.ts → src/auth/index.ts`

**New import creating the cycle:**

- From: src/auth/index.ts
- To: src/session/store.ts
- Line: 42
- Import: `import { SessionStore } from './session/store'`

**Cycle:**

`src/utils/helpers.ts → src/core/index.ts → src/utils/helpers.ts`

**New import creating the cycle:**

- From: src/utils/helpers.ts
- To: src/core/index.ts
- Import: `import { Core } from '../core'`

Reply "ignore" to silence ArcSight for this PR.

<!-- arcsight:pr-comment:v1:cycle-only -->
```

**Template invariants:**

- Header: `⚠️ ArcSight Safety Check` (exact, no variations)
- Intro: `This PR introduces new dependency cycles.` (exact, no variations)
- Sub-intro: `Below are the affected files and the imports that created each cycle:` (exact, no variations)
- Footer: `Reply "ignore" to silence ArcSight for this PR.` (exact, no variations)
- Fingerprint: `<!-- arcsight:pr-comment:v1:cycle-only -->` (exact, no variations)

---

## 7. Silent Mode Conditions

`prSurface.ts` MUST return empty string `""` when:

- `cyclesWithRootCause` is not an array
- `cyclesWithRootCause.length === 0`
- Any entry in `cyclesWithRootCause` is missing `canonicalCycle` property
- Any entry in `cyclesWithRootCause` is missing `rootCause` property
- Any `rootCause.canonicalCycle` does not match the paired `canonicalCycle`
- `fingerprint` is not a string
- Any structural validation fails

**Silent mode behavior:**

- Return `""` immediately (do not throw)
- Do not generate partial comment content
- Do not log errors
- Do not modify inputs
- Do not attempt recovery or heuristics

**Note:** Silent mode for confidence gating, safety switch, and upstream filtering is handled by `prHandler.ts` and other modules. `prSurface.ts` only handles input validation failures.

---

## 8. False-Negative Preference Rules

`prSurface.ts` MUST:

- Only format cycles that have valid root-cause edges (non-attributable cycles are excluded upstream)
- Never guess at missing root-cause information
- Never infer cycle relationships not present in inputs
- Never "repair" malformed canonical cycles
- Prefer returning empty string over formatting incomplete information

**If validation fails:**

- Return `""` immediately
- Do not attempt partial formatting
- Do not generate comment with missing data

**If a cycle has no matching root-cause edge:**

- That cycle is excluded upstream (never reaches `prSurface.ts`)
- `prSurface.ts` never sees cycles without root-cause edges

---

## 9. Forbidden Behaviors

`prSurface.ts` MUST NOT:

- Generate multiple comment bodies (only one string per call)
- Include prohibited terms: "architecture", "governance", "fragility", "risk", "stability", blame language
- Use metaphors or abstract language
- Include analytics, metrics, scores, or trends
- Add timestamps, dates, or dynamic content
- Log anything (no console output, no debug statements)
- Access filesystem or Git
- Modify input structures
- Infer missing data or guess at formatting
- Add features outside V1 scope (heatmaps, summaries, etc.)
- Use JSON structures or metadata not defined in contract
- Alter the canonical wording template
- Compare old vs new comments (handled by `prHandler.ts`)
- Extract fingerprints from existing comments (handled by `prHandler.ts`)
- Handle "ignore" commands (handled by `prIgnore.ts`)
- Throw exceptions (return `""` instead)
- Use nondeterministic APIs
- Depend on environment variables or runtime configuration

---

## 10. Comment Diff Detection (Not prSurface.ts Responsibility)

**Clarification:**

- `prSurface.ts` MUST NOT perform any diff or comparison with existing comments
- `prSurface.ts` ONLY generates a comment body string
- `prHandler.ts` is 100% responsible for:
  - Finding an existing ArcSight comment in the PR
  - Comparing old vs new comment bodies using exact string equality
  - Deciding whether to create a new comment or update the existing one

**Conclusion:** `prSurface.ts` knows nothing about "old vs new" comments, only how to generate the current one.

---

## 11. Update-in-Place Logic (Not prSurface.ts Responsibility)

**Clarification:**

- `prSurface.ts`:
  - Does NOT know about existing comments
  - Does NOT perform any comparison logic
  - Just returns the current "ideal" comment for the given `cyclesWithRootCause`
- `prHandler.ts`:
  - Calls `buildPRComment(...)`
  - Compares the returned string with the existing ArcSight comment body (exact byte-for-byte match)
  - Only updates the PR comment if the strings differ

**Conclusion:** All update-in-place logic lives in `prHandler.ts`, not `prSurface.ts`.

---

## 12. Edge Cases

### 12.1 Empty cycles

**Input:** `cyclesWithRootCause = []`

**Output:** `""`

**Behavior:** Silent (no comment generated)

### 12.2 Single cycle with all root-cause fields

**Input:** One cycle with `lineNumber` and `importLine` both provided

**Output:** Comment with both "Line:" and "Import:" lines displayed

**Behavior:** Format all available fields

### 12.3 Single cycle with minimal root-cause fields

**Input:** One cycle with only `from` and `to` (no `lineNumber`, no `importLine`)

**Output:** Comment with only "From:" and "To:" lines (no "Line:" or "Import:" lines)

**Behavior:** Format only available fields

### 12.4 Multiple cycles, sorted order

**Input:** Multiple cycles in unsorted order

**Output:** Comment with cycles formatted in alphabetical order by `canonicalCycle`

**Behavior:** Sort before formatting

### 12.5 Malformed input structure

**Input:** `cyclesWithRootCause` is not an array, or entries missing required fields

**Output:** `""`

**Behavior:** Return empty string immediately (silent)

### 12.6 Mismatched canonicalCycle

**Input:** `rootCause.canonicalCycle !== canonicalCycle` in the same entry

**Output:** `""`

**Behavior:** Return empty string immediately (silent)

### 12.7 Very long cycle paths

**Input:** Cycles with very long file paths

**Output:** Formatted as-is (no truncation)

**Behavior:** Display full canonical cycle string

### 12.8 Very long import lines

**Input:** `rootCause.importLine` is very long

**Output:** Formatted as inline code (backticks), no truncation

**Behavior:** Display full import line

### 12.9 Special characters in paths

**Input:** Paths or import lines containing special markdown characters

**Output:** Properly escaped or formatted (inline code prevents markdown interpretation)

**Behavior:** Use inline code formatting to prevent markdown issues

### 12.10 Empty fingerprint

**Input:** `fingerprint = ""` (empty string)

**Output:** `""` (validation failure)

**Behavior:** Return empty string (fingerprint must be the exact required string)

---

## 13. Boundary Constraints

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR handler modules (`pr/prHandler.ts`, `pr/prIgnore.ts`)
- safety modules (`safety/safetySwitch.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- confidence modules (`confidence/computeConfidence.ts`)

This module MAY depend on:

- `types.md` for type definitions (`CanonicalCycle`, `RootCauseEdge`)

This module MUST remain pure formatting.

---

## 14. Summary Checklist

`prSurface.ts` MUST:

✔ accept `CycleWithRootCause[]` and `fingerprint` string  
✔ validate input structure deterministically  
✔ sort cycles alphabetically by `canonicalCycle`  
✔ format cycles using exact markdown template  
✔ include fingerprint at end of comment body  
✔ return empty string on validation failure or empty input  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic string operations only  
✔ format root-cause edges with available fields only  
✔ separate per-cycle sections with blank lines  
✔ use inline code formatting for canonical cycles and import lines  
✔ never include prohibited terms  
✔ never log anything  
✔ never access filesystem or Git  
✔ never modify input structures  
✔ never compare old vs new comments  
✔ never extract fingerprints from existing comments  

`prSurface.ts` MUST NOT:

❌ generate multiple comment bodies  
❌ include timestamps or dynamic content  
❌ use nondeterministic APIs  
❌ infer missing data  
❌ add features outside V1 scope  
❌ perform comment comparison  
❌ handle "ignore" commands  
❌ throw exceptions  
❌ log errors or debug information  

---

## 15. Authority

This contract is authoritative over all implementation of `src/pr/prSurface.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)

Any deviation from this contract MUST be approved via ADR and SSOT update.

