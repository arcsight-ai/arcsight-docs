# ArcSight Wedge — Testing Philosophy (V1)

**Status:** Frozen for V1  

**Authority:** Foundation Contract + Safety Wedge Contract + Determinism Requirements  

**Scope:** Ensures ArcSight Wedge is fully deterministic, silent on uncertainty, and incapable of producing false positives.

This document defines HOW ArcSight Wedge must be tested.  

It is binding for all unit tests, determinism tests, integration tests, and PR replay tests.

---

# 1. Core Testing Principles

## Principle 1 — Determinism Above All

A test MUST explicitly verify that:

- Same input → same output across repeated runs.

- FS traversal ordering does NOT affect results.

- Canonical cycle representation is stable.

- Alias resolution is stable.

- Diff behavior is stable.

Any nondeterministic behavior MUST fail the test suite.

---

## Principle 2 — Silence on Uncertainty

A test MUST assert that ArcSight remains completely silent when:

- confidence < 0.8  

- alias resolution ambiguous  

- import graph incomplete  

- cycle > 5 nodes  

- repo < 10 files  

- monorepo detected  

- performance exceeds SLA  

- canonicalization mismatch occurs  

Silence includes:

- no warnings  

- no logs  

- no partial output  

---

## Principle 3 — Zero False Positives

ArcSight MUST NEVER emit a warning unless ALL conditions are met:

- cycle exists  

- cycle is new  

- size 2–5  

- involves PR-changed files  

- root-cause edge detected  

- determinism validated  

Tests MUST confirm:

- incorrect cycles → silent  

- ambiguous inputs → silent  

- cycles without diff relevance → silent  

False negatives are acceptable; false positives are not.

---

## Principle 4 — Contract-First Testing

Tests MUST derive solely from:

- Foundation Contract  

- Safety Wedge Contract  

- API Contract  

- Determinism Requirements  

- Module Boundaries  

Tests MUST NOT:

- imagine future features  

- assume architecture inference  

- introduce new signals  

- test platform capabilities  

---

## Principle 5 — Pure Functions & No Hidden State  

Modules under test MUST be:

- pure  

- stateless  

- deterministic  

ArcSight MUST NOT:

- use hidden caches  

- memoize across PR runs  

- store module-level mutable state  

- retain imported Maps/Sets between tests  

- keep implicit global state  

All state MUST be:

- constant  

- static  

- readonly  

Tests MUST verify absence of hidden side effects.

---

# 2. Categories of Tests

## 2.1 Unit Tests

Verify each module in isolation:

- canonicalization  

- alias resolution  

- cycle detection  

- cycle diffing  

- PR-diff file relevance  

- confidence scoring  

- silent-mode gating  

Unit tests MUST NOT:

- rely on real FS traversal  

- require large repos  

- produce logs  

---

## 2.2 Determinism Tests (Critical)

Located in:

```
tests/determinism/
```

Tests MUST validate:

- repeated-run consistency  

- randomized input ordering  

- path normalization  

- canonicalization invariance  

- alias map stability  

Any mismatched output MUST fail the test suite.

---

## 2.3 Integration Tests

Simulate full PR evaluation:

- base vs head commits  

- diff-based cycle emergence  

- cycle removal  

- alias map variations  

Integration tests MUST verify:

- PR Update Invariance  

- silence on uncertainty  

- full determinism  

---

## 2.4 Replay Tests ("Historical PR Replays")

For design partner readiness:

- replay 20–50 historical PRs  

- expect 0 false positives  

- comment rate ≤ 20%  

- runtime p95 < 3s  

---

## 2.5 Performance Tests

Test with:

- deep import chains  

- wide fan-out  

- thousands of small files  

Runtime MUST satisfy SLO/SLA:

- p95 < 3 seconds  

- p99 < 7 seconds  

- >7 seconds → silent mode  

---

## 2.6 Error-Path Tests

Simulate:

- missing files  

- invalid imports  

- alias conflicts  

- FS read failures  

- graph truncation  

Expected output: complete silence.

---

# 3. Test File Naming Convention (NEW)

All test files MUST follow one of these formats:

```
{module}.test.ts
{module}.determinism.test.ts
{module}.integration.test.ts
```

Forbidden:

- scenario names (e.g., cycles-behavior.test.ts)

- combined multi-module tests

- descriptive wording (e.g., weird-alias-sample.test.ts)

Reason:

- ensures consistent test discovery  

- prevents Cursor from inventing scenario-based tests  

- preserves deterministic ordering across environments  

---

# 4. Golden Test Suite Requirements

Golden tests MUST include scenarios:

1. Direct 2-node cycle (A↔B)  

2. 3-node cycle (A→B→C→A)  

3. PR breaks existing cycle  

4. PR introduces cycle  

5. Alias-based cycle  

6. Alias ambiguity → silent  

7. Test files ignored  

8. Monorepo → silent  

9. Tiny repo (<10 files) → silent  

10. Cycle >5 nodes → ignored  

---

# 5. Forbidden Testing Practices

Tests MUST NOT:

- depend on timers  

- use async sleeps  

- rely on Date.now  

- use random data  

- snapshot large graphs  

- assume unordered inputs  

- use real git operations  

- call out to external processes  

---

# 6. Pre-Merge Test Requirements

Before merge:

- determinism suite fully green  

- no hidden state violations  

- performance tests pass  

- safety switch stable  

- confidence gating validated  

---

# 7. Pre-Release Requirements

Before tagging a version:

- replay harness → 0 false positives  

- cycle diff tests stable  

- normalization pipeline validated  

- FS determinism validated  

---

# 8. Authority

This document governs:

- how tests are written  

- how regressions are detected  

- how determinism is validated  

Subordinate only to Foundation Contract.

