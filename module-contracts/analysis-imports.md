# Module Contract — src/analysis/imports.ts (V1 FINAL LOCKED)

**Layer:** analysis

**Responsibility:** Deterministic static import graph extraction for JS/TS repos.

This module produces:

- A deterministic ImportGraphList
- File-level analysis statistics (used by the confidence module)

This module MUST NOT:

- compute cycles
- compute PR semantics
- compute confidence
- detect monorepos
- modify global state
- log anything
- infer architecture

It is a pure analysis module.

---

## 1. Inputs & Outputs

### 1.1 Input

`repoPath: string`

Repo path MUST be treated as read-only.

### 1.2 Output

```ts
interface ImportAnalysisResult {
  importGraph: ImportGraphList;
  fileStats: {
    fileCount: number;
    analyzedFileCount: number;
    totalImportCount: number;
    unresolvedImportCount: number;
    unreadableFileCount: number;
    aliasAmbiguityDetected: boolean;
  };
}
```

**Zero-import files rule (MANDATORY):**

For every JS/TS source file that:

- is included by the extension/ignore rules, AND
- is successfully read and decoded as UTF-8, AND
- is considered "analyzed" (even if no valid imports are found),

there MUST be exactly one ImportGraph entry:

```ts
{ filePath: "<normalized-path>", imports: [] }
```

No analyzed file may be omitted from importGraph.

All types are defined in `types.md`. No deviations allowed.

---

## 2. Filesystem Traversal Requirements

### 2.1 Deterministic Traversal

ALL directory reads MUST:

- Collect entries.
- Sort lexicographically (ASCII, lowercase).
- Traverse in sorted order only.

No reliance on OS filesystem ordering is allowed.

### 2.2 Source File Inclusion / Exclusion

**Included extensions:**

- `.ts`
- `.tsx`
- `.js`
- `.jsx`

**Excluded directories (hard-coded):**

- `**/node_modules/`
- `**/.next/`
- `**/dist/`
- `**/build/`
- `**/coverage/`
- `**/vendor/`
- `**/generated/`
- `**/__generated__/`
- `**/__tests__/`
- `**/tests/`

**Excluded files:**

- `*.d.ts`

**File size rule (performance guard, deterministic):**

If a file exceeds 2MB:

- Increment `unreadableFileCount++`
- DO NOT attempt import parsing
- DO NOT add it to importGraph
- MUST NOT throw

### 2.3 Symlink Handling (Mandatory)

- MUST use `fs.realpathSync()` to resolve physical paths.
- MUST maintain a `Set<string>` of absolute realpaths visited.
- If a realpath has already been seen:
  - skip that file/directory (prevents infinite recursive loops).

---

## 3. Path Normalization (Mandatory)

For all paths (file paths and resolved imports):

1. Resolve symlink → realpath.
2. Compute repo-relative path from `repoPath`.
3. Convert `\` → `/`.
4. Lowercase entire string.

This normalized form is the canonical identity and MUST be used for:

- keys in maps
- entries in ImportGraphList
- import edges

No comparisons or sets may be based on unnormalized paths.

---

## 4. Text & Encoding Normalization (Mandatory)

Before parsing import statements, each source file MUST:

- Be read as raw bytes.
- Decode as UTF-8 ONLY:
  - If decoding fails OR invalid byte sequences are detected:
    - Increment `unreadableFileCount++`
    - DO NOT attempt partial parsing
    - DO NOT crash or throw
    - DO NOT add this file to importGraph
- Strip UTF-8 BOM (U+FEFF) if present at the beginning.
- Normalize line endings:
  - Convert all `\r\n` to `\n`.

All regex parsing MUST operate on this normalized UTF-8, BOM-stripped, LF-only string.

---

## 5. Import Extraction Rules (MANDATORY)

### 5.1 Regex Execution Order (Deterministic)

When scanning a file, regex patterns MUST be applied in this exact sequence:

1. ES Module import patterns
2. CommonJS require patterns
3. Side-effect import `'...'` patterns

This ordering MUST be fixed so that:

- `totalImportCount`
- `unresolvedImportCount`
- raw match iteration

remain deterministic across machines and versions.

### 5.2 Allowed Import Patterns (MUST detect)

Regex MUST detect imports where the string literal appears:

- on the same line, OR
- within up to 2 subsequent lines (multi-line allowance, deterministic)

**Valid ES Module Patterns:**

- `import X from '...';`
- `import { A, B } from "...";`
- `import * as Foo from '...';`
- `import X, { Y } from '...';`
- `import '...';` (side-effect import)

**Valid CommonJS Patterns:**

- `const x = require('...');`
- `var x = require('...');`
- `let x = require('...');`

**Valid Multi-line Examples (MUST support):**

```ts
import {
  A,
  B,
  C
} from './foo';

import A
  from
'./bar';

const {
  a,
  b
} = require('./baz');
```

**Multi-line import allowance (MANDATORY):**

After matching an import or require keyword, the module MUST search for the corresponding string literal on:

- the same line, OR
- line +1 OR
- line +2

If no string literal is found within 2 lines → treat as invalid import and ignore it.

**Comments MUST NOT count as imports:**

```ts
// import foo from './x';
/*
  import foo from './x';
*/
```

### 5.3 Forbidden / Ignored Patterns

These MUST NOT be treated as imports and MUST NOT affect `totalImportCount`:

- **Dynamic imports:**
  - `import('...')`
- **Template require:**
  - `require(${x}/foo)`
- **Conditional imports:**
  - `if (x) require('./foo')`
- **Imports inside non-top-level scopes** that don't match the allowed literal-within-2-lines pattern.

V1 is allowed to produce false negatives in these cases; this is acceptable and preferred over nondeterminism.

### 5.4 Type-only Imports — MUST Ignore

**Examples:**

```ts
import type { Foo } from './foo';
import { type Foo } from './foo';
```

These MUST:

- NOT produce edges
- NOT increment `totalImportCount`
- Count only toward `analyzedFileCount` (because file was successfully parsed)

---

## 6. Import Classification

### 6.1 Relative Imports (`./` or `../`)

For imports whose raw specifier starts with `./` or `../`:

- Build a base path (without extension).
- Apply extension inference in the following strict order:

Given base path X, try:

1. `X.ts`
2. `X.tsx`
3. `X.js`
4. `X.jsx`
5. `X/index.ts`
6. `X/index.tsx`
7. `X/index.js`
8. `X/index.jsx`

The first path that exists is the resolved path.

**If none exist:**

- Increment `unresolvedImportCount++`
- Do not add an edge to `ImportGraph.imports`.

**If resolved:**

- Normalize the resolved file path (Section 3).
- Add to that file's imports list (deduped later).

### 6.2 Alias Imports (only if aliasResolver is provided)

Given an alias import (non-relative) AND a configured alias map:

Call:

```ts
resolveAlias(importPath, aliasMap): string | null
```

**If `resolveAlias` returns a deterministic single resolved path:**

- Treat that as a relative-like path and apply the same extension inference.

**If `resolveAlias` returns null:**

- Increment `unresolvedImportCount++`

**If alias resolution is ambiguous (per aliasResolver semantics):**

- Set `aliasAmbiguityDetected = true`
- Increment `unresolvedImportCount++`
- Do NOT add an edge for this import.

**If alias resolver is NOT configured, all alias-style imports MUST be treated as unresolved (no edges, `unresolvedImportCount++`).**

### 6.3 Bare / External Imports

**Examples:**

```ts
import React from 'react';
import '@org/lib';
const x = require('fs');
```

For these:

- Increment `totalImportCount++`
- Do NOT:
  - add edges
  - mark unresolved
  - attempt path resolution

They are treated as external or runtime dependencies, not part of the local import graph.

---

## 7. Deterministic Ordering & Duplicate Removal (MANDATORY)

For each file:

1. Collect raw imports in source order.
2. Normalize all resolved imports as canonical repo-relative paths (Section 3).
3. Deduplicate using set semantics.
4. Sort the final list alphabetically.

`ImportGraph.imports` MUST always be:

- alphabetically sorted
- deduplicated
- normalized

**File-level import determinism rule:**

Even if a file contains:

```ts
import x from './foo';
import y from './foo';
const z = require('./foo');
```

And all three resolve to `src/foo.ts`, the final imports MUST be:

```ts
imports: ["src/foo.ts"]  // exactly once
```

**Top-level importGraph ordering rule (MANDATORY):**

`importGraph` MUST be sorted alphabetically by `filePath`.

Each `filePath` MUST appear at most once in `importGraph`.

This guarantees:

- Deterministic top-level ordering
- No duplicates at file level
- No need for callers to sort/dedupe `importGraph` further

---

## 8. Error Handling (Silent-Mode Rules)

### 8.1 File read / decode errors

If a file:

- cannot be read, OR
- cannot be decoded as UTF-8,

Then:

- Increment `unreadableFileCount++`
- Do NOT include this file in `importGraph`
- Do NOT increment `analyzedFileCount`
- MUST NOT crash or throw

### 8.2 Import parse failure

If file is readable/decodable but regex parsing fails:

- Count this file as `analyzedFileCount++`
- Treat as having zero imports
- MUST NOT count parsing errors as unresolved imports
- MUST NOT throw

### 8.3 Repo-level IO failure

If `repoPath` is invalid or unreadable:

- This module MAY throw a top-level error to the caller (e.g., "repoPath not found").
- It MUST NOT:
  - log
  - partially mutate global state

Higher layers (safety/errorPath) will handle this and enforce silent mode at PR level.

---

## 9. Soft Cap on Import-like Matches (Optional, Deterministic)

For performance, implementation MAY apply a fixed soft cap on import-like matches per file, e.g.:

```ts
MAX_IMPORT_MATCHES_PER_FILE = 500  // constant, hard-coded
```

**Soft cap semantics (MANDATORY if used):**

Once the cap is reached for a file:

- regex scanning MUST stop for that file
- additional potential matches MUST be treated as if they do not exist:
  - MUST NOT increment `totalImportCount`
  - MUST NOT increment `unresolvedImportCount`
  - MUST NOT create any import edges

**The cap value:**

- MUST be a fixed constant in this module
- MUST NOT depend on environment variables, config, or runtime settings
- MUST be applied consistently on every run, across all environments

This preserves determinism and keeps the behavior in "false negatives" territory, never "half-counted" territory.

---

## 10. Boundary Constraints

This module MUST NOT depend on:

- cycle detection modules (`cycles.ts`)
- diff modules (`cycleDiff.ts`, `rootCause.ts`)
- PR modules (`prHandler.ts`, `prSurface.ts`, `prIgnore.ts`)
- safety modules (`safetySwitch.ts`, `cooldown.ts`, `invariants.ts`, `errorPath.ts`)
- snapshot modules (`snapshotWriter.ts`)
- confidence module (`computeConfidence.ts`)

It MAY depend only on:

- alias resolution module (`aliasResolver.ts`), as allowed by module boundaries.

It MUST remain a pure analysis module with no PR/build/runtime semantics.

---

## 11. Edge Cases (Required Behaviors)

### 11.1 Empty repo / no source files

If there are no included JS/TS files:

```ts
importGraph = [];
fileStats = {
  fileCount: 0,
  analyzedFileCount: 0,
  totalImportCount: 0,
  unresolvedImportCount: 0,
  unreadableFileCount: 0,
  aliasAmbiguityDetected: false,
};
```

### 11.2 Extremely large files

If file size > 2MB:

- `unreadableFileCount++`
- MUST NOT attempt to parse imports

### 11.3 Mixed import syntaxes

Files containing both ES module and CJS patterns MUST be supported.

### 11.4 Multiple import patterns pointing to same file

All resolved to the same normalized path, final imports list MUST contain that path exactly once.

### 11.5 BOM or CRLF files

- BOM stripped
- CRLF normalized to LF
- Regex behavior MUST be identical across OSes.

### 11.6 Multi-line imports beyond 2 lines

If the import string literal is not found within 2 lines of the import or require keyword:

- treat as invalid
- do NOT create an edge
- do NOT increment `totalImportCount`

### 11.7 Circular symlinks

If realpath of a file or directory has already been visited:

- skip it
- do NOT recurse further

---

## 12. Summary

`imports.ts` MUST:

✔ deterministically walk the repo  
✔ apply strict include/exclude filters  
✔ resolve symlinks safely and idempotently  
✔ normalize all paths identically  
✔ normalize encoding (UTF-8) and line endings (LF)  
✔ support multi-line imports with up to 2-line lookahead  
✔ parse only explicitly allowed patterns  
✔ ignore forbidden/dynamic/import-like patterns  
✔ perform deterministic extension inference  
✔ sort & dedupe imports  
✔ include every analyzed file in importGraph (even zero-import files)  
✔ sort importGraph by filePath  
✔ track unresolved and unreadable files  
✔ track alias ambiguity  
✔ remain silent on all errors  
✔ produce identical output for identical repo input  

This module is the foundation upon which:

- cycle detection
- cycle diffing
- root-cause detection
- PR safety

all depend.
