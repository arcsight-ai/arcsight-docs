# RepoSnapshot Contract (FINAL UPDATED VERSION, Phase 1)

## 1. Purpose

This document defines the canonical RepoSnapshot format, the deterministic rules for constructing it, and the invariants required by the ArcSight engine.

RepoSnapshot is the sole input to the deterministic analyzer (`arcsight-wedge`).

It ensures:

- identical input → identical graph, cycles, envelope
- consistent behavior across machines, OSes, tar implementations
- stable hashing suitable for drift detection and caching
- strict separation of runtime responsibilities and engine purity

This contract prevents nondeterminism from entering the analysis pipeline.

---

## 2. Scope

This document governs:

### Included

- RepoSnapshot JSON shape
- canonicalization rules
- path normalization
- newline + Unicode normalization
- binary handling
- deterministic hashing
- snapshot metadata
- rejection and failure behavior
- serialization determinism
- symlink / hard link / submodule behavior
- snapshot size limits

### Not Included

- envelope structure (see [Envelope Format Spec](./15-envelope-format-spec.md))
- schema evolution (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- rulepack behavior (see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- runtime execution model (see [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md))
- engine invariants (see [Invariants Contract](./06-invariants-contract.md))

---

## 3. Definitions & Terms

**RepoSnapshot**  
Deterministic representation of a repository's file tree after canonicalization.

**Snapshot Builder**  
Runtime component that converts a tarball or filesystem tree into a RepoSnapshot.

**Canonicalization**  
Transformations that produce deterministic paths, content, and metadata.

**Binary File**  
A file whose extracted bytes are not valid UTF-8 text.

**Snapshot Hash**  
SHA-256 hash over the canonical snapshot using `stableStringify`.

---

## 4. Rules / Contract

### 4.1 RepoSnapshot Shape

A valid snapshot MUST match exactly:

```typescript
interface RepoSnapshot {
  files: Array<{
    path: string;          // canonical POSIX path
    content: string | null; // UTF-8 normalized text, or null for binary
    hash: string;          // SHA-256 hash of canonical content or raw bytes
    is_binary: boolean;    // indicates binary treatment
  }>;

  metadata: {
    repo_fingerprint: string; // hash of entire snapshot
    file_count: number;
    total_bytes: number;
    snapshot_version: 1;      // fixed for Phase 1
  };
}
```

No additional fields are permitted.

### 4.2 Canonical Path Rules

Paths MUST:

- use `/` separators
- collapse `.` and `..` segments
- collapse repeated slashes
- remove leading `./`
- never end in `/`
- never contain `\`
- never contain null bytes
- never escape root (e.g., `../../etc/passwd` MUST be rejected)

Forbidden entries MUST result in snapshot rejection.

### 4.3 Newline Normalization

All non-binary text MUST normalize:

```
CRLF → \n  
CR   → \n
```

Binary files MUST NOT undergo newline normalization.

### 4.4 Unicode Normalization

Text files MUST be normalized using:

**NFC (Normalization Form C)**

Binary files MUST NOT undergo Unicode normalization.

### 4.5 Binary File Detection

A file MUST be treated as binary if:

- UTF-8 decoding fails
- it contains disallowed control bytes
- it is recognized as binary by deterministic heuristics

Binary file rules:

```
content = null  
is_binary = true  
hash = SHA-256(raw bytes)
```

The engine MUST NOT inspect binary contents.

### 4.6 Symlink, Hard Link & Submodule Handling (NEW)

Runtime MUST apply the following rules before snapshot creation:

**Symlinks**

- Symlinks MUST be rejected.
- Snapshot builder MUST NOT follow or include them.
- The presence of any symlink causes a deterministic error.

**Submodules**

- Git submodules MUST be treated as empty directories.
- No traversal into submodule contents.
- The engine MUST NOT receive any files from submodule paths.

**Hard Links**

- Hard links MUST be treated as independent files.
- Runtime MUST NOT deduplicate based on inode or FS identity.
- Hashing MUST occur on the extracted file bytes.

**Path Escape Safety**

- Any file whose normalized path escapes the repo root MUST cause rejection.

These rules prevent filesystem nondeterminism and archive-based exploit surfaces.

### 4.7 Deterministic Sorting

All files MUST be sorted lexicographically by canonical path prior to:

- snapshot hash computation
- runtime-to-engine handoff
- golden test generation
- envelope production

Sorting MUST be stable.

### 4.8 Deterministic Binary Hashing (NEW)

Binary file hashing MUST occur over exact extracted bytes, with:

❌ no newline conversion  
❌ no Unicode normalization  
❌ no BOM removal  
❌ no charset reinterpretation  
❌ no trimming of null bytes  

Binary hashes MUST be raw-byte SHA-256.

### 4.9 Snapshot Size Limits (NEW)

Runtime MUST enforce deterministic snapshot limits before constructing RepoSnapshot:

- `max_file_count`
- `max_total_bytes`
- `max_path_length`
- `max_directory_depth` (if applicable)

If any limit is exceeded:

- snapshot MUST NOT be created
- engine MUST NOT be invoked
- runtime MUST return a deterministic error envelope

Limits reference:

[Limits and Sandbox Policy](./13-limits-and-sandbox-policy.md)

### 4.10 Snapshot Metadata Rules

| Field | Source | Deterministic? | Notes |
|-------|--------|----------------|-------|
| `repo_fingerprint` | `stableStringify(snapshot)` → sha256 | YES | used in envelope meta |
| `file_count` | `files.length` | YES | used for limits |
| `total_bytes` | sum of raw bytes | YES | must match binary vs text inputs |
| `snapshot_version` | constant = 1 | YES | increment only on structural changes |

### 4.11 Serialization Determinism (NEW)

Snapshots MUST serialize deterministically:

- stable field ordering
- stable array ordering
- no undefined values
- JSON-serializable only

MUST survive round-trip:

```
snapshot === stableParse(stableStringify(snapshot))
```

This is required for:

- snapshot caching
- shadow analyzer comparisons
- drift detection
- reproducible golden tests

### 4.12 Hash Algorithm Stability (NEW)

Phase 1 MUST use:

**SHA-256** for all file hashes and `repo_fingerprint`.

Changing hash algorithm REQUIRES:

- `snapshot_version` bump
- `analyzerVersion` bump
- schema adapter updates
- golden test updates

Hash changes MUST be treated as structural schema changes.

---

## 5. Determinism Impact

RepoSnapshot is the root source of determinism.

If snapshot construction is nondeterministic, all downstream outputs diverge:

- graph structure
- cycles
- rulepack outputs
- envelope signature
- drift detection
- shadow analyzer results
- golden tests

This contract ensures reproducibility across all execution environments.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- fully canonicalize snapshot
- enforce path safety, limits, and binary detection
- treat snapshot as immutable
- never mutate snapshot post-creation
- never "fix up" malformed snapshots
- reject invalid repos deterministically

**Engine MUST:**

- assume snapshot is canonical
- validate invariants
- refuse malformed snapshots
- operate without filesystem access

---

## 7. Versioning Impact

- `snapshot_version = 1` for Phase 1.

**`snapshot_version` MUST increment when:**

- new fields are added
- field types change
- canonicalization rules change
- hashing rules change
- symlink/hardlink/submodule rules change
- binary detection algorithm changes

**`snapshot_version` MUST NOT increment for:**

- performance refactors
- internal runtime improvements
- implementation detail changes that do NOT affect snapshot shape

**`snapshot_version++` also requires:**

- `analyzerVersion++`
- schema adapter updates
- new golden tests

---

## 8. Testing Requirements

### 8.1 Golden Snapshot Tests

Goldens MUST include:

- binary files
- CRLF → LF cases
- unicode normalization
- invalid paths
- symlinks (expect rejection)
- submodule presence
- large file counts
- long paths
- multibyte characters

Snapshots MUST be identical across OS, Node versions, and archive formats.

### 8.2 Serialization Round-Trip Tests

Every snapshot MUST satisfy:

```
snapshot === stableParse(stableStringify(snapshot))
```

### 8.3 Limit Failure Tests

Repos exceeding configured limits MUST:

- fail deterministically
- never reach `analyze()`
- return stable, testable error output

---

## 9. Cross-References

- [Full System Roadmap](./03-full-system-roadmap.md)
- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [Limits and Sandbox Policy](./13-limits-and-sandbox-policy.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added deterministic handling for symlinks, hard links, submodules, size limits, binary hashing rules, serialization determinism, and hash algorithm stability.

**v1.0.0** — Initial definition of RepoSnapshot v1 canonicalization rules.

