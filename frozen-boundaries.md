# Frozen Boundaries — Phase 1–3.3

**Version:** 1.0  
**Status:** FROZEN  
**Effective Date:** Phase 3.4

---

## Purpose

This document defines the frozen boundaries for Phase 1–3.3 of the ArcSight Wedge engine. Any changes to frozen components **require a version bump** and explicit approval.

---

## Frozen Components

### 1. Engine Pipeline (`src/engine/runEngine.ts`)

**Status:** FROZEN

**Frozen Elements:**
- Execution order: Canonicalize → Build Graph → Run Analyzers → Merge Violations → Apply Waivers → Build Summary
- Analyzer execution order: DriftAnalyzer → CycleAnalyzer → ChangeImpactAnalyzer
- Engine result structure (`EngineResult` interface)
- Summary calculation logic
- Repository fingerprint computation

**Prohibited Changes:**
- Reordering analyzer execution
- Modifying engine result structure without version bump
- Changing summary calculation without version bump
- Altering fingerprint computation algorithm

---

### 2. Analyzers (`src/core/analyzers/`)

**Status:** FROZEN

**Frozen Elements:**
- `DriftAnalyzer` — drift detection logic
- `CycleAnalyzer` — cycle detection logic
- `ChangeImpactAnalyzer` — change impact analysis logic
- Analyzer input/output contracts (`AnalyzerInput`, `AnalyzerOutput`)
- Analyzer version numbers (`ANALYZER_VERSIONS`)

**Prohibited Changes:**
- Modifying analyzer detection logic without version bump
- Changing analyzer input/output contracts
- Altering analyzer version numbers
- Adding new analyzers without Phase 4+ approval

---

### 3. CLI Adapter (`src/tools/arcsightCli.ts`)

**Status:** FROZEN

**Frozen Elements:**
- JSON output format (when `--json` flag is used)
- Exit codes: 1 (rulepack validation), 2 (engine execution), 3 (determinism violation), 4 (unexpected error)
- Command structure: `check`, `replay`, `stats`
- CI mode output format

**Prohibited Changes:**
- Modifying JSON output schema without version bump
- Changing exit codes
- Altering command structure
- Modifying CI mode output format

---

### 4. GitHub PR Adapter (`src/pr/`)

**Status:** FROZEN

**Frozen Elements:**
- PR comment format (`buildPRComment` output)
- PR comment fingerprint: `<!-- arcsight:pr-comment:v1:cycle-only -->`
- PR handler lifecycle (`handlePullRequest`)
- Ignore request handling (`checkIgnoreRequest`)

**Prohibited Changes:**
- Modifying PR comment markdown format without version bump
- Changing fingerprint marker
- Altering PR handler lifecycle
- Modifying ignore request logic

---

### 5. Waiver Contracts (`src/core/waivers/`)

**Status:** FROZEN

**Frozen Elements:**
- Waiver contract structure (`Waiver` interface)
- Waiver validation logic (`validateWaivers`)
- Waiver application logic (`applyWaivers`)
- Waiver extension data structure

**Prohibited Changes:**
- Modifying waiver contract structure without version bump
- Changing waiver validation rules
- Altering waiver application logic
- Modifying extension data structure

---

### 6. Risk Model

**Status:** FROZEN

**Frozen Elements:**
- Risk scoring calculation (if implemented)
- Severity levels: `error`, `warning`, `info`
- Category classification

**Prohibited Changes:**
- Modifying risk scoring algorithm without version bump
- Changing severity level definitions
- Altering category classification

---

## Prohibited Areas

### Engine Mutation
- **DO NOT** mutate engine state across runs
- **DO NOT** maintain global state in engine modules
- **DO NOT** cache results in engine modules

### Analyzer Mutation
- **DO NOT** mutate analyzer input snapshots
- **DO NOT** maintain state in analyzer functions
- **DO NOT** modify analyzer contracts without version bump

### Snapshot Mutation
- **DO NOT** mutate repository snapshots during processing
- **DO NOT** modify snapshot structure without version bump
- **DO NOT** alter canonicalization logic

### Phase 4+ Features
- **DO NOT** add Phase 4/virality/adoption features to frozen components
- **DO NOT** modify frozen contracts for new features
- **DO NOT** break backward compatibility

---

## Version Bump Requirements

Any change to frozen components **MUST** include:

1. **Version bump** in relevant version files:
   - `src/versions/schemaVersion.ts` (for schema changes)
   - `src/versions/analyzerVersion.ts` (for analyzer changes)
   - `src/versions/rulepackVersions.ts` (for rulepack changes)

2. **Contract documentation** update:
   - Update relevant contract documents in the docs-submodule
   - Document breaking changes
   - Provide migration guide if needed

3. **Test updates**:
   - Update golden tests
   - Add backward compatibility tests
   - Verify determinism is preserved

4. **Approval process**:
   - Explicit approval for version bump
   - Review of breaking changes
   - Verification of backward compatibility

---

## Enforcement

- **Automated tests** verify frozen boundaries are not violated
- **Integration audit** (`tests/integration/e2e-audit.test.ts`) checks for mutations
- **Contract validation** ensures schema compliance
- **Determinism tests** verify identical outputs across runs

---

## Exceptions

**No exceptions** are allowed without explicit approval and version bump.

---

## Related Documents

- `determinism-rules.md` — Determinism requirements
- `philosophy.md` — Design philosophy
- `phase3.1-waiver-audit.md` — Waiver system audit
- Contract documents in `module-contracts/`

---

**Last Updated:** Phase 3.4  
**Next Review:** Phase 4 planning

