# Module Contract — src/safety/safetySwitch.ts (V1 FINAL LOCKED)

**Layer:** safety

**Responsibility:** Deterministic safety switch evaluation to enforce silent mode on anomalies, nondeterminism, or uncertainty.

This module produces:

- A deterministic boolean indicating whether to enter silent mode
- Safety gate evaluation based on analysis quality and determinism checks

This module MUST NOT:

- inspect cycle structure
- access PR logic
- modify outputs
- detect cycles or perform diffing
- compute confidence
- log anything
- infer architecture
- add new signals

It is a pure evaluation module.

---

## 1. Module Purpose & Responsibilities

**Location:** `safety/`

**Scope:** This module operates on analysis context and quality metrics.

**Function:** Evaluates analysis results and quality indicators to determine if safety switch should activate, forcing silent mode.

**This module is the ONLY place where safety switch logic exists.**

**Boundaries:**

- Does NOT inspect cycle structure (delegated to analysis modules)
- Does NOT access PR logic (delegated to PR modules)
- Does NOT modify outputs (read-only evaluation)
- Does NOT detect cycles or perform diffing
- Does NOT compute confidence (delegated to confidence module)
- MUST evaluate determinism and consistency
- MUST enforce silent mode on any anomaly
- MUST be a pure function

---

## 2. Inputs & Outputs

### 2.1 Input

```ts
interface SafetySwitchContext {
  determinism: {
    repeatedRunConsistent: boolean;
    canonicalizationStable: boolean;
    importGraphStable: boolean;
  };
  performance: {
    runtimeSeconds: number;
  };
  quality: {
    aliasAmbiguityDetected: boolean;
    importGraphComplete: boolean;
    rootCauseDetectionStable: boolean;
  };
  errors: {
    analysisFailed: boolean;
    canonicalizationFailed: boolean;
    rootCauseDetectionFailed: boolean;
  };
}
```

**Input assumptions:**

- `determinism`: Determinism check results
  - `repeatedRunConsistent`: `true` if repeated runs produce identical results, `false` otherwise
  - `canonicalizationStable`: `true` if canonicalization is stable, `false` if inconsistent
  - `importGraphStable`: `true` if import graph is stable across runs, `false` otherwise
- `performance`: Performance metrics
  - `runtimeSeconds`: Analysis runtime in seconds (number)
- `quality`: Quality indicators
  - `aliasAmbiguityDetected`: `true` if alias resolution is ambiguous, `false` if unambiguous
  - `importGraphComplete`: `true` if import graph is complete, `false` if incomplete
  - `rootCauseDetectionStable`: `true` if root-cause detection is stable, `false` if unstable
- `errors`: Error indicators
  - `analysisFailed`: `true` if analysis failed, `false` otherwise
  - `canonicalizationFailed`: `true` if canonicalization failed, `false` otherwise
  - `rootCauseDetectionFailed`: `true` if root-cause detection failed, `false` otherwise

**Input validation:**

- If `context` is missing or `null`/`undefined` → return `true` (silent mode, fail-safe)
- If any required field is missing or invalid type → return `true` (silent mode)
- If context structure is malformed → return `true` (silent mode)

### 2.2 Output

**Function signature:**

```ts
function shouldSilence(context: SafetySwitchContext): boolean
```

**Return type:** `boolean`

**Output semantics:**

- `true`: Safety switch active → enter silent mode (do not create/update PR comment)
- `false`: Safety switch inactive → proceed with normal PR comment lifecycle

**Output rules:**

- `true` is the safe default (prefer silence over incorrect output)
- `false` only when ALL safety conditions are met
- Must be deterministic (same context → same result)

---

## 3. Safety Switch Trigger Conditions

**Safety switch MUST activate (return `true`) when ANY of the following are true:**

1. **Determinism fails:**
   - `determinism.repeatedRunConsistent === false`
   - `determinism.canonicalizationStable === false`
   - `determinism.importGraphStable === false`

2. **Performance budget exceeded:**
   - `performance.runtimeSeconds > 7.0` (SLA violation)
   - Threshold is exclusive: `> 7.0` (not `>= 7.0`)

3. **Quality indicators negative:**
   - `quality.aliasAmbiguityDetected === true`
   - `quality.importGraphComplete === false`
   - `quality.rootCauseDetectionStable === false`

4. **Errors detected:**
   - `errors.analysisFailed === true`
   - `errors.canonicalizationFailed === true`
   - `errors.rootCauseDetectionFailed === true`

5. **Context validation failure:**
   - Context is missing, null, or undefined
   - Required fields are missing or invalid types
   - Context structure is malformed

**Safety switch MUST NOT activate (return `false`) when:**
- `determinism.repeatedRunConsistent === true`
- `determinism.canonicalizationStable === true`
- `determinism.importGraphStable === true`
- `performance.runtimeSeconds <= 7.0`
- `quality.aliasAmbiguityDetected === false`
- `quality.importGraphComplete === true`
- `quality.rootCauseDetectionStable === true`
- `errors.analysisFailed === false`
- `errors.canonicalizationFailed === false`
- `errors.rootCauseDetectionFailed === false`
- Context is valid and complete

**Evaluation algorithm:**

1. Validate context structure (return `true` if invalid)
2. Check determinism flags (return `true` if any `false`)
3. Check performance threshold (return `true` if `runtimeSeconds > 7.0`)
4. Check quality indicators (return `true` if any negative)
5. Check error flags (return `true` if any `true`)
6. If all checks pass → return `false` (proceed)

---

## 4. Deterministic Behavior Rules

`safetySwitch.ts` MUST:

- Produce identical output for identical input context
- Use deterministic comparison logic (exact equality, numeric comparisons)
- Apply trigger conditions deterministically (fixed evaluation order)
- Never use timestamps or dynamic content in decision logic
- Never depend on environment or runtime state

`safetySwitch.ts` MUST NOT:

- Use `Date.now()`, `Math.random()`, or other nondeterministic APIs
- Depend on iteration order of Maps/Sets
- Vary behavior based on environment variables
- Use nondeterministic comparison operations
- Perform repeated runs internally (receives pre-computed results)

**Evaluation order (deterministic):**

1. Context validation
2. Determinism checks
3. Performance check
4. Quality indicators
5. Error flags

This order MUST be fixed to ensure deterministic evaluation.

---

## 5. Silent Mode Rules

`safetySwitch.ts` MUST return `true` (silent mode) when:

- ANY trigger condition is detected
- Context is malformed or invalid
- Required fields are missing
- Any uncertainty is present
- ANY boolean flag indicates failure
- Performance threshold is exceeded

**Silent mode behavior:**

- Return `true` immediately on any trigger (short-circuit evaluation)
- Do not attempt partial evaluation
- Do not log errors
- Do not modify context
- Prefer false positives (err on side of silence)

`safetySwitch.ts` MUST return `false` (proceed) when:

- ALL trigger conditions are absent
- ALL quality indicators are positive
- Context is valid and complete
- No anomalies detected
- All checks pass

---

## 6. False-Positive Preference Rules

`safetySwitch.ts` MUST:

- Prefer activating safety switch (silence) over allowing potentially incorrect output
- NEVER allow output when uncertain
- Treat ANY ambiguity as a trigger
- Activate on partial failures or inconsistencies
- Return `true` when context is invalid or incomplete

**If evaluation is ambiguous:**

- Return `true` (silent mode)
- Do not attempt to "fix" or interpret context
- Prefer false positive (incorrect silence) over false negative (incorrect output)

**Fail-safe principle:**

- When in doubt, silence
- Invalid context → silence
- Missing data → silence
- Ambiguous indicators → silence

---

## 7. Forbidden Behaviors

`safetySwitch.ts` MUST NOT:

- Inspect cycle structure (delegated to analysis modules)
- Access PR logic (delegated to PR modules)
- Modify outputs (read-only evaluation)
- Detect cycles or perform diffing
- Compute confidence (delegated to confidence module)
- Log anything (no console output, no debug statements)
- Infer architecture
- Add new signals
- Use timestamps or dynamic content
- Depend on environment variables
- Throw exceptions (return boolean instead)
- Use nondeterministic APIs
- Access filesystem or external services
- Perform repeated runs internally (receives pre-computed results)
- Measure performance internally (receives runtime as input)
- Compute determinism checks internally (receives results as input)

---

## 8. Edge Cases

### 8.1 Context missing or null

**Input:** `context` is `null`, `undefined`, or missing

**Output:** `true` (silent mode)

**Behavior:** Invalid context → fail-safe silence

### 8.2 All quality indicators positive

**Input:** All checks pass, no anomalies

**Output:** `false` (proceed)

**Behavior:** Safety switch inactive, allow PR comment

### 8.3 Partial determinism failure

**Input:** Some determinism checks pass, some fail

**Output:** `true` (silent mode)

**Behavior:** Any failure triggers safety switch

### 8.4 Performance exactly at threshold

**Input:** `runtimeSeconds === 7.0` (exactly at threshold)

**Output:** `false` (proceed)

**Behavior:** Threshold is exclusive (`> 7.0`), so exactly 7.0 does not trigger

### 8.5 Performance just above threshold

**Input:** `runtimeSeconds === 7.0001` (just above threshold)

**Output:** `true` (silent mode)

**Behavior:** Exceeds threshold, triggers safety switch

### 8.6 Multiple trigger conditions

**Input:** Multiple conditions trigger simultaneously

**Output:** `true` (silent mode)

**Behavior:** Any trigger activates safety switch (short-circuit evaluation)

### 8.7 Missing required fields

**Input:** Context missing `determinism`, `performance`, `quality`, or `errors` fields

**Output:** `true` (silent mode)

**Behavior:** Invalid context structure → fail-safe silence

### 8.8 Invalid field types

**Input:** Fields have wrong types (e.g., `runtimeSeconds` is string instead of number)

**Output:** `true` (silent mode)

**Behavior:** Invalid types → fail-safe silence

### 8.9 All boolean flags false/true

**Input:** All failure flags are `false`, all success flags are `true`

**Output:** `false` (proceed)

**Behavior:** All checks pass, safety switch inactive

### 8.10 Single trigger condition

**Input:** Only one trigger condition is true, all others pass

**Output:** `true` (silent mode)

**Behavior:** Any single trigger activates safety switch

---

## 9. Boundary Constraints

This module MUST depend on:

- Type definitions (if shared types are needed)

This module MUST NOT depend on:

- cycle detection modules (`analysis/cycles.ts`)
- diff modules (`diff/cycleDiff.ts`, `diff/rootCause.ts`)
- PR modules (`pr/prHandler.ts`, `pr/prSurface.ts`, `pr/prIgnore.ts`)
- confidence modules (`confidence/computeConfidence.ts`)
- snapshot modules (`snapshot/snapshotWriter.ts`)
- analysis modules directly (`analysis/imports.ts`, etc.)

This module MUST remain a pure evaluation layer.

---

## 10. Summary Checklist

`safetySwitch.ts` MUST:

✔ accept `SafetySwitchContext` input structure  
✔ validate context structure deterministically  
✔ check determinism flags (repeated-run, canonicalization, import graph)  
✔ check performance threshold (`runtimeSeconds > 7.0`)  
✔ check quality indicators (alias ambiguity, graph completeness, root-cause stability)  
✔ check error flags (analysis, canonicalization, root-cause failures)  
✔ return `true` on ANY trigger condition  
✔ return `false` only when ALL checks pass  
✔ use deterministic evaluation order  
✔ prefer false positives (err on side of silence)  
✔ treat invalid context as trigger  
✔ never throw exceptions  
✔ produce identical output for identical inputs  
✔ use deterministic comparison operations only  
✔ never log anything  
✔ never access filesystem or external services  
✔ never modify context  
✔ never inspect cycle structure  
✔ never access PR logic  

`safetySwitch.ts` MUST NOT:

❌ inspect cycle structure  
❌ access PR logic  
❌ modify outputs  
❌ detect cycles or perform diffing  
❌ compute confidence  
❌ log or print anything  
❌ infer architecture  
❌ add new signals  
❌ use timestamps or dynamic content  
❌ depend on environment variables  
❌ throw exceptions  
❌ use nondeterministic APIs  
❌ perform repeated runs internally  
❌ measure performance internally  
❌ compute determinism checks internally  

---

## 11. Authority

This contract is authoritative over all implementation of `src/safety/safetySwitch.ts`.

It is subordinate to:

- Foundation Contract
- Safety Wedge Contract
- API Contract
- Module Boundaries
- Types (`types.md`)
- Determinism Requirements
- ADRs

Any deviation from this contract MUST be approved via ADR and SSOT update.

