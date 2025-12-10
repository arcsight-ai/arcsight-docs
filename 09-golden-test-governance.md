# Golden Test Governance (Final Version)

## ArcSight Phase-1 & Phase-2 Deterministic Golden Suite Rules

Golden tests are ArcSight's determinism anchor.

They guarantee that the engine's observable behavior evolves intentionally, predictably, and safely, across years and multiple analyzer generations.

This document governs:

- how golden tests are created
- how and when they may be updated
- how drift is detected
- how analyzer evolution interacts with golden outputs
- how Phase-2 shadow analyzers and sentinel rely on goldens

It is binding across:

- `arcsight-wedge` (engine)
- `arcsight-cli` (golden runner)
- `arcsight-github-app` (consumer only; must not depend on goldens)
- Phase-2 packages: sentinel, dashboard, rulepacks, arc-shared

---

## 1. Purpose of Golden Tests

Golden tests verify that:

**For the same `RepoSnapshot` + config + `analyzerVersion` → ArcSight produces the exact same `Envelope`.**

They ensure:

- deterministic engine behavior
- stable evolution
- regressions cannot hide
- schema evolution remains backward-compatible
- rulepack behavior remains predictable

Golden tests are the **executable specification** of the analyzer.

---

## 2. What Goldens ARE vs. Are NOT

✔ **Goldens ARE:**

- deterministic snapshots of expected engine output
- the canonical truth for analyzer behavior
- a guard against accidental drift
- the stability contract across analyzer versions

✘ **Goldens are NOT:**

- feature tests
- debugging output
- giant real-world repos
- arbitrary integration tests
- a place to store unrelated fixture data

Goldens exist solely to enforce determinism and intentional analyzer evolution.

---

## 3. Golden Repository Requirements

Golden repos MUST be:

- minimal, surgical, small
- designed to isolate specific deterministic behaviors
- <100 KB whenever possible
- OS-agnostic
- consistent across environments

**Rule:** Goldens must remain minimal and non-redundant.

If a smaller repo reproduces the behavior → use the smaller version.

This prevents golden bloat and keeps deterministic comparisons meaningful.

---

## 4. Golden Envelope Requirements

Golden envelopes MUST:

- be produced via the full canonical engine pipeline
- include deterministic signature
- be normalized using:
  - canonical paths
  - normalized newlines
  - Unicode NFC
  - sorted structures
  - `stableStringify`
- be upgraded to the current schema via the adapter chain before comparison

Golden envelopes MUST NOT contain:

- timestamps
- OS metadata
- nondeterministic fields
- machine paths or environment values

---

## 5. When Goldens MUST Be Updated

A golden update is required only when:

- analyzer semantics intentionally change
- cycle detection logic evolves
- canonicalization rules change
- schema structure changes (schema bump)
- a rulepack introduces new deterministic output

### 🚨 CRITICAL RULE — Analyzer Version Bump Required

**Golden tests must NEVER be updated without bumping `analyzerVersion`.**

This prevents "golden leakage," where regressions are accidentally blessed into correctness.

**This rule is absolute.**

---

## 6. Golden Test Update Protocol

Whenever goldens must be updated:

1. Update analyzer or rulepack logic.
2. Update golden repo(s).
3. Update expected envelopes.
4. Bump:
   - `analyzerVersion`
   - rulepack versions (if needed)
   - `schemaVersion` (only if structure changed)
5. Update schema adapters if structure changed.
6. Regenerate goldens via `arcsight-cli`.
7. Run cross-platform determinism checks.
8. Include before/after diffs in PR.
9. Explain semantic reason for change.

**Reviewers MUST verify:**

- diffs match intended semantic behavior
- no unrelated drift exists
- signature change is consistent with envelope changes
- `config_snapshot_hash` and `repo_fingerprint` change only when expected

---

## 7. Golden Suite Coverage Expansion (Rulepacks)

When introducing new deterministic output (e.g., new rulepacks):

- Add new golden repos if needed
- Follow minimality rules
- Include expected envelopes
- Update test harness to include new rulepack coverage

Extensions that often require goldens:

- drift detection
- hotspots
- boundaries / architectural rules
- enterprise rulepacks

**Rule:** If a new deterministic output exists → it MUST be covered by a golden.

---

## 8. Golden Drift Detection (Phase 1 + Phase 2)

Drift detection MUST validate:

- full envelope equality
- deterministic cycle ordering
- `graph_stats` stability
- no unexpected changes in node or edge count
- signature stability
- no ordering drift in arrays or objects

If drift is detected:

- either fix nondeterminism OR
- confirm intentional semantic change and bump `analyzerVersion`

**Drift is always a bug unless explicitly approved.**

---

## 9. Golden Fingerprint Drift (Phase 2 Activation)

Phase 2 introduces a drift fingerprint per golden repo:

- `repo_fingerprint`
- `cycle_count`
- `graph_stats`
- `rulepack-output-hash`

The Sentinel MUST verify:

- drift only occurs for valid semantic changes
- unexpected fingerprint changes block analyzer promotion

This enables safe shadow analyzer rollouts.

---

## 10. Golden Test Execution Pipeline

`arcsight-cli` MUST:

1. Build `RepoSnapshot` via canonical builder.
2. Run engine `analyze(snapshot, config)`.
3. Apply schema adapters → `CurrentSchemaEnvelope`.
4. Load expected golden envelope.
5. Compare envelopes with:
   - deep structural comparison
   - `stableStringify`
   - deterministic cycle ordering checks
   - deterministic signature comparison

**CI MUST run this pipeline across OSes.**

---

## 11. Prohibited Patterns

**Forbidden:**

- updating goldens to "make tests pass"
- updating goldens without `analyzerVersion` bump
- mixing multiple analyzer changes into one golden update
- using goldens for debugging
- large repos or unrelated fixtures
- relying on goldens in runtime or production code

---

## 12. Golden Repository Structure (Phase 1)

```
arcsight-wedge/
  tests/
    golden/
      repos/
        small-repo/
        medium-repo/
        weird-repo/
      expected/
        small-repo.json
        medium-repo.json
        weird-repo.json
```

Repos → behaviors  
Expected → envelopes

---

## 13. Golden Determinism Rules

Golden suite MUST enforce:

- strict cycle ordering invariants
- sorted adjacency and file lists
- deterministic `graph_stats`
- canonical JSON via `stableStringify`
- `deepSort` for all nested structures
- deterministic signature generation
- no object key iteration drift
- no random ordering

**Goldens are the deterministic oracle.**

---

## 14. CI + Reviewer Checklist

**Reviewer MUST verify:**

- `analyzerVersion` bumped
- golden diffs match semantic intent
- no unrelated fields changed
- ordering & cycle sets stable
- schema adapters updated if needed
- no nondeterministic drift
- `repo_fingerprint` stable unless expected

**CI MUST verify:**

- goldens match exactly
- repeat-run determinism
- cross-platform determinism
- schema-normalized comparison
- no drift fingerprints (Phase 2)

---

## 15. The Golden Stability Window (NEW — Required)

To prevent accidental golden mutation within a PR:

**A PR may update golden tests only once per `analyzerVersion` bump.**

After goldens have been updated in a PR:

- no further commits in that PR may modify golden outputs
- any additional changes require a new PR and a new `analyzerVersion` bump

This ensures:

- clean, auditable analyzer evolution
- clear semantic mapping: one bump → one behavioral change
- drift cannot creep in through iterative commits
- shadow analyzers compare against stable baselines

This rule mirrors how Google Tricorder, Meta Infer, and Stripe SCA enforce analyzer stability.

---

## 16. Ownership

- Engine team owns the golden suite
- CLI team owns golden runner
- Runtime MUST NOT depend on goldens
- Sentinel (Phase 2) uses goldens for promotion gating

---

## 17. Summary

Golden tests enforce:

- deterministic engine behavior
- intentional evolution
- backward compatibility
- drift detection
- safety across analyzer versions
- correctness of schema and rulepacks
- safe shadow analyzer rollouts

**Golden tests are ArcSight's single most important correctness mechanism, and this governance guarantees stability for a decade.**

---

## Cross-References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Dependency Contract](./5A-dependency-contract.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added Change Log section and standardized Cross-References heading.

**v1.0.0** — Initial version.

