# Module Contract — src/analysis/cycles.ts (V1 FINAL LOCKED)

**Layer:** analysis

**Responsibility:** Deterministic cycle detection from normalized import graphs.

This module produces:

- A deterministic list of canonical cycles (`CanonicalCycle[]`)
- Raw cycle arrays for internal diagnostics
- Error detection flag for nondeterminism

This module MUST NOT:

- read files or parse imports
- interact with the filesystem
- inspect PR diffs or changedFiles
- compute root-cause edges
- detect god files, boundaries, forbidden imports, or any non-cycle signals
- infer architecture or module semantics
- modify ImportGraphList
- introduce heuristics or optimizations
- log anything
- maintain global state, caches, or cross-call memory

It is a pure cycle detection module.

---

## 1. Inputs & Outputs

### 1.1 Input (Internal API)

`importGraph: ImportGraphList`

Exactly as defined in `types.md`:

- `filePath`: repo-relative, normalized, lowercase
- `imports`: repo-relative, normalized, lowercase, sorted, deduped

`cycles.ts` MUST treat its input as:

- fully normalized
- structurally trusted
- but NOT semantically guaranteed (it MUST validate structure deterministically)

### 1.2 Output

```ts
interface CycleDetectionResult {
  canonicalCycles: CanonicalCycle[];
  rawCycles: string[][];
  errorDetected: boolean;
}
```

**canonicalCycles:**

- MUST contain ONLY canonicalized string cycles
- MUST be sorted alphabetically
- MUST be deduped after canonicalization
- MUST use `" → "` as the join separator

**rawCycles:**

```ts
rawCycles: string[][];
```

Each entry is an array of normalized paths representing one raw cycle.

**INTERNAL ONLY:** diagnostic use for determinism tests & snapshot inspection.

MUST NOT be used by the PR surface, V1 signals, or any user-facing output.

MAY be removed or replaced in V2+ without breaking the public API.

**errorDetected:**

- `true` ONLY when `cycles.ts` cannot guarantee deterministic correctness
- `false` when operating normally, INCLUDING "no cycles found" and "empty graph"

---

## 2. Graph Construction Rules (Deterministic)

`cycles.ts` MUST:

- Treat each unique `filePath` as a node
- Build adjacency lists from `imports` (directed edges)
- Sort ALL adjacency lists lexicographically
- Sort ALL node lists lexicographically
- Reject malformed nodes (missing `filePath`/`importArray` → silent mode)

**ImportGraph-order invariance (MANDATORY):**

`cycles.ts` MUST produce identical results regardless of the input order of `ImportGraphList` entries.

To guarantee this:

- Node list MUST be built from all `filePath` values, then sorted alphabetically.
- Adjacency lists for each node MUST be sorted alphabetically.

`cycles.ts` MUST NOT:

- rewrite file paths
- normalize paths again (`imports.ts` already guarantees normalization)
- remove disconnected nodes
- build reverse graphs or undirected graphs

---

## 3. Cycle Detection Algorithm (Deterministic)

### 3.1 Allowed Algorithms

**Tarjan's Strongly Connected Components (allowed):**

- MUST be implemented deterministically
- MUST treat SCCs of size ≥ 2 as cycles
- MUST represent SCC cycles explicitly (produce raw cycles)

**Deterministic DFS Cycle Enumeration (allowed):**

If DFS is used:

- MUST detect cycles by tracking recursion stack
- MUST avoid duplication by canonicalizing cycle node arrays
- MUST guarantee stable ordering across runs

### 3.2 Forbidden Algorithms

❌ Forbidden:

- heuristic cycle detection
- probabilistic cycle sampling
- partial enumeration
- cycle ranking or scoring
- order-dependent traversal
- graph pruning that affects determinism

---

## 4. Canonicalization Rules (MANDATORY)

**Global rule:**

All raw cycles MUST be represented as arrays with length ≥ 2.

Self-cycles MUST be represented with the starting node repeated at the end.

For each raw cycle array:

1. Ensure all paths are normalized (repo-relative, POSIX, lowercase)
2. Rotate cycle so lexicographically smallest node is first
3. Ensure cycle is listed in forward order only (no reversed variants)
4. Join nodes with `" → "`
5. Alphabetically sort the final list of canonical cycles
6. Deduplicate canonical cycles

---

## 5. Silent Mode Conditions (MANDATORY)

`cycles.ts` MUST return:

```ts
{
  canonicalCycles: [],
  rawCycles: [],
  errorDetected: true
}
```

ONLY if:

### 5.1 ImportGraphList is malformed

- missing `filePath`
- `filePath` not normalized
- `imports` not an array of normalized strings
- adjacency list inconsistent

**If ANY entry in `ImportGraphList` is malformed, `cycles.ts` MUST:**

- set `errorDetected = true`
- return `canonicalCycles = []` and `rawCycles = []`
- NOT attempt partial cycle detection on the remaining well-formed entries

### 5.2 Cycle detection algorithm violates determinism

Detected when:

- repeated runs produce mismatched results
- canonically rotated cycles disagree
- internal consistency check fails
- adjacency permutation changes output

### 5.3 Recursion depth exceeds deterministic threshold

To prevent stack blowouts or nondeterministic OS behavior.

### 5.4 Canonicalization fails

Any cycle that cannot be rotated, joined, or deterministically represented.

### 5.5 Unexpected internal exception

`cycles.ts` MUST catch and convert ALL internal exceptions to silent mode.

### 5.6 Valid Non-Error Cases (NOT Silent)

These MUST produce:

```ts
{
  canonicalCycles: [],
  rawCycles: [],
  errorDetected: false
}
```

- Empty graph (no nodes)
- Graph with nodes but no edges
- Graph with edges but no cycles
- Large graphs with deterministically detectable cycle absence

**Hard rule:**

`cycles.ts` MUST NOT set `errorDetected = true` solely because no cycles were found.

"Zero cycles" is a valid, non-error outcome.

Empty = Fine.  
Quiet = Fine.  
`errorDetected` must remain `false`.

---

## 6. False-Negative Preference Rules

`cycles.ts` MUST:

- Prefer missing cycles (false negatives) over nondeterminism
- Drop any cycle that cannot be deterministically represented
- Suppress output for cycles involving impossible-to-validate structures
- Never guess or infer missing edges

Producing no cycles is ALWAYS safer than producing a wrong cycle.

---

## 7. Internal Data Structure Constraints

`cycles.ts` MUST use:

- local Maps/Sets
- pure functions
- NO global or static variables
- NO memoization outside the function scope
- NO caching across calls
- NO retained context

`cycles.ts` MUST NOT:

- hold adjacency lists across runs
- maintain registry of visited nodes globally
- mutate input structures
- rely on iteration order of Maps/Sets

---

## 8. Edge Cases (Required Behaviors)

### 8.1 Self cycle

`A → A`

- MUST be detected as a valid cycle.
- MUST be represented as a path with at least 2 nodes:
  - e.g. `["src/a.ts", "src/a.ts"]`
- CanonicalCycle string becomes: `"src/a.ts → src/a.ts"`.

### 8.2 Binary cycle

`A → B → A`

- MUST be detected and canonicalized

### 8.3 Ternary cycle

`A → B → C → A`

- MUST be detected and canonicalized

### 8.4 Nested cycles inside SCCs

- Cycles MUST be deduped

### 8.5 Parallel edges

`A → B` (twice)

- MUST NOT produce duplicates

### 8.6 Disconnected subgraphs

- Only cycles within connected components are processed

### 8.7 Graphs with repeated imports

- `imports.ts` dedupes, `cycles.ts` trusts it

### 8.8 Thousands of nodes

- Algorithm MUST remain deterministic within performance constraints

---

## 9. Boundary Constraints

This module MUST NOT depend on:

- filesystem modules (`fs`)
- import parsing modules (`imports.ts` beyond receiving its output)
- diff modules (`cycleDiff.ts`, `rootCause.ts`)
- PR modules (`prHandler.ts`, `prSurface.ts`, `prIgnore.ts`)
- safety modules (`safetySwitch.ts`, `cooldown.ts`, `invariants.ts`, `errorPath.ts`)
- snapshot modules (`snapshotWriter.ts`)
- confidence module (`computeConfidence.ts`)

It MAY depend only on:

- `canonicalize.ts` (if separate canonicalization utilities exist)
- Standard TypeScript/JavaScript data structures

It MUST remain a pure cycle detection module with no PR/build/runtime semantics.

---

## 10. Summary

`cycles.ts` MUST:

✔ be deterministic  
✔ treat input `ImportGraphList` as normalized  
✔ detect cycles via SCC or deterministic DFS  
✔ sort everything alphabetically  
✔ canonicalize cycles identically every run  
✔ dedupe cycles  
✔ remain silent on ANY nondeterminism  
✔ return `{}` with `errorDetected: true` ONLY on true errors  
✔ prefer false negatives  
✔ avoid all PR semantics  
✔ avoid architecture inference  
✔ avoid external signals  
✔ be pure, functional, stateless  
✔ avoid logging, caching, side effects  

`cycles.ts` MUST NOT:

❌ compute root-cause  
❌ examine diff  
❌ inspect PR changes  
❌ interpret alias ambiguity  
❌ modify `ImportGraphList`  
❌ reorder nodes beyond deterministic sorting  
❌ introduce new V1 signals  
❌ break module boundaries  

This module is the foundation upon which:

- cycle diffing
- root-cause detection
- PR safety

all depend.

