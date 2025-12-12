# Module Contract — src/pr/prHandler.ts (V1 FINAL LOCKED)

**Layer:** pr

**Responsibility:** Orchestrate PR comment lifecycle: coordinate cycle analysis, check ignore status, generate comment content, manage PR API interactions, and enforce PR Update Invariance.

This module produces:

- PR comment creation, updates, or deletions via GitHub API
- Deterministic comment lifecycle management
- PR Update Invariance enforcement

This module MUST NOT:

- detect cycles
- perform diffing
- compute confidence
- format comment body
- detect ignore replies
- parse cycles or diffs
- modify global state (except PR comment state via APIs)
- log anything
- infer architecture

It is an orchestration module that coordinates other modules and manages PR API interactions.

---

## 1. Module Purpose & Responsibilities

**Location:** `pr/`

**Scope:** This module operates on PR events and coordinates the full PR comment lifecycle.

**Function:** Orchestrates PR comment management by calling analysis modules, formatting modules, and PR APIs to create/update/delete PR comments based on cycle analysis results.

**This module is the ONLY place where PR API calls occur.**

**Boundaries:**

- Does NOT detect cycles (delegated to `analyzeCyclesForPR`)
- Does NOT perform diffing (delegated to `analyzeCyclesForPR`)
- Does NOT compute confidence (delegated to `analyzeCyclesForPR`)
- Does NOT format comment body (delegated to `prSurface.ts`)
- Does NOT detect ignore replies (delegated to `prIgnore.ts`)
- Does NOT parse cycles or diffs
- MUST coordinate other modules
- MUST manage PR API interactions
- MUST enforce PR Update Invariance

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface PullRequestEvent {
  repository: {
    owner: string;
    name: string;
    fullName: string;  // "owner/name"
  };
  pullRequest: {
    number: number;
    baseSha: string;
    headSha: string;
    changedFiles: string[];  // POSIX-normalized, lowercased file paths
  };
  repoPath: string;  // Local filesystem path to repository
}
```

**Input assumptions:**

- `repository`: GitHub repository information
  - `owner`: Repository owner (username or organization)
  - `name`: Repository name
  - `fullName`: Full repository name in "owner/name" format
- `pullRequest`: PR metadata
  - `number`: PR number (integer)
  - `baseSha`: Base commit SHA (string)
  - `headSha`: Head commit SHA (string)
  - `changedFiles`: Array of file paths changed in PR (already normalized)
- `repoPath`: Local filesystem path to repository (string)
  - Repository MUST be checked out at base and head commits
  - Path MUST be valid and accessible

**Input validation:**

- If any required field is missing or invalid type → enter silent mode (no comment action)
- If `repoPath` is invalid or inaccessible → enter silent mode
- If PR metadata is malformed → enter silent mode

### 2.2 Output

**Function signature:**

```ts
function handlePullRequest(event: PullRequestEvent): Promise<void>
```

**Return type:** `Promise<void>`

**Output semantics:**

- Resolves successfully: PR comment lifecycle completed (created/updated/deleted as needed)
- Rejects or fails: Silent mode (no comment action, no errors thrown to caller)
- No return value (side effects via PR API)

**Side effects:**

- Creates, updates, or deletes PR comment via GitHub API
- No other side effects
- No logging
- No state mutation beyond PR comment

---

## 3. PR Comment Lifecycle Management

### 3.1 Cycle Analysis

**Step 1: Call `analyzeCyclesForPR`**

- Call: `analyzeCyclesForPR(baseSha, headSha, changedFiles, repoPath)`
- Receive: `PRCycleAnalysis` with `relevantCycles`, `rootCauses`, `confidence`
- If call fails or returns error → enter silent mode

**Step 2: Confidence Gating**

- Check: `confidence >= 0.8`
- If `confidence < 0.8` → enter silent mode (do not create/update comment)
- If confidence check fails → enter silent mode

**Step 3: Safety Switch Check**

- Check safety switch status (if available from analysis result or separate call)
- If safety switch active → enter silent mode
- If safety switch check fails → enter silent mode

### 3.2 Ignore Status Check

**Step 4: Fetch PR Comments**

- Fetch all PR comments via GitHub API for the PR
- If API call fails → enter silent mode
- Build `CommentThread` structure:
  ```ts
  {
    arcSightCommentId: string | null,  // Found via fingerprint search
    comments: Array<{ id, body, author, inReplyToId }>
  }
  ```

**Step 5: Find ArcSight Comment**

- Search all comment bodies for fingerprint: `<!-- arcsight:pr-comment:v1:cycle-only -->`
- Extract comment ID and body if found
- Set `arcSightCommentId` to comment ID if found, `null` otherwise

**Step 6: Check Ignore Request**

- Call: `checkIgnoreRequest(commentThread)` from `prIgnore.ts`
- Receive: `IgnoreCheckResult` with `shouldIgnore` and `errorDetected`
- If `shouldIgnore === true` → enter silent mode (do not create/update/delete)
- If `errorDetected === true` → enter silent mode (validation failure)

### 3.3 Comment Content Generation

**Step 7: Pair Cycles with Root-Cause Edges**

- Build `CycleWithRootCause[]` array:
  - For each cycle in `relevantCycles`, find matching root-cause edge in `rootCauses`
  - Match by: `rootCause.canonicalCycle === cycle`
  - If cycle has no matching root-cause edge → exclude from pairing
  - Sort pairs alphabetically by `canonicalCycle`
- Result: Array of cycles with their root-cause edges (cycles without root-cause excluded)

**Step 8: Generate Comment Body**

- Call: `buildPRComment(cyclesWithRootCause, fingerprint)` from `prSurface.ts`
- Fingerprint: `"<!-- arcsight:pr-comment:v1:cycle-only -->"`
- Receive: Comment body string (markdown format)
- If call returns empty string → proceed to deletion logic
- If call fails → enter silent mode

### 3.4 Comment Comparison & Update Decision

**Step 9: Compare Old vs New Comment**

- If existing ArcSight comment found:
  - Compare old comment body with new comment body using exact string equality
  - If identical (byte-for-byte match) → do not update (PR Update Invariance)
  - If different → proceed to update
- If no existing comment:
  - Proceed to create if new body is non-empty
  - Do not create if new body is empty

**Step 10: Execute Comment Action**

- **Delete:** If new comment body is empty string and existing comment exists
  - Delete existing comment via GitHub API
  - If deletion fails → enter silent mode
- **Create:** If new comment body is non-empty and no existing comment
  - Create new comment via GitHub API
  - If creation fails → enter silent mode
- **Update:** If new comment body is non-empty and different from existing
  - Update existing comment via GitHub API
  - If update fails → enter silent mode
- **No-op:** If new comment body is identical to existing
  - Do nothing (PR Update Invariance)

---

## 4. PR Update Invariance Enforcement

**Rule (from Safety Wedge Contract Section 8):** ArcSight MUST NOT update its PR comment unless:
- The set of relevant cycles has changed, OR
- The associated root-cause edges have changed

**Implementation:**

- Compare old comment body with new comment body using exact string equality (byte-for-byte)
- Only update if strings differ
- No updates for:
  - Identical content
  - Cosmetic changes (if strings are identical)
  - Whitespace-only diffs (if strings are identical)
  - Re-runs with same cycle analysis results

**Edge cases:**

- Same cycles, same root-cause edges → no update (strings identical)
- Same cycles, different root-cause edges → update (strings differ)
- Different cycles, same root-cause edges → update (strings differ)
- Cycles resolved (empty) → delete comment (new body is empty string)

---

## 5. Fingerprint Extraction

**Responsibility:**

- Extract fingerprint from existing PR comments
- Search for exact string: `<!-- arcsight:pr-comment:v1:cycle-only -->`
- Identify which comment contains the fingerprint
- Extract comment ID and body for comparison

**Algorithm:**

1. Fetch all PR comments via GitHub API
2. For each comment:
   - Search comment body for fingerprint string (exact match)
   - If found: extract comment ID and body, return immediately
3. If no match found: return `null` for `arcSightCommentId`

**Fingerprint string (exact):**

```
<!-- arcsight:pr-comment:v1:cycle-only -->
```

**Placement:**

- Fingerprint is inserted by `prSurface.ts` at the end of comment body
- Search for exact string match (case-sensitive, whitespace-sensitive)

---

## 6. Ignore Command Handling

**Responsibility:**

- Check if developer has replied "ignore" to ArcSight's comment
- Call `checkIgnoreRequest` from `prIgnore.ts`
- If `shouldIgnore === true`, suppress all comment actions (silent mode)

**Behavior:**

- Ignore check happens after fetching comments but before comment generation
- If ignored, do not create/update/delete any comments
- If ignored, do not perform comment generation (early exit for efficiency)
- If ignored, still perform cycle analysis (for potential future use, but do not comment)

**Integration:**

- Build `CommentThread` from fetched PR comments
- Pass to `checkIgnoreRequest` function
- Respect `shouldIgnore` flag in decision logic

---

## 7. Safety Switch & Confidence Gating

**Confidence Gating:**

- Receive `confidence` from `analyzeCyclesForPR` result
- Check: `confidence >= 0.8`
- If `confidence < 0.8` → enter silent mode (do not create/update comment)
- If confidence is undefined or invalid → enter silent mode

**Safety Switch:**

- Check safety switch status (if available from analysis result or separate module call)
- If safety switch active → enter silent mode
- If safety switch check fails → enter silent mode

**Integration:**

- Confidence gating happens immediately after cycle analysis
- Safety switch check happens after confidence gating
- Both must pass for comment generation to proceed

---

## 8. Error Handling & Silent Mode

**PR API errors:**

- If GitHub API call fails (fetch comments, create comment, update comment, delete comment):
  - Enter silent mode (no comment action)
  - Do not throw exceptions
  - Do not log errors
  - Fail silently
  - Return resolved Promise (do not reject)

**Analysis errors:**

- If `analyzeCyclesForPR` fails or throws:
  - Enter silent mode
  - Do not create/update comment
  - Do not throw exceptions
  - Return resolved Promise

**Validation errors:**

- If inputs are malformed:
  - Enter silent mode
  - Do not attempt partial operations
  - Return resolved Promise

**Module call errors:**

- If `buildPRComment` fails or returns error:
  - Enter silent mode
  - Do not create/update comment
  - Return resolved Promise

- If `checkIgnoreRequest` fails:
  - Enter silent mode (treat as error)
  - Do not create/update comment
  - Return resolved Promise

**Silent mode behavior:**

- No comment actions (create/update/delete)
- No exceptions thrown
- No logging
- No partial operations
- Return resolved Promise (successful completion, even if silent)

---

## 9. Deterministic Behavior Rules

`prHandler.ts` MUST:

- Produce deterministic comment actions (same PR state → same comment action)
- Use exact string comparison for comment diff detection
- Sort cycles deterministically before pairing with root-cause edges
- Apply PR Update Invariance deterministically
- Handle errors deterministically (always silent mode on failure)

`prHandler.ts` MUST NOT:

- Use timestamps or dynamic content in decision logic
- Depend on comment order or iteration order (sort before processing)
- Use nondeterministic APIs beyond GitHub PR API calls (API calls are external, but decision logic is deterministic)
- Vary behavior based on environment or runtime state

**Note:** PR API calls themselves are external and may have timing variations, but the decision logic (whether to create/update/delete) must be deterministic based on PR state and analysis results.

---

## 10. Cycle-RootCause Pairing Logic

**Pairing algorithm:**

1. For each cycle in `relevantCycles`:
   - Find matching root-cause edge in `rootCauses` where `rootCause.canonicalCycle === cycle`
   - If match found: create `CycleWithRootCause` pair
   - If no match found: exclude cycle from pairing (cycles without root-cause are not included in comment)

2. Sort pairs alphabetically by `canonicalCycle`:
   - Use `localeCompare` for deterministic sorting
   - Sort before passing to `buildPRComment`

3. Result: `CycleWithRootCause[]` array
   - Contains only cycles that have matching root-cause edges
   - Sorted alphabetically by `canonicalCycle`

**Edge cases:**

- No cycles → empty array → `buildPRComment` returns empty string → delete comment
- Cycles without root-cause → excluded from pairing → not included in comment
- Multiple root-cause edges for same cycle → use first match (should not happen, but handle gracefully)

---

## 11. Forbidden Behaviors

`prHandler.ts` MUST NOT:

- Detect cycles (delegated to `analyzeCyclesForPR`)
- Perform diffing (delegated to `analyzeCyclesForPR`)
- Compute confidence (delegated to `analyzeCyclesForPR`)
- Format comment body (delegated to `prSurface.ts`)
- Detect ignore replies (delegated to `prIgnore.ts`)
- Parse cycles or diffs
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Modify input structures (except via PR API)
- Throw exceptions (fail silently, return resolved Promise)
- Use heuristics beyond exact matching
- Retry failed API calls (fail silently on first failure)
- Perform partial operations (all-or-nothing)

---

## 12. Edge Cases

### 12.1 No existing ArcSight comment, cycles found

**Input:** No existing comment, `relevantCycles` non-empty

**Output:** Create new comment with cycle information

**Behavior:** Generate comment body, create via API

### 12.2 No existing ArcSight comment, no cycles

**Input:** No existing comment, `relevantCycles` empty

**Output:** Do not create comment

**Behavior:** No comment action (silent, no-op)

### 12.3 Existing comment, cycles resolved

**Input:** Existing comment, `relevantCycles` empty

**Output:** Delete existing comment

**Behavior:** `buildPRComment` returns empty string, delete via API

### 12.4 Existing comment, content unchanged

**Input:** Existing comment, new comment body identical to old

**Output:** Do not update comment

**Behavior:** PR Update Invariance, no API call

### 12.5 Existing comment, content changed

**Input:** Existing comment, new comment body different from old

**Output:** Update existing comment

**Behavior:** Update via API

### 12.6 Developer replied "ignore"

**Input:** `checkIgnoreRequest` returns `shouldIgnore: true`

**Output:** Do not create/update/delete comment

**Behavior:** Silent mode, no comment actions

### 12.7 Confidence < 0.8

**Input:** `confidence < 0.8`

**Output:** Do not create/update comment

**Behavior:** Silent mode, no comment actions

### 12.8 Safety switch active

**Input:** Safety switch indicates silence required

**Output:** Do not create/update comment

**Behavior:** Silent mode, no comment actions

### 12.9 PR API failure

**Input:** GitHub API call fails (network error, auth error, etc.)

**Output:** Silent mode, no comment action

**Behavior:** Fail silently, return resolved Promise

### 12.10 Analysis failure

**Input:** `analyzeCyclesForPR` fails or throws

**Output:** Silent mode, no comment action

**Behavior:** Fail silently, return resolved Promise

### 12.11 Multiple ArcSight comments (should not happen)

**Input:** Multiple comments contain fingerprint

**Output:** Update first found, or handle gracefully

**Behavior:** Use first match, or enter silent mode if ambiguous

### 12.12 Cycles without root-cause edges

**Input:** `relevantCycles` contains cycles, but some have no matching `rootCause`

**Output:** Only cycles with root-cause edges included in comment

**Behavior:** Pairing logic excludes cycles without root-cause

---

## 13. Boundary Constraints

This module MUST depend on:

- `analyzeCyclesForPR` (public API function)
- `buildPRComment` from `pr/prSurface.ts`
- `checkIgnoreRequest` from `pr/prIgnore.ts`
- GitHub PR API (external, via API client)

This module MUST NOT depend on:

- cycle detection modules directly (`analysis/cycles.ts`)
- diff modules directly (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- confidence modules directly (`confidence/computeConfidence.ts`)
- safety modules directly (`safety/safetySwitch.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)

This module MUST remain an orchestration layer.

---

## 14. Summary Checklist

`prHandler.ts` MUST:

✔ accept `PullRequestEvent` input structure  
✔ call `analyzeCyclesForPR` for cycle analysis  
✔ enforce confidence gating (`confidence >= 0.8`)  
✔ check safety switch status  
✔ fetch PR comments via GitHub API  
✔ extract fingerprint from existing comments  
✔ call `checkIgnoreRequest` to check ignore status  
✔ pair cycles with root-cause edges deterministically  
✔ call `buildPRComment` to generate comment body  
✔ compare old vs new comment content (exact string equality)  
✔ enforce PR Update Invariance (only update if content changed)  
✔ create/update/delete comments via GitHub API  
✔ handle all errors via silent mode  
✔ never throw exceptions  
✔ never log anything  
✔ never perform analysis, formatting, or matching logic  
✔ coordinate other modules correctly  
✔ return Promise that resolves (never rejects)  

`prHandler.ts` MUST NOT:

❌ detect cycles  
❌ perform diffing  
❌ compute confidence  
❌ format comment body  
❌ detect ignore replies  
❌ parse cycles or diffs  
❌ log or print anything  
❌ throw exceptions  
❌ retry failed API calls  
❌ perform partial operations  
❌ use timestamps or dynamic content in decision logic  
❌ depend on iteration order  

---

## 15. Authority

This contract is authoritative over all implementation of `src/pr/prHandler.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)

Any deviation from this contract MUST be approved via ADR and SSOT update.

