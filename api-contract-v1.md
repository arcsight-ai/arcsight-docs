# ArcSight Wedge — API Contract (V1)

**Status:** Frozen  

**Authority:** Foundation Contract → Safety Wedge Contract → Types  

This document defines the **public and internal API** of ArcSight Wedge V1.

Only the **Public Entry Points** are considered part of the external API surface.  

All other functions are internal utilities and MAY change across versions, as long as contracts in this file remain valid.

Types referenced here are defined in `types.md`.

---

## 1. Public Entry Points (ONLY) - Single Entry Point Rule

ArcSight Wedge V1 exposes exactly **two** public entry points and NO MORE.

This is the **Single Entry Point Rule** - no additional public functions are allowed unless approved by ADR and SSOT update.

### 1.1 `analyzeCyclesForCommit`

```ts
function analyzeCyclesForCommit(
  repoPath: string
): Promise<{
  canonicalCycles: CanonicalCycle[];
  importGraph: ImportGraphList;
  confidence: number;
}>;
```

**Input:** `repoPath` — local path to the repository to analyze.

**Output:**

- `canonicalCycles`: all canonical cycles present at this commit (sorted).
- `importGraph`: resolved internal import graph (JS/TS only).
- `confidence`: confidence score for this snapshot.

This is a pure structural analysis with no PR semantics.

### 1.2 `analyzeCyclesForPR`

```ts
function analyzeCyclesForPR(
  baseSha: string,
  headSha: string,
  changedFiles: string[],
  repoPath: string
): Promise<PRCycleAnalysis>;
```

**Inputs:**

- `baseSha`: base commit SHA
- `headSha`: head commit SHA
- `changedFiles`: list of POSIX-normalized, lowercased file paths changed in the PR
- `repoPath`: local path to the repository

**Output:** `PRCycleAnalysis` (see `types.md`):

- `relevantCycles`: canonical cycles that are new in this PR and satisfy all V1 filters
- `rootCauses`: root-cause edges for those cycles
- `confidence`: repo confidence score for this analysis

This is the only function the PR bot should call.

---

## 2. Internal Utility Contracts (Non-Public)

These functions are internal but must respect their contracts to preserve determinism.

### 2.1 `diffCycles`

```ts
function diffCycles(
  base: CanonicalCycle[],
  head: CanonicalCycle[]
): {
  newCycles: CanonicalCycle[];
  removedCycles: CanonicalCycle[];
};
```

Both `base` and `head` MUST be sorted canonical cycle lists.

Output arrays MUST be sorted canonical lists.

### 2.2 `resolveAlias`

```ts
function resolveAlias(
  importPath: string,
  aliasMap: Record<string, string>
): string | null;
```

Returns a POSIX-normalized, lowercased resolved path, or `null` if unresolved.

MUST be deterministic.

If multiple resolutions are possible, callers MUST treat this as ambiguity and enter silent mode.

### 2.3 `computeConfidence` & `bucketConfidence`

```ts
function computeConfidence(q: SegmentationQuality): number;
// Returns a score in [0, 1]

type RepoBucket = "High" | "Low";

function bucketConfidence(confidence: number): RepoBucket;
```

**Inputs:** see `SegmentationQuality` in `types.md`.

Confidence ≥ 0.8 → "High", otherwise "Low".

### 2.4 `writeSnapshot`

```ts
function writeSnapshot(snapshot: CycleSnapshot): Promise<void>;
```

MUST append-only.

MUST NOT fail open; on failure, caller MUST treat as silent-error (no PR output impact in V1).

---

## 3. API Freeze

Only `analyzeCyclesForCommit` and `analyzeCyclesForPR` are public.

All other functions are internal but contractually constrained for determinism.

Any change to:

- function names
- parameter shapes
- output shapes

requires:

- RFC
- ADR
- SSOT & types update
