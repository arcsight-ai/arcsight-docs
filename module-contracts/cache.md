# Module Contract — src/cache/cache.ts (V1 FINAL LOCKED)

**Layer:** cache  
**Responsibility:** Deterministic cache of structural truth for commit-level analysis results.

This module provides:
- Deterministic cache key generation
- Cache write operations (when allowed)
- Cache read operations (when allowed)
- Cache invalidation rules

This module MUST NOT:
- cache PR-level results (PR context is too dynamic)
- cache results when silent mode was active (uncertainty)
- cache results when errorDetected = true
- introduce non-deterministic cache keys
- store partial or speculative resultsnpm run build

- modify wedge analysis behavior
- log anything
- maintain hidden state

It is a pure, deterministic caching layer.

---

## 1. Inputs & Outputs

### 1.1 Cache Key Derivation

**Key Components:**
- `repoPath`: Normalized, POSIX, lowercase repository path
- `headCommit`: Git commit SHA (normalized, lowercase)
- `WEDGE_VERSION`: Constant string "v1" (hard-coded)

**Key Generation Rule (MANDATORY):**
```
cacheKey = `${normalizePath(repoPath)}:${headCommit.toLowerCase()}:v1`
```

**Normalization Requirements:**
- `repoPath` MUST be:
  - Resolved to absolute path
  - Converted to POSIX format (`\` → `/`)
  - Lowercased
  - Trailing slashes removed
  
- `headCommit` MUST be:
  - Lowercase
  - Trimmed of whitespace
  - Validated as 40-character hex string (or short SHA)

**Determinism Guarantee:**
Same `(repoPath, headCommit)` MUST always produce the same cache key across:
- Different machines
- Different filesystem layouts
- Different Node.js versions
- Different execution contexts

### 1.2 CachedResult Interface

```typescript
interface CachedResult {
  // Cache metadata
  cacheKey: string;
  cachedAt: number; // Unix timestamp (milliseconds)
  wedgeVersion: string; // Always "v1"
  
  // Analysis results (same shape as CommitAnalysisResult)
  importAnalysis: ImportAnalysisResult | null;
  cycleDetection: CycleDetectionResult | null;
  cycleDiff: CycleDiffResult | null;
  rootCause: RootCauseResult | null;
  confidence: number | null;
  invariants: InvariantsResult | null;
  safety: SafetyDecision | null;
  snapshotWritten: boolean;
  errorDetected: boolean;
  
  // Cache validity flags
  wasSilent: boolean; // If true, cache read is forbidden
  wasError: boolean; // If true, cache read is forbidden
}
```

**Type Definitions:**
All types referenced here are defined in `types.md` and module contracts.

---

## 2. Cache Write Rules (MANDATORY)

### 2.1 When Cache Write IS Allowed

Cache write MUST occur ONLY when ALL of the following conditions are met:

1. **Analysis completed successfully:**
   - `errorDetected === false`
   - All pipeline stages completed without exceptions

2. **Not in silent mode:**
   - `confidence !== null && confidence >= 0.8`
   - `safety.allowSurface === true`
   - `invariants.allInvariantsSatisfied === true`

3. **Deterministic results:**
   - All canonical cycles are deterministically formatted
   - Import graph is stable
   - No ambiguity in results

4. **Valid cache key:**
   - `repoPath` is valid and accessible
   - `headCommit` is a valid git SHA
   - Cache key derivation succeeded

### 2.2 When Cache Write IS FORBIDDEN

Cache write MUST NOT occur when ANY of the following is true:

1. **Error detected:**
   - `errorDetected === true`
   - Any pipeline stage threw an exception
   - Analysis incomplete

2. **Silent mode active:**
   - `confidence === null || confidence < 0.8`
   - `safety.allowSurface === false`
   - `invariants.allInvariantsSatisfied === false`

3. **Uncertainty present:**
   - Alias ambiguity detected
   - Monorepo detected (low confidence)
   - File limit exceeded
   - Determinism violation detected

4. **Invalid inputs:**
   - `repoPath` is invalid or inaccessible
   - `headCommit` is invalid or not found
   - Cache key derivation failed

5. **PR-level analysis:**
   - PR context is too dynamic
   - Base/head comparison results are context-dependent
   - Changed files vary by PR metadata

**Rationale:**
Caching uncertain or error-state results would:
- Violate determinism guarantees
- Cache incorrect structural truth
- Break silent-mode semantics
- Introduce false positives

---

## 3. Cache Read Rules (MANDATORY)

### 3.1 When Cache Read IS Allowed

Cache read MUST occur ONLY when ALL of the following conditions are met:

1. **Valid cache key:**
   - Key derivation succeeded
   - Cache entry exists for the key

2. **Cache entry is valid:**
   - `cachedResult.wasSilent === false`
   - `cachedResult.wasError === false`
   - `cachedResult.wedgeVersion === "v1"`
   - `cachedResult.errorDetected === false`

3. **Cache entry is fresh:**
   - Cache TTL rules are satisfied (if implemented)
   - Cache entry has not been invalidated

4. **Inputs match:**
   - Current `repoPath` matches cached `repoPath`
   - Current `headCommit` matches cached `headCommit`
   - Current `WEDGE_VERSION` matches cached `wedgeVersion`

### 3.2 When Cache Read IS FORBIDDEN

Cache read MUST NOT occur when ANY of the following is true:

1. **Was silent:**
   - `cachedResult.wasSilent === true`
   - Cache entry represents uncertain analysis
   - Silent mode results are not trustworthy

2. **Was error:**
   - `cachedResult.wasError === true`
   - `cachedResult.errorDetected === true`
   - Error-state results must not be reused

3. **Version mismatch:**
   - `cachedResult.wedgeVersion !== "v1"`
   - Cache entry from different wedge version
   - Incompatible result structure

4. **Cache entry missing:**
   - No cache entry exists for the key
   - Cache file is corrupted or unreadable

5. **Input mismatch:**
   - Current `repoPath` does not match cached `repoPath`
   - Current `headCommit` does not match cached `headCommit`

**Rationale:**
Reading uncertain or error-state cache entries would:
- Violate silent-mode guarantees
- Reuse incorrect structural truth
- Break determinism
- Introduce false positives

---

## 4. Explicitly Forbidden Cache Contents

The cache MUST NOT store:

1. **PR-level results:**
   - PR numbers
   - Changed files lists
   - PR-specific diff results
   - PR comment content

2. **Uncertain results:**
   - Silent mode outputs
   - Error-state outputs
   - Low-confidence results (< 0.8)
   - Ambiguous alias resolutions

3. **Non-deterministic data:**
   - Timestamps (except `cachedAt` metadata)
   - Random values
   - Environment-specific paths
   - Machine-specific identifiers

4. **Partial results:**
   - Incomplete analysis stages
   - Speculative cycles
   - Unvalidated invariants

5. **Wedge-internal state:**
   - Safety switch internal flags
   - Cooldown state
   - Error path internal state
   - Invariant violation details

**Rationale:**
These items are either:
- Too dynamic (PR context)
- Too uncertain (silent mode)
- Non-deterministic (timestamps, randomness)
- Internal implementation details

---

## 5. Cache Storage Requirements

### 5.1 Storage Location

Cache files MUST be stored in:
- Location: `{cacheDir}/arcsight-cache/{normalizedRepoPath}/{headCommit}.json`
- Format: NDJSON (one cache entry per line) OR single JSON file per commit
- Encoding: UTF-8

**Cache Directory Rules:**
- `cacheDir` MUST be configurable (default: OS temp directory)
- Path normalization MUST be applied consistently
- Directory structure MUST be deterministic

### 5.2 Cache File Format

Each cache entry MUST be:
- Valid JSON
- Deterministically serialized (sorted keys)
- Atomic writes (write to temp file, then rename)
- Append-only (if using NDJSON) OR overwrite (if using single file)

### 5.3 Cache Invalidation

Cache entries MUST be invalidated when:
- `WEDGE_VERSION` changes
- Cache file is corrupted
- Cache entry is too old (if TTL implemented)
- Manual invalidation requested

---

## 6. Determinism Requirements

### 6.1 Key Derivation Determinism

Cache key generation MUST be:
- Identical for same inputs across all environments
- Independent of filesystem layout
- Independent of Node.js version
- Independent of execution context

### 6.2 Cache Read/Write Determinism

Cache operations MUST be:
- Idempotent (same operation, same result)
- Order-independent (read before write = write then read)
- Race-condition safe (atomic operations)

### 6.3 Result Serialization Determinism

Cached results MUST be:
- Serialized with sorted object keys
- Normalized paths (POSIX, lowercase)
- Canonical cycle format
- Deterministic JSON structure

---

## 7. Error Handling

### 7.1 Cache Write Errors

If cache write fails:
- MUST NOT throw exception
- MUST NOT affect analysis results
- MUST log error to stderr (tooling exception)
- Analysis continues normally (cache is optional)

### 7.2 Cache Read Errors

If cache read fails:
- MUST return `null` (cache miss)
- MUST NOT throw exception
- MUST NOT affect analysis flow
- Analysis continues with fresh computation

### 7.3 Cache Corruption

If cache file is corrupted:
- MUST treat as cache miss
- MUST NOT attempt repair
- MUST NOT throw exception
- Analysis continues with fresh computation

---

## 8. Boundary Constraints

This module MUST NOT depend on:
- PR modules (`prHandler.ts`, `prSurface.ts`, `prIgnore.ts`)
- Safety modules beyond reading results (`safetySwitch.ts`, `cooldown.ts`)
- Snapshot modules (`snapshotWriter.ts`)

This module MAY depend on:
- `types.md` for type definitions
- Node.js `fs` for file operations
- `path` for path normalization

This module MUST remain:
- Pure (no side effects beyond cache I/O)
- Deterministic (same inputs → same cache operations)
- Optional (analysis works without cache)

---

## 9. Performance Requirements

### 9.1 Cache Read Performance

Cache read MUST:
- Complete in < 50ms for typical cache entries
- Not block analysis pipeline
- Use efficient JSON parsing

### 9.2 Cache Write Performance

Cache write MUST:
- Complete in < 100ms for typical results
- Not block analysis pipeline
- Use atomic file operations

### 9.3 Cache Size Limits

Cache entries MUST:
- Not exceed reasonable size limits (e.g., 10MB per entry)
- Be pruned if cache grows too large
- Not cause disk space issues

---

## 10. Summary

`cache.ts` MUST:

✔ Generate deterministic cache keys (repoPath + headCommit + WEDGE_VERSION)  
✔ Write cache only when analysis succeeded and not silent  
✔ Read cache only when entry is valid and not silent/error  
✔ Forbid caching PR results, uncertain results, or error states  
✔ Maintain determinism across all cache operations  
✔ Handle errors gracefully without affecting analysis  
✔ Remain optional (analysis works without cache)  
✔ Preserve wedge contracts (no logging, no hidden state)  

It is a deterministic, optional caching layer that accelerates repeated commit analyses without compromising wedge guarantees.

