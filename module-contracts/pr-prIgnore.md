# Module Contract — src/pr/prIgnore.ts (V1 FINAL LOCKED)

**Layer:** pr

**Responsibility:** Deterministic detection of "ignore" replies to ArcSight's PR comments.

This module produces:

- A deterministic boolean indicating whether the developer has requested to ignore ArcSight for this PR
- An error detection flag for input validation failures

This module MUST NOT:

- modify PR comments
- delete comments
- perform PR API calls
- parse cycles or diffs
- examine file contents
- log anything
- infer meaning beyond exact matching rules
- add new signals
- interpret additional words like "mute", "shut up", "silence", etc.
- match substrings inside other words
- mutate input structures

It is a pure matching module.

---

## 1. Module Purpose & Responsibilities

**Location:** `pr/`

**Scope:** This module operates ONLY on GitHub comment thread data.

**Function:** Detects whether a developer has replied "ignore" (case-insensitive) to ArcSight's PR comment and produces a deterministic boolean indicating suppression.

**This module is the ONLY place where "ignore" logic exists.**

**Boundaries:**

- Does NOT touch PR API calls (handled by `prHandler.ts`)
- Does NOT modify or delete comments (handled by `prHandler.ts`)
- Does NOT parse cycles or diffs (handled by other modules)
- Does NOT examine file contents
- MUST be a pure function
- MUST return deterministic output

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface CommentThread {
  arcSightCommentId: string | null;     // The GitHub comment ID of ArcSight's PR comment
  comments: Array<{
    id: string;
    body: string;
    author: string;                     // GitHub username
    inReplyToId: string | null;         // Parent comment ID
  }>;
}
```

**Input assumptions:**

- `arcSightCommentId`: GitHub comment ID of ArcSight's PR comment, or `null` if ArcSight hasn't commented yet
- `comments`: Array of all PR comments
  - Each entry MUST have `id` (string, GitHub comment ID)
  - Each entry MUST have `body` (string, comment text, may contain markdown, code blocks, emojis, punctuation)
  - Each entry MUST have `author` (string, GitHub username)
  - Each entry MUST have `inReplyToId` (string | null, parent comment ID if reply, null if top-level)
  - Bodies may contain markdown, code blocks, emojis, punctuation
  - Author field may include bots and humans
  - ArcSight identifies its own comment via ID + fingerprint
  - Replies to the ArcSight comment will have `inReplyToId === arcSightCommentId`
  - Input array MUST NOT be modified

**Input validation:**

- If `comments` is not an array → return `{ shouldIgnore: false, errorDetected: true }`
- If `arcSightCommentId` is not `null` and not a string → return `{ shouldIgnore: false, errorDetected: true }`
- If any comment entry is missing required fields (`id`, `body`, `author`, `inReplyToId`) → return `{ shouldIgnore: false, errorDetected: true }`
- If string normalization fails (e.g., invalid unicode) → return `{ shouldIgnore: false, errorDetected: true }`

### 2.2 Output

```ts
interface IgnoreCheckResult {
  shouldIgnore: boolean;
  errorDetected: boolean;
}
```

**Output semantics:**

- `shouldIgnore = true` only when a human (non-bot) replies with the keyword "ignore" to ArcSight's comment
- `errorDetected = true` only when input shape is invalid / nondeterministic
- For ANY ambiguous case → return `{ shouldIgnore: false, errorDetected: false }`

**Output rules:**

- `shouldIgnore = false` is the safe default (prefer false negatives)
- `errorDetected = true` indicates input validation failure
- `errorDetected = false` with `shouldIgnore = false` indicates valid input but no ignore match
- `errorDetected = false` with `shouldIgnore = true` indicates valid input with ignore match

---

## 3. Deterministic Ignore-Matching Rules

**Ignore MUST trigger ONLY when ALL of the following are true:**

1. **The comment is authored by a non-bot account**
   - Must be deterministic: username NOT ending in `[bot]`
   - Case-sensitive check: `!author.endsWith('[bot]')`
   - Bot pattern is exact: `[bot]` at the end of the author string
   - No other bot patterns are checked (e.g., `-bot`, `-ci` are NOT considered bots)

2. **The comment is a direct reply to ArcSight's comment**
   - `inReplyToId === arcSightCommentId`
   - `arcSightCommentId` must not be `null`
   - Must be exact match (string equality)

3. **The body contains "ignore"**
   - Case-insensitive matching
   - Must match whole word using token boundaries
   - Acceptable forms: `ignore`, `Ignore`, `IGNORE`, ` please ignore `, `"ignore"`, `(ignore)`, `ignore!!!`
   - NOT acceptable: `ignored`, `ignoring`, `ignoreThis`, `re-IGNORE-me`, `signal:ignore`

**Deterministic matching algorithm:**

1. **Normalize body:**
   - Trim leading and trailing whitespace
   - Lowercase entire string
   - Replace all non-alphanumeric, non-whitespace characters with spaces
     - This includes: punctuation, emojis, markdown syntax, special characters
     - Use regex: `/[^a-z0-9\s]/gi` → replace with space
   - Replace all consecutive whitespace (spaces, tabs, newlines) with single space
   - Trim again after replacement

2. **Split on whitespace:**
   - Split normalized string on single space character
   - Filter out empty tokens (resulting from multiple spaces or edge cases)

3. **Check for "ignore" token:**
   - Iterate through all tokens
   - Check if ANY token === `"ignore"` (exact match, case-insensitive already handled)
   - If YES → `shouldIgnore = true`
   - If NO → `shouldIgnore = false`

**Punctuation replacement rules:**

- All characters that are NOT alphanumeric (a-z, A-Z, 0-9) and NOT whitespace are replaced with spaces
- This ensures `"ignore"`, `(ignore)`, `ignore!!!` all become `ignore` token after normalization
- Markdown syntax (backticks, asterisks, etc.) is treated as punctuation and replaced
- Emojis are treated as punctuation and replaced

**Whole-word matching:**

- After normalization and splitting, check for exact token match `=== "ignore"`
- This prevents substring matches like `ignored`, `ignoring`, `ignoreThis`
- Token boundaries are enforced by the split operation

**Code block handling:**

- Comment bodies are treated as plain text for matching
- Markdown code blocks are NOT stripped before matching
- Code block syntax (backticks) is replaced with spaces during normalization
- Example: `` `ignore` `` becomes `ignore` token after normalization

**Multiple "ignore" tokens:**

- If a comment contains "ignore" multiple times, it still matches
- Check all tokens, stop after first match (or continue to verify, but result is same)

---

## 4. Silent Mode Rules

`prIgnore.ts` MUST return `{ shouldIgnore: false, errorDetected: true }` when:

- `comments` is not an array
- `arcSightCommentId` is not `null` and not a string (type check fails)
- Any comment entry is missing required fields (`id`, `body`, `author`, `inReplyToId`)
- String normalization fails (e.g., invalid unicode sequences that cannot be processed)

**Silent mode behavior:**

- Return `{ shouldIgnore: false, errorDetected: true }` immediately (do not throw)
- Do not attempt partial matching
- Do not log errors
- Do not modify inputs
- Do not attempt recovery or heuristics

`prIgnore.ts` MUST return `{ shouldIgnore: false, errorDetected: false }` when:

- No matching "ignore" token found
- No replies to ArcSight's comment exist
- All replies are from bots
- All replies are to other comments (not ArcSight's)
- Inputs are valid but produce no ignore-match
- `arcSightCommentId` is `null` (ArcSight hasn't commented yet)

**Note:** Silent mode for confidence gating, safety switch, and upstream filtering is handled by `prHandler.ts` and other modules. `prIgnore.ts` only handles input validation failures.

---

## 5. False-Negative Preference Rules

`prIgnore.ts` MUST:

- Prefer missing "ignore" over a false ignore
- NEVER guess intended meaning
- NEVER match substrings
- NEVER interpret synonyms
- Ignore comments replying to other threads
- Ignore comments by bots
- Ignore ambiguous comments with multiple conflicting meanings

**If matching is ambiguous:**

- Return `{ shouldIgnore: false, errorDetected: false }`
- Do not attempt to "fix" or interpret the comment
- Prefer false negative (missed ignore) over false positive (incorrect ignore)

**Synonym handling:**

- Do NOT match: "mute", "shut up", "silence", "disable", "stop", etc.
- Only exact "ignore" token matches
- No interpretation of user intent beyond exact token matching

---

## 6. Forbidden Behaviors

`prIgnore.ts` MUST NOT:

- Touch PR state or call GitHub APIs
- Format or modify cycles
- Read files or perform Git operations
- Log or print anything (no console output, no debug statements)
- Generate multiple outputs (only one `IgnoreCheckResult` per call)
- Introduce new "commands" like `/ignore` or `/mute`
- Save or rely on state across calls
- Depend on timestamps, locales, or environment
- Use regex heuristics beyond the deterministic algorithm
- Parse markdown or code blocks (treat body as plain text for matching)
- Infer user intent beyond exact token match
- Match synonyms or variations of "ignore"
- Modify input structures
- Throw exceptions (return error flag instead)
- Use nondeterministic APIs
- Access filesystem or external services

---

## 7. Edge Cases

### 7.1 No ArcSight comment found

**Input:** `arcSightCommentId = null`

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** Cannot check replies if ArcSight comment doesn't exist. Valid input, no ignore match.

### 7.2 ArcSight comment exists, but no replies

**Input:** `arcSightCommentId` is valid string, but no comments have `inReplyToId === arcSightCommentId`

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** No replies to check, so no ignore match. Valid input.

### 7.3 User replies with "ignore" but to wrong thread

**Input:** Comment has `inReplyToId !== arcSightCommentId` (replying to different comment)

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** Not a reply to ArcSight's comment, so ignore. Valid input, no match.

### 7.4 Bot replies with "ignore"

**Input:** Comment has `author.endsWith('[bot]')` and contains "ignore" token

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** Bots cannot trigger ignore (must be human). Valid input, no match.

### 7.5 User replies "Ignore" in a code block

**Input:** Comment body contains "ignore" inside markdown code blocks (e.g., `` `ignore` ``)

**Output:** `{ shouldIgnore: true, errorDetected: false }` (if all other conditions met)

**Behavior:** Normalization treats code blocks as text (backticks replaced with spaces), so "ignore" token will be found. Valid match.

### 7.6 User replies "ignored"

**Input:** Comment contains "ignored" (not exact token "ignore")

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** Whole-word matching prevents substring match. "ignored" is a different token. Valid input, no match.

### 7.7 Multiple users reply

**Input:** Multiple comments with "ignore" replying to ArcSight's comment

**Output:** `{ shouldIgnore: true, errorDetected: false }`

**Behavior:** ANY valid "ignore" match triggers `shouldIgnore = true`. First match found is sufficient.

### 7.8 Malformed comment bodies

**Input:** Comment entries missing required fields (`id`, `body`, `author`, or `inReplyToId`)

**Output:** `{ shouldIgnore: false, errorDetected: true }`

**Behavior:** Validation failure triggers error flag. Silent mode.

### 7.9 Empty comments array

**Input:** `comments = []`

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** Valid input (empty array), no replies to check. No ignore match.

### 7.10 Invalid unicode in comment body

**Input:** Comment body contains invalid unicode sequences that cannot be normalized

**Output:** `{ shouldIgnore: false, errorDetected: true }`

**Behavior:** Normalization failure triggers error flag. Silent mode.

### 7.11 Author field format variations

**Input:** `author` field is not a string, or has unexpected format

**Output:** `{ shouldIgnore: false, errorDetected: true }`

**Behavior:** Validation failure. Author must be string for bot check to work.

### 7.12 Comment with only whitespace

**Input:** Comment body contains only whitespace characters

**Output:** `{ shouldIgnore: false, errorDetected: false }`

**Behavior:** After normalization (trim + split), no tokens remain. No match. Valid input.

---

## 8. Boundary Constraints

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR handler modules (`pr/prHandler.ts`, `pr/prSurface.ts`)
- safety modules (`safety/safetySwitch.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- confidence modules (`confidence/computeConfidence.ts`)

This module MAY depend on:

- Type definitions (if shared types are needed)

This module MUST remain pure matching.

---

## 9. Summary Checklist

`prIgnore.ts` MUST:

✔ accept `CommentThread` input structure  
✔ validate input structure deterministically  
✔ check for bot authors using `author.endsWith('[bot]')`  
✔ filter comments by `inReplyToId === arcSightCommentId`  
✔ normalize comment bodies (trim, lowercase, replace punctuation, split)  
✔ match exact "ignore" token (whole-word matching)  
✔ return `IgnoreCheckResult` with `shouldIgnore` and `errorDetected` flags  
✔ return empty string on validation failure or ambiguous cases  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic string operations only  
✔ prefer false negatives over false positives  
✔ treat ambiguity as "do not ignore"  
✔ never log anything  
✔ never access filesystem or Git  
✔ never modify input structures  
✔ never interpret synonyms or variations  
✔ never match substrings  

`prIgnore.ts` MUST NOT:

❌ touch PR state or call GitHub APIs  
❌ format or modify cycles  
❌ read files or perform Git operations  
❌ log or print anything  
❌ generate multiple outputs  
❌ introduce new commands  
❌ save or rely on state across calls  
❌ depend on timestamps, locales, or environment  
❌ use regex heuristics beyond deterministic algorithm  
❌ parse markdown or code blocks (treat as plain text)  
❌ infer user intent beyond exact token match  
❌ match synonyms or variations of "ignore"  
❌ throw exceptions  
❌ use nondeterministic APIs  

---

## 10. Authority

This contract is authoritative over all implementation of `src/pr/prIgnore.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)

Any deviation from this contract MUST be approved via ADR and SSOT update.

