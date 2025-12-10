# Runtime ↔ Engine Contract (Phase 1 Final, Updated)

## The ArcSight Deterministic Engine & GitHub Runtime Boundary Contract

This document defines the formal, permanent contract between:

- `arcsight-wedge` — the deterministic engine
- `arcsight-github-app` — the runtime orchestrator

This contract ensures:

- deterministic output
- purity of analysis
- strict responsibility boundaries
- schema stability
- long-term maintainability
- safety for Phase-2 shadow analyzers

This is one of the most important documents in ArcSight's architecture.

---

## 1. Purpose

This contract ensures:

✔ Engine outputs remain deterministic  
✔ Runtime cannot influence deterministic analysis  
✔ Engine does not depend on runtime behavior  
✔ Snapshot → Envelope pipeline remains pure  
✔ Schema evolution is predictable  
✔ Sentinel (Phase 2) can run multiple analyzers safely  

This boundary is strict, final, and inviolable.

---

## 2. High-Level Contract Summary

| Component | Responsibilities | Forbidden |
|-----------|-----------------|-----------|
| **Engine** (`arcsight-wedge`) | Pure deterministic analysis. Accepts `RepoSnapshot` + config. Produces `Envelope`. | No FS, no network, no env vars, no GitHub awareness. |
| **Runtime** (`arcsight-github-app`) | IO, GitHub integration, tarball → snapshot, Check Runs, retries, DLQ. | Must not influence or modify deterministic engine output. |

**Engine is pure.**  
**Runtime is impure.**  
**The boundary is absolute.**

---

## 3. Engine Input Contract

### 3.1 RepoSnapshot — the ONLY engine input

Runtime MUST produce a `RepoSnapshot` that is:

- canonical
- deterministic
- immutable
- fully normalized
- stripped of all filesystem metadata

**NEW RULE (Micro-Improvement 3):**

**Runtime MUST NOT mutate a `RepoSnapshot` after construction.**  
`RepoSnapshot` MUST be treated as fully immutable until passed into the engine.

This protects:

- caching
- deterministic replays
- golden test stability
- Phase 2 multi-analyzer comparison

**Snapshot builder requirements:**

Runtime MUST ensure:

- normalized paths
- normalized newlines
- sorted files
- UTF-8 content where possible
- binary files treated consistently
- no symlinks
- no path escapes (`../`)
- no timestamps
- no permission bits
- no inode IDs
- no OS metadata leaks

**Engine MUST assume snapshot is already canonical.**

**Engine MUST NOT:**

- perform filesystem inspection
- interpret OS metadata
- resolve symlinks
- mutate snapshot

### 3.2 Analyzer Config

Runtime MAY pass config, but:

- config MUST be canonicalizable
- config MUST NOT include nondeterministic values
- config MUST not contain timestamps or environmental state

Engine computes `config_snapshot_hash` using `stableStringify`.

---

## 4. Engine Output Contract

The engine returns a deterministic `Envelope`:

```json
{
  "version",
  "identity",
  "core",
  "extensions",
  "meta"
}
```

Runtime MUST treat the `Envelope` as:

- immutable
- opaque
- final

Runtime MUST NOT:

- patch
- correct
- edit
- fix-up
- reorder
- merge
- drop fields

**NEW RULE (Micro-Improvement 1):**

**Runtime MUST NOT mutate, correct, or "fix up" any invalid, degraded, or error envelope.**  
Engine output MUST be treated as final.

**NEW RULE (Micro-Improvement 2):**

**If runtime-derived information conflicts with engine output, engine output is authoritative.**  
Runtime MUST NOT override engine conclusions under any circumstances.

This prevents future semantic drift.

---

## 5. Responsibilities Boundary

### 5.1 Engine Responsibilities (MUST)

Engine MUST:

- canonicalize snapshot
- build graph deterministically
- detect cycles deterministically
- generate core fields
- generate deterministic signature
- compute repo fingerprint
- enforce invariant rules
- degrade on limits/timeouts
- produce error envelopes deterministically
- apply schema adapters to produce `CurrentSchemaEnvelope`

Engine MUST NOT:

- fetch files
- call GitHub
- perform IO
- read environment variables
- use clocks or randomness
- access global mutable state
- read/write FS
- retry analysis

### 5.2 Runtime Responsibilities (MUST)

Runtime MUST:

- receive GitHub events
- dedupe events
- manage PR mutex
- fetch tarball
- enforce tarball security
- convert tarball → canonical snapshot
- construct identity chain
- invoke `analyze()` exactly once per analysis
- publish Check Run results
- handle degraded results faithfully
- enqueue DLQ items

Runtime MUST NOT:

- modify any `Envelope` (success, degraded, or error)
- override engine behavior
- construct cycles, graphs, or rulepack outputs
- interpret engine internals
- skip stages of canonicalization
- inject timestamps into envelopes or metadata
- mutate snapshots after creation
- add runtime-specific fields

**NEW RULE (Micro-Improvement 5):**

**Runtime MUST NOT implement fallback analyzers or alternative analyzers.**  
Only the primary engine runs in Phase 1.  
Multi-analyzer comparison belongs exclusively to Sentinel in Phase 2.

---

## 6. Dependency Contract

**Phase 1 Allowed Graph:**

```
runtime → engine
cli → engine
engine → (no external packages)
runtime ⨯ cli (never)
engine ⨯ runtime (never)
```

Runtime MUST NOT import engine internals.

Engine MUST NOT import runtime modules.

---

## 7. Error & Status Contract

### 7.1 Engine Errors: deterministic semantics

Engine may produce:

- success
- degraded
- error

**If degraded:**

- must include deterministic degraded reason
- runtime must pass through unchanged

**If error:**

- engine must still produce a full, structurally valid envelope

**NEW RULE (Micro-Improvement 4): Structural Validity Defined**

**Error envelopes MUST contain:**

- all required fields
- version block
- identity block
- full meta block
- `schema_version`
- deterministic `error_code`
- deterministic signature

Runtime MUST treat even error envelopes as authoritative.

### 7.2 Runtime Errors: operational

Runtime errors include:

- GitHub 403/429
- token refresh failures
- network errors
- tarball extraction violations
- mutex contention

Runtime MUST NOT attempt fallback analysis or partial analysis.

Runtime MUST NOT synthesize envelopes.

---

## 8. Identity Chain Contract

Runtime MUST fill all identity fields exactly.

Engine MUST copy identity into envelope without modification.

Identity MUST be preserved even for degraded/error cases.

---

## 9. Schema Versioning & Adapter Chain

Runtime MUST:

- treat `schema_version` as opaque
- never branch logic on `schema_version`
- always consume envelopes as-is

Engine MUST:

- apply adapter chain internally
- guarantee adapter continuity (no skipping versions)
- guarantee adapters preserve unknown extension fields
- never allow adapter-induced nondeterminism

---

## 10. Snapshot → Engine → Envelope Immutability

- Snapshot immutability
- Envelope immutability
- Identity chain immutability

These three invariants guarantee:

- deterministic caching
- reproducible analysis
- safe shadow analyzer comparisons
- stable golden tests
- future multi-tenant correctness

---

## 11. Runtime MUST NOT Inject Timestamps

Runtime MUST NOT include timestamps in:

- identity
- meta
- extensions
- error blocks
- config
- envelope-adjacent metadata

Timestamps introduce nondeterminism and must remain runtime-only.

---

## 12. Golden Test Implications

Golden tests validate:

- engine determinism
- snapshot → envelope stability
- schema adapter correctness
- degraded/error invariant behavior
- signature consistency

Runtime is not tested through golden tests.

Runtime is tested via webhook fixtures and mocks.

---

## 13. Summary (With Improvements)

This contract guarantees:

✔ Engine is pure, deterministic, isolated  
✔ Runtime is an IO host only  
✔ Engine outputs are final and authoritative  
✔ No fix-ups, no patches, no mutations in runtime  
✔ Snapshots are immutable  
✔ Error envelopes remain structurally valid  
✔ No fallback analyzers in runtime  
✔ No timestamps in deterministic pipeline  
✔ Schema adapters remain contained and deterministic  
✔ Identity is stable and untouched  
✔ Clear boundaries for Phase 2 multi-analyzer Sentinel rollout  

This contract protects ArcSight's determinism, correctness, and evolution for the next decade.

---

## Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Dependency Contract](./5A-dependency-contract.md)
- [Golden Test Governance](./09-golden-test-governance.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added Change Log section and standardized Cross-References heading.

**v1.0.0** — Initial version.

