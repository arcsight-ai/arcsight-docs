# ArcSight Wedge — Core Types (V1)

**Status:** Frozen  

**Authority:** Foundation Contract → Safety Wedge Contract → API Contract → Determinism Requirements  

This document defines ALL shared types used across:

- `api-contract-v1.md`

- `v1-design-doc.md`

- implementation modules under `src/`

No other shared types are allowed in V1 without ADR.

---

## 1. CanonicalCycle

```ts
export type CanonicalCycle = string;

// Example: "src/auth/index.ts → src/session/store.ts → src/auth/index.ts"
```

A CanonicalCycle is a fully normalized, deterministic string representation of a dependency cycle:

- All paths POSIX-normalized (`/`)
- All paths lowercased
- Rotated so the lexicographically smallest path is first
- The first node is repeated at the end to emphasize closure (optional but consistent)
- Nodes joined with `" → "`

---

## 2. ImportGraph

```ts
export interface ImportGraph {
  /** Absolute or repo-relative, POSIX-normalized, lowercased file path */
  filePath: string;
  /** List of absolute or repo-relative, POSIX-normalized, lowercased file paths this file imports */
  imports: string[];
}

export type ImportGraphList = ImportGraph[];
```

**Notes:**

- Only `.ts`, `.tsx`, `.js`, `.jsx` files appear in ImportGraph.
- Type-only imports do not create edges.
- Files in ignored directories (`node_modules`, `dist`, `build`, `.next`, `tests`, etc.) NEVER appear here.

---

## 3. RootCauseEdge

```ts
export interface RootCauseEdge {
  /** POSIX-normalized, lowercased path of the importing file */
  from: string;
  /** POSIX-normalized, lowercased path of the imported file */
  to: string;
  /** Optional: 1-based line number of the import causing the edge (if available) */
  lineNumber?: number;
  /** Optional: original import line text (trimmed) */
  importLine?: string;
  /** The canonical cycle this edge is associated with */
  canonicalCycle: CanonicalCycle;
}
```

A RootCauseEdge is the specific import edge, coming from a line in the PR diff, that completes a cycle.

If no such edge can be identified with certainty, no RootCauseEdge is emitted for that cycle and the cycle is excluded from V1 output.

---

## 4. SegmentationQuality

```ts
export type AliasStatus = "ok" | "uncertain";

export interface SegmentationQuality {
  /** Total number of JS/TS source files considered */
  fileCount: number;
  /** Number of JS/TS files that were successfully analyzed for imports */
  analyzedFileCount: number;
  /** Ratio (0..1) of analyzed files to total files */
  analyzedFileCoverage: number;
  /** Whether alias resolution was unambiguous and deterministic */
  aliasStatus: AliasStatus;
  /** Heuristic monorepo detection flag */
  isMonorepo: boolean;
  /** True if import graph is stable across repeated runs for this repo */
  importGraphStable: boolean;
  /**
   * Ratio (0..1) of imports that could NOT be resolved to a local JS/TS file.
   * 0 = all imports resolved, 1 = none resolved.
   */
  unresolvedImportRatio: number;
}
```

These fields are the only inputs to `computeConfidence`.

---

## 5. CycleSnapshot

```ts
export interface CycleSnapshot {
  /** Repo identifier (string chosen by caller) */
  repoId: string;
  /** Commit SHA or unique identifier for this snapshot */
  commitSha: string;
  /** ISO-8601 UTC timestamp (second precision) */
  timestamp: string;
  /** Canonical cycles at this commit, sorted lexicographically */
  canonicalCycles: CanonicalCycle[];
  /** Confidence score in [0, 1] */
  confidence: number;
}
```

Used ONLY for append-only storage.

Snapshots are not interpreted by the wedge in V1 (no drift, no trends).

---

## 6. PRCycleAnalysis

```ts
export interface PRCycleAnalysis {
  /** Canonical cycles that are new in this PR and pass all filters */
  relevantCycles: CanonicalCycle[];
  /** Root-cause edges for relevant cycles */
  rootCauses: RootCauseEdge[];
  /** Confidence score in [0, 1] for this repo at this commit */
  confidence: number;
}
```

This is the internal structured result that the PR layer uses to decide whether to comment and how to format the message.

Only `analyzeCyclesForCommit` and `analyzeCyclesForPR` are public entry points in V1. All other types are shared but not public APIs.

