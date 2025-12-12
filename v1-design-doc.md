# ArcSight Wedge — V1 Design Document

**Status:** Frozen (V1)  

**Scope:** Cycle detection only  

**Authority:** Foundation Contract + Safety Wedge Contract + API Contract  

---

# 1. Overview

ArcSight Wedge V1 is a deterministic, zero-noise PR safety engine that identifies **new dependency cycles** introduced in a pull request. The design is intentionally minimal, safe, and contract-first.

---

# 2. Problem Statement

JS/TS codebases frequently accumulate hidden cycles that slow development, break bundlers, block refactors, and increase cognitive burden. These issues are not detected by tests, linters, or human review.

ArcSight provides a **single structural safety signal** to prevent accidental architectural decay.

---

# 3. Requirements (From Contracts)

## MUST

- Detect cycles sized 2–5  

- Only show cycles involving PR-diff files  

- Only show cycles introduced by the PR  

- Detect diff-based root-cause import edge  

- Maintain full determinism  

- Stay silent unless confidence ≥0.8

## MUST NOT

- Use AST parsers  

- Infer architecture  

- Analyze boundaries  

- Log anything  

- Emit additional signals  

- Support monorepos  

---

# 4. System Architecture

Each module is pure, deterministic, isolated, and adheres to strict boundaries.

```
analysis ─→ diff ─→ pr
│         │      │
├─ alias  │      │
├─ cycles │      │
└─ canonicalize
```

- analysis: builds import graph, resolves aliases, finds cycles  

- diff: compares base/head cycles & root cause  

- confidence: repo quality evaluation only  

- safety: silent mode, error gating  

- pr: render/update a single PR comment  

- snapshot: write cycle metadata  

---

# 5. Data Flow (PR Evaluation)

1. Checkout base + head  

2. Run `analyzeCyclesForCommit` for both  

3. Diff cycle sets  

4. Filter cycles by PR-changed files  

5. Detect root-cause import edges  

6. Compute confidence  

7. If confidence <0.8 → silent  

8. If any uncertainty → silent  

9. Render or update a single PR comment  

---

# 6. Root Cause Detection Algorithm (Mandatory)

For each NEW canonical cycle detected in `head` but not in `base`, ArcSight MUST attempt to identify a **single root-cause import edge** responsible for closing that cycle.

The algorithm is:

1. **Build the set of edges for the cycle**  
   - For a cycle with nodes `[f1, f2, ..., fn]`, edges are:
     - `(f1 → f2), (f2 → f3), ..., (fn-1 → fn), (fn → f1)`

2. **Restrict to PR-changed files**  
   - Only consider edges where `from` is in the `changedFiles` set.

3. **Map PR diff to import edges**  
   - For each changed file:
     - Parse the diff hunks.
     - Collect lines added in the PR.
     - From those added lines, extract **new import statements only**.
   - Map each new import line to a candidate edge `(fromFile → resolvedTargetFile)` using the same import resolution pipeline as the import graph.

4. **Candidate root-cause edges**  
   - For a given cycle, candidate root-cause edges are those edges that:
     - belong to the cycle's edge set, AND
     - originate from a file in `changedFiles`, AND
     - are derived from a new import line in the diff.

5. **Edge selection priority (deterministic sorting required)**  
   If multiple candidate edges exist for the same cycle:
   1. Sort candidate edges lexicographically by `from` path.
   2. If still multiple: sort by `to` path.
   3. If still multiple: prefer edges from files closest to repo root (shortest path depth).
   4. Select the first edge after sorting (deterministic selection).

6. **No root-cause edge → cycle discarded for V1**  
   - If **no edge** in the cycle can be matched to a new import line in the PR diff:
     - the cycle is treated as **non-attributable** for V1, and
     - is excluded from `relevantCycles`.

7. **Renames / moves**  
   - For V1, file renames or moves are treated as:
     - "changed files" with new imports in the **new path only**.
   - If a cycle appears only due to rename without new imports:
     - it is treated as **non-attributable** and excluded.

This ensures:

- Every surfaced cycle has a concrete "this line did it" explanation.

- Cycles without a clear root-cause import are safely ignored in V1.

---

# 7. Key Design Decisions

## Decision 1: No AST Parsing  

Avoids nondeterminism, complexity, runtime variability.

## Decision 2: No Logging  

Production Guide & Foundation Contract prohibit any console output.

## Decision 3: False Negatives Allowed  

Missing a cycle is acceptable; emitting incorrect warnings is not.

## Decision 4: Alias Ambiguity → Silent  

Alias maps come from config; ambiguity collapses to silence.

## Decision 5: Canonicalization Required  

Stable, reproducible cycle representation ensures trust.

## **Decision 6 (NEW): Deterministic Filesystem Traversal**  

ArcSight MUST:

- explicitly sort all file and directory listings  

- normalize all paths to POSIX format  

- lowercase all paths  

- enforce lexicographic order before any recurrence or iteration  

Node's `fs.readdir` CANNOT be trusted to return stable ordering.  

Sorting is required at every FS boundary.

Failure to sort → nondeterminism → safety violation.

---

# 8. Confidence Scoring & Segmentation Quality

## 8.1 Segmentation Quality Semantics

`SegmentationQuality` is defined in `types.md` and interpreted as:

- `fileCount`: total JS/TS source files considered after applying ignore rules.
- `analyzedFileCount`: number of files for which import analysis succeeded.
- `analyzedFileCoverage`: `analyzedFileCount / fileCount`. Values <0.6 indicate poor coverage.
- `aliasStatus`:
  - `"ok"` → all aliases resolved deterministically.
  - `"uncertain"` → any alias ambiguity detected.
- `isMonorepo`:
  - true if monorepo heuristics (see Monorepo Detection) match.
- `importGraphStable`:
  - true if import graph is identical across repeated runs for this repo.
- `unresolvedImportRatio`:
  - number of unresolved imports / total imports.

`computeConfidence` MUST use only these fields and MUST:

- return 0 immediately if:
  - `fileCount < 10`, OR
  - `aliasStatus === "uncertain"`, OR
  - `isMonorepo === true`, OR
  - `importGraphStable === false`.
- otherwise compute a score in [0,1] using a documented formula in code comments.

**V1 recommended formula (for implementation):**

```
confidence =
  0.4 * analyzedFileCoverage +
  0.3 * (1 - unresolvedImportRatio) +
  0.3 * (importGraphStable ? 1 : 0)
```

**Threshold:**

- confidence ≥ 0.8 → High
- otherwise → Low (silent)

## 8.2 Monorepo Detection (Heuristic, Deterministic)

In V1, monorepos are forcibly treated as Low confidence and MUST be silent.

A repo is considered a **monorepo** if ANY of the following deterministic checks succeed:

1. Presence of a top-level `pnpm-workspace.yaml` OR `yarn.workspaces` in `package.json`.
2. Presence of BOTH:
   - a top-level `packages/` or `apps/` directory, AND
   - at least two subdirectories each containing their own `package.json`.

If these conditions are met:

- `SegmentationQuality.isMonorepo` MUST be set to `true`.
- `computeConfidence` MUST return 0.
- The wedge MUST remain silent for this repo in V1.

**Edge cases:**

- Single-app repos with `/src` and `/test` MUST NOT be treated as monorepos, even if they contain a `packages/` directory name incidentally.

## 8.3 Source File Counting Rules (JS/TS Only)

For the purposes of `fileCount` and `analyzedFileCount`:

**Included extensions:**

- `.ts`
- `.tsx`
- `.js`
- `.jsx`

**Excluded directories (hard-coded):**

- `**/node_modules/**`
- `**/.next/**`
- `**/dist/**`
- `**/build/**`
- `**/coverage/**`
- `**/vendor/**`
- `**/generated/**`
- `**/__generated__/**`
- `**/__tests__/**`
- `**/tests/**`

**Excluded files:**

- `*.d.ts` (type declarations only)

**Counting rules:**

- `fileCount` = number of files matching **included extensions** minus **excluded directories/files**.
- `analyzedFileCount` = subset of `fileCount` for which import analysis completed successfully.

---

# 9. Performance Model

SLO:

- p95 < 3 seconds  

- p99 < 7 seconds  

SLA:

- If runtime > 7 seconds → analysis treated as uncertain → silent  

Memory target: <500MB  

No caching required in V1.

---

# 10. Failure Modes & Error Handling

Silent mode MUST trigger on:

- alias ambiguity  

- incomplete import graph  

- inconsistent cycle detection  

- root-cause detection failure  

- canonicalization mismatch  

- monorepo detected  

- repo <10 files  

- performance exceeding SLA  

- repeated-run nondeterminism  

ArcSight MUST fail closed, never open.

---

# 11. Security Considerations

ArcSight MUST:

- never execute user code  

- never modify repositories  

- never transmit data  

- never log anything  

---

# 12. Determinism Requirements

All modules must:

- avoid randomness  

- avoid Date/time  

- avoid stateful dependencies  

- avoid concurrency hazards  

- canonicalize all outputs  

- sort all lists  

- normalize all paths  

**Deterministic Sorting Requirements:**

- Root-cause candidate edges MUST be sorted lexicographically before selection.

- Final PR cycles list MUST be sorted alphabetically (canonical cycle strings).

- Diff ordering MUST be deterministic: compare canonical cycles lexicographically.

- All file lists, import lists, and alias map keys MUST be sorted before processing.

**CanonicalCycle Path Invariant:**

- CanonicalCycle MUST contain only repo-relative normalized paths (POSIX, lowercased).

- CanonicalCycle MUST NOT contain relative paths (e.g., `../` or `./`).

- All paths in a CanonicalCycle MUST be absolute repo-relative paths from the repository root.

These rules are enforced through deterministic test suites.

---

# 13. Future-Proofing (Frozen for V1)

No expansion beyond cycles is allowed until:

- wedge validated  

- design partner confirms correctness  

- ADRs propose expansion  

V1 design is immutable.

