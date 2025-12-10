# ArcSight Determinism Contract (Phase 1 — Final Version)

This document defines the determinism guarantees of the ArcSight engine and the constraints required to maintain them.

It applies to:

- `arcsight-wedge` — MUST follow this contract fully
- `arcsight-cli` — MUST NOT break deterministic assumptions
- `arcsight-github-app` — MUST NOT introduce nondeterministic input into the engine

---

## 1. Deterministic Output Guarantee

For the same:

- Canonical `RepoSnapshot`
- Analyzer config
- Analyzer version
- Rulepack version

ArcSight MUST produce the exact same byte-for-byte `Envelope`, across:

- machines
- operating systems
- CPU architectures
- Node versions
- execution times

This applies equally to success, degraded, and error envelopes.

### 1.1 Deterministic Envelope Fields Matrix (NEW — Required)

This matrix defines which fields in the Envelope:

- MUST be deterministic
- MUST be canonicalized
- MUST NOT vary run-to-run
- MUST NOT contain runtime or environment pollutants

This prevents future contributors from accidentally introducing nondeterministic fields.

| Envelope Field | Deterministic? | Derived From | Notes |
|----------------|----------------|--------------|-------|
| `version.analyzer` | YES | Constant | Only changes when intentionally bumped |
| `version.schema` | YES | Constant | Schema evolution via adapters |
| `version.rulepack` | YES | Constant | No dynamic rulepack loading in Phase 1 |
| `identity.*` | YES | Runtime input | Must be copied exactly; no transformation |
| `core.cycles` | YES | RepoSnapshot | After canonical cycle rooting + sorting |
| `core.status` | YES | Engine rules | success / degraded / error |
| `core.limits` | YES | Constant config | No runtime-derived limit values |
| `core.graph_stats` | YES | Graph | Must be fully reproducible |
| `extensions.*` | YES | Rulepacks | Must be deterministic or null |
| `meta.config_snapshot_hash` | YES | Canonical config | stableStringify only |
| `meta.repo_fingerprint` | YES | Canonical snapshot | Deterministic based on paths + content |
| `meta.sandbox_policy_version` | YES | Constant | Sandbox policies cannot vary at runtime |
| `meta.signature` | YES | Entire envelope | Calculated via deepSort → stableStringify → SHA-256 |

**This table is binding.**

If a field is not listed here, it must be added before introducing it to the Envelope.

---

## 2. Forbidden Sources of Nondeterminism (Engine)

Forbidden APIs and behaviors in `arcsight-wedge/src`:

- `Date.now()`, `new Date()`, or any time-based API
- `Math.random()`
- Network I/O
- Filesystem I/O (tests only)
- Reading environment variables
- Mutable global state
- Node process state (CWD, env mutation)
- Non-deterministic concurrency or race ordering

**Any usage is a determinism violation.**

---

## 3. Canonicalization Requirements (Snapshot → Engine Input)

All analysis MUST operate on a **canonical** `RepoSnapshot`.

### 3.1 Path normalization

- POSIX normalized
- Collapse `.` and `..`
- Remove duplicate slashes
- Example: `./src//A.ts` → `src/A.ts`

### 3.2 Newline normalization

- All newlines MUST be `\n`

### 3.3 Unicode normalization

- All text contents MUST be normalized to **NFC form**
- (prevents OS-dependent Unicode drift)

### 3.4 File ordering

- Files MUST be sorted lexicographically by canonical path

### 3.5 Encoding normalization

- Treat contents as UTF-8 where possible
- Binary files MUST NOT introduce nondeterministic metadata or ordering

### 3.6 Snapshot metadata prohibition

`RepoSnapshot` MUST NOT include:

- timestamps
- file permissions
- inode numbers
- OS-specific metadata
- user/group IDs
- umask-influenced values

Only canonical path + canonical content may influence engine behavior.

### 3.7 Tarball-to-snapshot determinism (runtime contract)

Runtime MUST produce the same `RepoSnapshot` regardless of:

- OS tar implementation
- archive ordering
- symlink interpretation
- path sanitization

Symlinks, invalid paths, or `../` escapes MUST be handled identically.

---

## 4. Deterministic Data Structures

### 4.1 Ban on relying on JS object key iteration

Object iteration order is not deterministic when populated from dynamic data sources.

**Rules:**

- Use `Map` or arrays for internal collections
- Objects may only be constructed from pre-sorted keys

### 4.2 Canonical JSON serialization

Before signing:

```
Envelope → deepSort → stableStringify → SHA-256 signature
```

This enforces deterministic ordering for:

- nested arrays
- nested objects
- rulepack outputs
- extension fields

---

## 5. Graph & Cycle Determinism

### 5.1 Graph Determinism

Graph MUST be:

- built from sorted file list
- sorted for adjacency lists
- stable for `graph_stats`
- independent of algorithm traversal order

### 5.2 Cycle Determinism

**Mandatory rules:**

**Canonical cycle rooting (critical)**
- Rotate cycle so the lexicographically smallest node is index 0.

**Sorting cycles**
- Sort cycles by:
  1. length ascending
  2. lexicographical order of node sequence

**Cycle validity rules**
- No duplicate cycles
- No repeated node within a cycle

This guarantees cycle output is identical across all environments.

---

## 6. Deterministic Error Behavior

Error envelopes MUST be deterministic:

- No stack traces
- No machine paths
- No timestamps
- Only canonical `error_code`
- Error messages must be constant, not environment dependent

Error paths must be as deterministic as success paths.

---

## 7. Config Determinism

Analyzer config MUST be:

- Deep-sorted
- Canonicalized
- Stringified with `stableStringify` only

The `config_snapshot_hash` MUST be deterministic and based strictly on canonical config content.

---

## 8. Concurrency Rules

Phase 1 engine MUST be **single-threaded**.

If concurrency appears in Phase 2:

- Work units MUST NOT influence output ordering
- All intermediate results MUST be sorted before use
- No reliance on completion or worker scheduling order

Parallelism MUST not leak nondeterminism into the envelope.

---

## 9. Testing Determinism

Determinism MUST be validated with:

### 9.1 Golden tests

Canonical test repos MUST produce identical envelopes over time.

### 9.2 Repeat-run tests

Running `analyze()` twice must produce byte-for-byte identical output.

### 9.3 Cross-environment tests (CI)

Goldens MUST match across different environments.

### 9.4 Deep structural comparison

Tests MUST compare deep structure, NOT just JSON output.

This detects ordering drift early.

---

## 10. Violations & Severity

Any violation of this contract is a **Critical determinism bug**.

**Required actions:**

- Immediate fix
- Update or add golden tests
- Possibly bump `analyzer_version`
- Document root cause

A nondeterministic analyzer is unacceptable.

---

## 11. Ownership & Enforcement

- Engine team owns this document
- Runtime & CLI teams must respect the canonical snapshot and envelope contracts
- Cursor must enforce these rules per the development protocol
- Any architectural ambiguity MUST resolve in favor of determinism

---

## 12. References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Folder Scaffold](./02-phase1-folder-scaffold.md)
- [System Roadmap](./03-full-system-roadmap.md)
- [Dependency Contract](./5A-dependency-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Cursor Development Protocol](./10C-cursor-development-protocol.md)

---

# 🎉 FINAL RESULT

This is the complete, hardened, FAANG-grade determinism contract for ArcSight.

Nothing else is needed.

This is now architecturally unbreakable.
