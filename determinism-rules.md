# Determinism Rules

**Version:** 1.0  
**Status:** MANDATORY  
**Effective Date:** Phase 1

---

## Purpose

This document defines the determinism rules for the ArcSight Wedge engine. All code **MUST** follow these rules to ensure byte-for-byte identical outputs across runs.

---

## Core Principle

**Identical inputs → Identical outputs**

Given the same repository snapshot, rulepack, and evaluation date, the engine **MUST** produce byte-for-byte identical outputs across all runs, regardless of:
- System time
- Environment variables
- Operating system
- Execution order (where deterministic)
- Random number generation

---

## Allowed Operations

### 1. Sorting

**Allowed:**
- Explicit sorting of arrays before output
- Deterministic sort functions (e.g., `localeCompare`, numeric comparison)
- Stable sort algorithms

**Example:**
```typescript
// ✅ Allowed: Explicit sorting
const sorted = [...array].sort((a, b) => a.id.localeCompare(b.id));

// ✅ Allowed: Stable sort
const sorted = [...array].sort((a, b) => {
  if (a.date < b.date) return -1;
  if (a.date > b.date) return 1;
  return a.id.localeCompare(b.id);
});
```

### 2. Canonicalization

**Allowed:**
- Path normalization (POSIX separators, case-preserved)
- String normalization (newlines, whitespace)
- Hash computation (SHA-256, deterministic)
- Deep sorting of objects/arrays

**Example:**
```typescript
// ✅ Allowed: Path normalization
const normalized = path.replace(/\\/g, '/');

// ✅ Allowed: Hash computation
const hash = canonicalHash(content);

// ✅ Allowed: Deep sorting
const sorted = deepSort(object);
```

### 3. Evaluation Date Injection

**Allowed:**
- Injecting evaluation date as parameter (ISO 8601 YYYY-MM-DD)
- Using injected date for expiry calculations
- Computing days between dates deterministically

**Rule:** All functions that rely on dates **MUST** accept `evaluationDate: string` and **MUST NOT** compute the current system date internally.

**Example:**
```typescript
// ✅ Allowed: Date injection
function computeWaiverMetrics(waivers: Waiver[], evaluationDate: string) {
  // Use evaluationDate, not Date.now()
}

// ✅ Allowed: TypeScript interface pattern
interface DateDependentFunction {
  (data: Waiver[], evaluationDate: string): WaiverResult;
}

// ❌ Forbidden: Internal date computation
function computeWaiverMetrics(waivers: Waiver[]) {
  const today = new Date().toISOString().split('T')[0]; // ❌ Forbidden
  // ...
}
```

---

## Forbidden Operations

### 1. System Time Access

**Forbidden:**
- `Date.now()`
- `new Date()` (without explicit input)
- `process.hrtime()`
- `performance.now()`
- Any time-based API

**Example:**
```typescript
// ❌ Forbidden: System time
const now = Date.now();

// ❌ Forbidden: Current date
const today = new Date().toISOString().split('T')[0];

// ✅ Allowed: Injected date
function process(waivers: Waiver[], evaluationDate: string) {
  const today = evaluationDate; // Use injected date
}
```

### 2. Random Number Generation

**Forbidden:**
- `Math.random()`
- `crypto.randomBytes()`
- Any random number generator

**Example:**
```typescript
// ❌ Forbidden: Random numbers
const id = Math.random().toString(36);

// ✅ Allowed: Deterministic ID
const id = canonicalHash(content);
```

### 3. Environment Reads

**Forbidden:**
- `process.env.*`
- `os.hostname()`
- `os.platform()`
- `os.userInfo()`
- Any environment-dependent API

**Example:**
```typescript
// ❌ Forbidden: Environment reads
const hostname = os.hostname();

// ❌ Forbidden: Environment variables
const mode = process.env.NODE_ENV;
```

### 4. Non-Canonical Iteration

**Forbidden:**
- Iterating over `Object.keys()` without sorting
- Iterating over `Map`/`Set` without sorting
- Relying on insertion order

**Example:**
```typescript
// ❌ Forbidden: Unsorted iteration
for (const key of Object.keys(obj)) {
  // Order is not guaranteed
}

// ✅ Allowed: Sorted iteration
const sortedKeys = Object.keys(obj).sort();
for (const key of sortedKeys) {
  // Order is deterministic
}
```

### 5. Timestamps in Output

**Forbidden:**
- Including timestamps in JSON output
- Including timestamps in PR comments
- Including timestamps in CLI output

**Example:**
```typescript
// ❌ Forbidden: Timestamps in output
const output = {
  timestamp: new Date().toISOString(),
  data: result,
};

// ✅ Allowed: No timestamps
const output = {
  data: result,
};
```

---

## Sorting Guidance

### Violations

**Rule:** Sort violations by `id` (lexicographic)

```typescript
const sorted = violations.sort((a, b) => a.id.localeCompare(b.id));
```

### Extension Data

**Rule:** Sort all arrays and objects in extension data

```typescript
const sorted = deepSort(extensionData);
```

### Waivers

**Rule:** Sort waivers by:
1. `expiryDate` (ascending)
2. `ruleId` (lexicographic)
3. `filePath` (undefined last, then lexicographic)

```typescript
const sorted = waivers.sort((a, b) => {
  if (a.expiryDate < b.expiryDate) return -1;
  if (a.expiryDate > b.expiryDate) return 1;
  if (a.ruleId < b.ruleId) return -1;
  if (a.ruleId > b.ruleId) return 1;
  // filePath handling...
});
```

### Categories

**Rule:** Sort category keys before output

```typescript
const sortedKeys = Object.keys(categories).sort();
const sortedCategories: Record<string, number> = {};
for (const key of sortedKeys) {
  sortedCategories[key] = categories[key];
}
```

---

## Golden Test Enforcement

### Purpose

Golden tests ensure deterministic outputs by:
1. Running the same code 3× times
2. Comparing outputs byte-for-byte
3. Failing if any difference is detected

### Implementation

```typescript
it('should produce identical outputs', () => {
  const results: string[] = [];
  
  for (let i = 0; i < 3; i++) {
    const result = runEngine(snapshot, rulepack);
    results.push(stableStringify(result));
  }
  
  expect(results[0]).toBe(results[1]);
  expect(results[1]).toBe(results[2]);
});
```

### CI Integration

**CI systems MUST verify determinism automatically:**

1. **Run engine multiple times** and compare outputs:
   ```bash
   # Verify engine determinism
   node -e "const {runEngine} = require('./dist/engine/runEngine'); 
            const r1 = runEngine(snapshot, rulepack);
            const r2 = runEngine(snapshot, rulepack);
            if (JSON.stringify(r1) !== JSON.stringify(r2)) process.exit(1);"
   ```

2. **Run CLI adapter multiple times** and compare JSON outputs:
   ```bash
   # Verify CLI determinism
   arcsight check . --json > output1.json
   arcsight check . --json > output2.json
   diff output1.json output2.json || exit 1
   ```

3. **Run PR adapter multiple times** and compare markdown outputs:
   ```bash
   # Verify PR comment determinism
   node -e "const {buildPRComment} = require('./dist/pr/prSurface');
            const c1 = buildPRComment(cycles, fingerprint);
            const c2 = buildPRComment(cycles, fingerprint);
            if (c1 !== c2) process.exit(1);"
   ```

4. **Use golden test harness** (3× repeated runs):
   ```typescript
   // All tests should verify 3× runs produce identical outputs
   for (let i = 0; i < 3; i++) {
     const result = runEngine(snapshot, rulepack);
     results.push(stableStringify(result));
   }
   expect(results[0]).toBe(results[1]);
   expect(results[1]).toBe(results[2]);
   ```

### Requirements

- **All** engine outputs must pass golden tests
- **All** CLI JSON outputs must pass golden tests
- **All** PR comment outputs must pass golden tests
- **All** waiver outputs must pass golden tests
- **CI must run determinism checks** on every commit

---

## Verification

### Automated Checks

- **Determinism tests** (`tests/waivers/waiver-determinism.test.ts`)
- **Integration audit** (`tests/integration/e2e-audit.test.ts`)
- **Golden tests** (3× repeated runs)

### Manual Checks

- Review code for forbidden operations
- Verify sorting is explicit
- Check for timestamps in output
- Validate date injection usage

---

## Common Pitfalls

### 1. Implicit Date Usage

```typescript
// ❌ Wrong: Implicit date
function isExpired(expiryDate: string): boolean {
  return expiryDate < new Date().toISOString().split('T')[0];
}

// ✅ Correct: Injected date
function isExpired(expiryDate: string, evaluationDate: string): boolean {
  return expiryDate < evaluationDate;
}
```

### 2. Unsorted Iteration

```typescript
// ❌ Wrong: Unsorted
for (const [key, value] of Object.entries(obj)) {
  // Order not guaranteed
}

// ✅ Correct: Sorted
const sortedEntries = Object.entries(obj).sort(([a], [b]) => a.localeCompare(b));
for (const [key, value] of sortedEntries) {
  // Order guaranteed
}
```

### 3. Random IDs

```typescript
// ❌ Wrong: Random
const id = Math.random().toString(36);

// ✅ Correct: Deterministic
const id = canonicalHash(content);
```

---

## Phase 4+ Future-Proofing

**Important:** Any new outputs added in Phase 4+ (viral hooks, badges, PR footers, etc.) **MUST** conform to these same determinism rules.

### Rules for New Outputs

1. **No timestamps** in any new output format
2. **No random values** in any new output format
3. **Explicit sorting** of all arrays/objects
4. **Evaluation date injection** for any date-dependent logic
5. **Golden tests required** for all new output formats

### Example: Adding a PR Footer

```typescript
// ✅ Correct: Deterministic footer
function buildPRFooter(metadata: PRMetadata): string {
  // No timestamps
  // No random values
  // Sorted metadata
  const sortedMetadata = deepSort(metadata);
  return `---\nMetadata: ${stableStringify(sortedMetadata)}`;
}

// ❌ Wrong: Non-deterministic footer
function buildPRFooter(): string {
  return `---\nGenerated: ${new Date().toISOString()}`; // ❌ Timestamp
}
```

**Prevents "accidental drift"** when new sections are added in Phase 4+.

---

## Related Documents

- `docs/frozen-boundaries.md` — Frozen component boundaries
- `docs/philosophy.md` — Design philosophy
- `docs/phase3.1-waiver-audit.md` — Waiver system audit

**Note:** After migration to docs-submodule, update links to reflect submodule paths (e.g., `../frozen-boundaries.md` or `docs-submodule/frozen-boundaries.md` when accessed from wedge).

---

**Last Updated:** Phase 3.4  
**Enforcement:** Mandatory

