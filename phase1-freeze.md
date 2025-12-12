# Phase 1 Freeze — Wedge Core Engine

**Status:** ❄️ FROZEN  
**Release Version:** v1.0.0-phase1  
**Freeze Date:** 2025-01-27  
**Tag:** `v1.0.0-phase1`

---

## Purpose

This document establishes the Phase 1 freeze boundary for the ArcSight Wedge engine. All components listed as **❄️ Frozen** are immutable and MUST NOT be changed without a schema or analyzer version bump. This freeze ensures deterministic behavior, contract compliance, and provides a stable foundation for Phase 2+ development.

---

## Freeze Status Table

| Area | Status | Notes |
|------|--------|-------|
| **Analyzer interface** | ❄️ Frozen | `Analyzer`, `AnalyzerInput`, `AnalyzerOutput`, `AnalyzerConfig` types are immutable |
| **Envelope schema** | ❄️ Frozen | `EnvelopeSchema`, `PRSummary` structure locked |
| **Snapshot schema** | ❄️ Frozen | `RepoSnapshot` contract immutable |
| **Violation schema** | ❄️ Frozen | `Violation` interface, ID generation, sorting rules locked |
| **Rulepack schema** | ❄️ Frozen | `Rulepack`, `RulepackAnalyzer` structure immutable |
| **Canonicalization rules** | ❄️ Frozen | All sorting, hashing, normalization algorithms locked |
| **Analyzer ordering** | ❄️ Frozen | Execution order: `DriftAnalyzer` → `CycleAnalyzer` → `ChangeImpactAnalyzer` |
| **Analyzer logic** | 🔒 Only with version bump | Analyzer implementations may change only with version increment |
| **Engine pipeline** | ❄️ Frozen | `runEngine.ts` orchestration logic locked |
| **Rulepacks** | 🚀 Phase 2+ | External rulepack support deferred |
| **CLI UX** | 🚀 Phase 2+ | Enhanced CLI features deferred |
| **GitHub UX** | 🚀 Phase 2+ | GitHub App enhancements deferred |
| **Golden test harness** | ❄️ Frozen | Test fixtures and expected outputs locked |

---

## Red Line Enforcement

> **Any change that alters golden test output without a schema or analyzer version bump is considered a bug.**

This means:

- ✅ **Allowed:** Adding new analyzers with new versions
- ✅ **Allowed:** Extending schemas with optional fields (backward compatible)
- ✅ **Allowed:** Bug fixes that preserve output determinism
- ❌ **Forbidden:** Changing analyzer execution order
- ❌ **Forbidden:** Modifying violation ID generation algorithm
- ❌ **Forbidden:** Altering canonicalization rules
- ❌ **Forbidden:** Changing envelope structure
- ❌ **Forbidden:** Breaking golden test expectations without version bump

---

## Phase 1 Components

### Analyzers

| Analyzer | Version | Status |
|----------|---------|--------|
| `DriftAnalyzer` | `0.1.0` | ❄️ Frozen |
| `CycleAnalyzer` | `0.1.0` | ❄️ Frozen |
| `ChangeImpactAnalyzer` | `0.1.0` | ❄️ Frozen |

### Schema Versions

| Schema | Version | Status |
|--------|---------|--------|
| Analyzer Version | `1.0.0` | ❄️ Frozen |
| Schema Version | `1.0.0` | ❄️ Frozen |
| Violation Schema | `v1` | ❄️ Frozen |
| Envelope Schema | `v1` | ❄️ Frozen |
| Rulepack Schema | `v1` | ❄️ Frozen |

### Engine Pipeline

- **Execution Order:** Deterministic, fixed order
- **Canonicalization:** All outputs canonicalized before envelope construction
- **Determinism:** Byte-for-byte identical outputs across runs
- **Contract Compliance:** All outputs conform to Contract Layer Spec v1

---

## Golden Test Lock

The golden test harness (`tests/engine/`) provides the behavioral lock for Phase 1:

- **Fixtures:** `tiny-snapshot.json`, `medium-snapshot.json`, `weird-snapshot.json`
- **Expected Outputs:** Versioned in `tests/engine/expected/v1/`
- **Test Coverage:** 69 tests, all passing
- **Determinism:** Verified byte-for-byte across multiple runs

Any change that breaks golden tests MUST be accompanied by:
1. Schema version bump, OR
2. Analyzer version bump, OR
3. Explicit approval via RFC/ADR

---

## Cross-Platform Determinism

Phase 1 outputs are verified to be deterministic across:
- ✅ macOS
- ✅ Linux (CI)
- ✅ Path normalization (Windows-style `\` vs POSIX `/`)
- ✅ Import ordering
- ✅ JSON key ordering
- ✅ Violation ordering

---

## Phase 2+ Scope

The following areas are explicitly **NOT** frozen and may be developed in Phase 2+:

- External rulepack support
- Enhanced CLI UX (colors, progress bars, interactive mode)
- GitHub App UI/UX improvements
- Advanced analyzer heuristics
- Performance optimizations (within determinism constraints)
- Additional analyzers (with new versions)

---

## Enforcement

### Pre-Commit Checks

Before committing changes to Phase 1 frozen components:

1. Run golden test suite: `npm test -- tests/engine/`
2. Verify determinism: Run engine twice, compare outputs byte-for-byte
3. Check contract compliance: All outputs must match Contract Layer Spec v1
4. Validate analyzer ordering: Ensure execution order unchanged

### CI Enforcement

CI MUST:
- Run all golden tests
- Verify determinism across multiple runs
- Check analyzer ordering
- Validate contract compliance
- Fail on any golden test drift without version bump

---

## Fingerprint

See `docs/phase1-fingerprint.txt` for the deterministic Phase 1 fingerprint hash.

---

## References

- [Contract Layer Spec v1](../../docs-submodule/21-contract-layer-spec-v1.md)
- [Foundation Contract](./00-foundation-contract.md)
- [Determinism Requirements](./determinism-requirements.md)
- [Golden Test Harness](../tests/engine/README.md)

---

**Last Updated:** 2025-01-27  
**Maintainer:** ArcSight Wedge Team

