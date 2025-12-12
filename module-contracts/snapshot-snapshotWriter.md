# Module Contract — src/snapshot/snapshotWriter.ts (V1 FINAL LOCKED)

**Layer:** snapshot

**Responsibility:** Deterministic append-only writing of cycle snapshots to persistent storage, ensuring cycle history is preserved without affecting PR output on failure.

This module produces:

- Append-only snapshot storage (NDJSON format)
- Deterministic snapshot persistence
- Fail-safe error handling (failures do not affect PR output)

This module MUST NOT:

- read historical snapshots (no read API in V1)
- compute deltas or trends
- modify existing snapshots (append-only)
- export FS helpers usable elsewhere
- log anything
- infer architecture
- maintain state across calls
- affect PR output on failure

It is a pure append-only storage module.

---

## 1. Module Purpose & Responsibilities

**Location:** `snapshot/`

**Scope:** This module operates on `CycleSnapshot` objects to write them to persistent storage.

**Function:** Writes cycle snapshots to append-only storage in a deterministic format, ensuring cycle history is preserved for future analysis (post-V1).

**This module ensures snapshots are written deterministically and failures do not affect PR output.**

**Boundaries:**

- Does NOT read snapshots (no read API in V1)
- Does NOT compute deltas or trends
- Does NOT modify existing snapshots (append-only only)
- Does NOT export FS helpers usable elsewhere
- Does NOT affect PR output on failure (fail-safe)
- MUST write snapshots deterministically
- MUST be append-only
- MUST be a pure function (no state)

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface CycleSnapshot {
  repoId: string;
  commitSha: string;
  timestamp: string;  // ISO-8601 UTC, second precision
  canonicalCycles: CanonicalCycle[];
  confidence: number;  // [0, 1]
}

function writeSnapshot(
  snapshot: CycleSnapshot,
  storagePath: string
): Promise<void>
```

**Input assumptions:**

- `snapshot`: Fully-formed `CycleSnapshot` object
  - `repoId`: Repository identifier (string, chosen by caller)
  - `commitSha`: Commit SHA or unique identifier (string)
  - `timestamp`: ISO-8601 UTC timestamp with second precision (string, provided by caller)
  - `canonicalCycles`: Array of canonical cycles, already sorted lexicographically (caller responsibility)
  - `confidence`: Confidence score in [0, 1] (number)
- `storagePath`: Base directory path for snapshot storage (string)
  - Directory will be created if missing
  - Path MUST be valid and writable
  - Path MUST be deterministic (same path for same repoId)

**Input validation:**

- If `snapshot` is missing or `null`/`undefined` → throw Error (fail-safe)
- If `storagePath` is missing or empty → throw Error (fail-safe)
- If `snapshot.repoId` is missing or empty → throw Error (fail-safe)
- If `snapshot.commitSha` is missing or empty → throw Error (fail-safe)
- If `snapshot.timestamp` is missing or invalid ISO-8601 format → throw Error (fail-safe)
- If `snapshot.canonicalCycles` is missing or not an array → throw Error (fail-safe)
- If `snapshot.confidence` is missing or not a number → throw Error (fail-safe)
- If `snapshot.confidence` is not in [0, 1] → throw Error (fail-safe)

### 2.2 Output

**Function signature:**

```ts
function writeSnapshot(
  snapshot: CycleSnapshot,
  storagePath: string
): Promise<void>
```

**Return types:**

- `Promise<void>`: Resolves on successful write, rejects on failure

**Output semantics:**

- Resolves: Snapshot successfully written to storage
- Rejects: Error occurred during write (caller MUST treat as silent-error, no PR output impact)

**Output rules:**

- MUST throw Error on validation failure
- MUST throw Error on filesystem failure
- MUST NOT log errors
- MUST NOT affect PR output on failure (fail-safe)
- MUST be deterministic (same input → same write behavior)

---

## 3. Storage Format & File Structure

### 3.1 File Format

**Format:** NDJSON (Newline-Delimited JSON)

- One JSON object per line
- Each line is a complete, valid JSON object
- Lines are separated by `\n` (LF, not CRLF)
- Append-only: new snapshots are appended to the end of the file

### 3.2 File Naming

**Deterministic file naming:**

- Filename: `{repoId}.ndjson`
- Full path: `{storagePath}/{repoId}.ndjson`
- Example: `snapshots/my-org/my-repo.ndjson`

**Naming rules:**

- `repoId` MUST be used as-is (no normalization beyond what caller provides)
- Filename MUST be deterministic (same repoId → same filename)
- Path separator MUST be POSIX (`/`)

### 3.3 Directory Structure

**Directory creation:**

- If `storagePath` directory does not exist → create it (recursive)
- If parent directories do not exist → create them (recursive)
- Directory creation MUST be deterministic (same path → same directory structure)

### 3.4 JSON Serialization

**Deterministic JSON serialization:**

- JSON keys MUST be sorted alphabetically
- JSON MUST use 2-space indentation (for readability, but single-line in NDJSON)
- JSON MUST use UTF-8 encoding
- JSON MUST NOT include trailing commas
- JSON MUST escape special characters correctly
- Floating-point numbers MUST be serialized deterministically (no precision loss beyond standard JSON)

**JSON structure:**

```json
{
  "canonicalCycles": ["a → b → a"],
  "commitSha": "abc123",
  "confidence": 0.85,
  "repoId": "my-org/my-repo",
  "timestamp": "2025-01-15T12:34:56Z"
}
```

**Key ordering (alphabetical):**

1. `canonicalCycles`
2. `commitSha`
3. `confidence`
4. `repoId`
5. `timestamp`

---

## 4. Deterministic Behavior Rules

`snapshotWriter.ts` MUST:

- Produce identical file content for identical input
- Use deterministic JSON serialization (sorted keys, fixed formatting)
- Create directories deterministically (same path → same structure)
- Write files deterministically (append-only, no overwrites)
- Use deterministic file naming (same repoId → same filename)
- Never use timestamps in decision logic (use provided timestamp)
- Never depend on environment or runtime state
- Never maintain state across calls (pure function)

`snapshotWriter.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets (must sort keys)
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Store state in module-level variables
- Cache results across calls
- Read existing snapshots to determine write behavior

**JSON serialization determinism:**

- Keys MUST be sorted alphabetically before serialization
- Arrays MUST preserve order (canonicalCycles order is deterministic from caller)
- Numbers MUST be serialized with standard JSON precision
- Strings MUST be serialized with standard JSON escaping

**File system determinism:**

- Directory creation MUST be deterministic (same path → same structure)
- File writes MUST be append-only (no overwrites, no truncation)
- File naming MUST be deterministic (same repoId → same filename)

---

## 5. Error Handling & Fail-Safe Rules

`snapshotWriter.ts` MUST:

- Throw Error on validation failure (caller handles)
- Throw Error on filesystem failure (caller handles)
- Never log errors (no console output)
- Never affect PR output on failure (fail-safe)
- Never write partial snapshots (atomic append or fail)

**Error handling contract:**

- Validation errors → throw Error immediately
- Filesystem errors (permissions, disk full, etc.) → throw Error
- Directory creation failures → throw Error
- File write failures → throw Error
- JSON serialization errors → throw Error

**Fail-safe principle:**

- When in doubt, throw Error (caller treats as silent-error)
- Invalid input → throw Error
- Filesystem failure → throw Error
- Partial write → throw Error (no partial snapshots)

**Caller responsibility:**

- Caller MUST catch errors and treat as silent-error
- Caller MUST NOT let snapshot write failures affect PR output
- Caller MUST continue PR processing even if snapshot write fails

---

## 6. Input Validation Rules

`snapshotWriter.ts` MUST validate:

### 6.1 Snapshot Structure Validation

- `snapshot` must be an object (not null, not undefined)
- `snapshot.repoId` must be a non-empty string
- `snapshot.commitSha` must be a non-empty string
- `snapshot.timestamp` must be a valid ISO-8601 UTC string with second precision
- `snapshot.canonicalCycles` must be an array (may be empty)
- `snapshot.confidence` must be a number in [0, 1]

### 6.2 Timestamp Format Validation

**ISO-8601 UTC format (second precision):**

- Format: `YYYY-MM-DDTHH:mm:ssZ` or `YYYY-MM-DDTHH:mm:ss+00:00`
- Examples:
  - Valid: `"2025-01-15T12:34:56Z"`
  - Valid: `"2025-01-15T12:34:56+00:00"`
  - Invalid: `"2025-01-15T12:34:56.123Z"` (millisecond precision not allowed)
  - Invalid: `"2025-01-15T12:34:56"` (missing timezone)

**Validation rules:**

- MUST match ISO-8601 UTC pattern with second precision
- MUST include timezone indicator (`Z` or `+00:00`)
- MUST NOT include milliseconds or microseconds
- If invalid → throw Error

### 6.3 Canonical Cycles Validation

- `canonicalCycles` must be an array
- Each element must be a string (CanonicalCycle)
- Array may be empty (no cycles)
- Array order is preserved (caller responsibility to sort)

### 6.4 Confidence Validation

- `confidence` must be a number
- `confidence` must be finite (not NaN, not Infinity)
- `confidence` must be in range [0, 1]
- If out of range → throw Error

### 6.5 Storage Path Validation

- `storagePath` must be a non-empty string
- `storagePath` must be a valid directory path
- If invalid → throw Error

---

## 7. Forbidden Behaviors

`snapshotWriter.ts` MUST NOT:

- Read existing snapshots (no read API in V1)
- Compute deltas or trends
- Modify existing snapshots (append-only only)
- Export FS helpers usable elsewhere
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps in decision logic (use provided timestamp)
- Depend on environment variables
- Maintain state across calls (must be pure function)
- Store snapshots in module-level variables
- Cache results across calls
- Overwrite existing snapshots (append-only)
- Truncate existing files
- Read files to determine write behavior
- Use nondeterministic APIs
- Access filesystem beyond write operations
- Modify input snapshot object

---

## 8. Edge Cases

### 8.1 Empty canonicalCycles

**Input:** `canonicalCycles: []`

**Output:** Valid snapshot written with empty array

**Behavior:** Empty cycles array is valid (no cycles detected)

### 8.2 Storage path does not exist

**Input:** `storagePath` pointing to non-existent directory

**Output:** Directory created recursively, then snapshot written

**Behavior:** Create directory structure deterministically

### 8.3 Concurrent writes

**Input:** Multiple `writeSnapshot` calls for same `repoId` simultaneously

**Output:** All snapshots appended (NDJSON format supports concurrent appends)

**Behavior:** Append-only writes are safe for concurrent access (OS-level file append is atomic)

### 8.4 Disk full

**Input:** Filesystem has no space

**Output:** Error thrown

**Behavior:** Throw Error, caller treats as silent-error

### 8.5 Permission denied

**Input:** Insufficient permissions to write to `storagePath`

**Output:** Error thrown

**Behavior:** Throw Error, caller treats as silent-error

### 8.6 Invalid timestamp format

**Input:** `timestamp` not in ISO-8601 UTC format

**Output:** Error thrown

**Behavior:** Validate format, throw Error on invalid

### 8.7 Confidence out of range

**Input:** `confidence < 0` or `confidence > 1`

**Output:** Error thrown

**Behavior:** Validate range, throw Error on invalid

### 8.8 Missing required fields

**Input:** `snapshot` missing `repoId`, `commitSha`, `timestamp`, `canonicalCycles`, or `confidence`

**Output:** Error thrown

**Behavior:** Validate all required fields, throw Error on missing

### 8.9 Very long repoId

**Input:** `repoId` exceeds filesystem filename length limits

**Output:** Error thrown (or filesystem error)

**Behavior:** Filesystem will reject, throw Error

### 8.10 Very large snapshot

**Input:** `canonicalCycles` array contains thousands of cycles

**Output:** Snapshot written (no size limit in V1, but may be slow)

**Behavior:** Write all cycles (no truncation, no filtering)

---

## 9. Integration with Other Modules

**Caller responsibilities:**

- Caller must provide fully-formed `CycleSnapshot` with valid timestamp
- Caller must ensure `canonicalCycles` are sorted lexicographically
- Caller must catch errors and treat as silent-error (no PR output impact)
- Caller must provide valid `storagePath` directory

**Integration flow:**

1. Analysis modules produce cycle data
2. Caller constructs `CycleSnapshot` with:
   - `repoId`: Repository identifier
   - `commitSha`: Commit SHA
   - `timestamp`: ISO-8601 UTC timestamp (caller generates)
   - `canonicalCycles`: Sorted canonical cycles
   - `confidence`: Confidence score
3. Caller calls `writeSnapshot(snapshot, storagePath)`
4. If write succeeds → snapshot persisted
5. If write fails → caller treats as silent-error, continues PR processing

**This module does NOT:**

- Read from snapshots directly
- Access PR API directly
- Inspect cycles or imports
- Modify data structures
- Compute cycles or diffs
- Affect PR output on failure

---

## 10. Boundary Constraints

This module MUST depend on:

- Node.js `fs` module (for file writing)
- Node.js `path` module (for path manipulation)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- safety modules (`safety/safetySwitch.ts`, etc.)
- confidence modules (`confidence/computeConfidence.ts`)
- analysis modules (`analysis/imports.ts`, etc.) - except for type imports
- other snapshot modules (none exist)

This module MUST remain a pure storage layer with hard isolation.

**Type imports allowed:**

- `CycleSnapshot` type from `types.md` or shared type definitions
- No runtime dependencies on other modules

---

## 11. Summary Checklist

`snapshotWriter.ts` MUST:

✔ accept `CycleSnapshot` and `storagePath` inputs  
✔ validate `CycleSnapshot` structure deterministically  
✔ validate timestamp format (ISO-8601 UTC, second precision)  
✔ validate confidence range [0, 1]  
✔ create storage directory if missing (recursive)  
✔ write snapshots in NDJSON format (append-only)  
✔ use deterministic file naming (`{repoId}.ndjson`)  
✔ use deterministic JSON serialization (sorted keys)  
✔ throw Error on validation failure  
✔ throw Error on filesystem failure  
✔ never log anything  
✔ never affect PR output on failure (fail-safe)  
✔ never read existing snapshots (no read API in V1)  
✔ never modify existing snapshots (append-only)  
✔ never export FS helpers usable elsewhere  
✔ produce identical file content for identical inputs  
✔ use deterministic file system operations  
✔ never maintain state across calls  
✔ never use timestamps in decision logic  

`snapshotWriter.ts` MUST NOT:

❌ read existing snapshots (no read API in V1)  
❌ compute deltas or trends  
❌ modify existing snapshots (append-only only)  
❌ export FS helpers usable elsewhere  
❌ log or print anything  
❌ infer architecture  
❌ add new signals  
❌ use timestamps in decision logic  
❌ depend on environment variables  
❌ throw exceptions that affect PR output  
❌ use nondeterministic APIs  
❌ maintain state across calls  
❌ store snapshots in module-level variables  
❌ cache results across calls  
❌ overwrite existing snapshots  
❌ truncate existing files  
❌ read files to determine write behavior  

---

## 12. Authority

This contract is authoritative over all implementation of `src/snapshot/snapshotWriter.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract (Section 2.4: writeSnapshot)
- Module Boundaries (Section 2.6: snapshot/ with Append-Only Rule)
- Types (`types.md` Section 5: CycleSnapshot)
- Determinism Requirements
- Production Guide (Section 5: Snapshot Requirements)
- ADRs
- v1-design-doc.md

Any deviation from this contract MUST be approved via ADR and SSOT update.

