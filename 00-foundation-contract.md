# ArcSight Wedge — Foundation Contract (V1)

## Purpose

This document defines the foundational constraints, guarantees, and non-negotiable principles governing the ArcSight Wedge repository. It establishes the ground rules that all code, documentation, tests, and future extensions MUST follow. This contract is authoritative and immutable for V1.

ArcSight Wedge exists to provide a **deterministic, zero-noise, PR-time safety engine** that warns developers only when a **new dependency cycle** is introduced.

---

## Core Principles

### 1. Determinism Above All

- Same input → same output, across machines, runs, environments.

- No randomness, timestamps, non-deterministic libraries, or FS-order assumptions.

### 2. Zero False Positives

- ArcSight MUST NOT generate an incorrect warning under any circumstance.

- False negatives are allowed; false positives are fatal.

### 3. Silence on Uncertainty

- If the system is not 100% certain about correctness, it MUST remain silent.

### 4. Minimalism

- V1 MUST implement the smallest possible safety surface: **cycles only**.

- No heuristics, no scoring, no architecture inference.

### 5. Safety Over Coverage

- Missing a cycle is acceptable.

- Emitting incorrect information is not.

### 6. Contract-First Development

- All behavior MUST be defined in this document, the Safety Wedge Contract, and the API Contract before code is written.

### 7. No Feature Creep

- The wedge MUST NOT evolve beyond V1 scope without RFC + ADR approval.

### 8. **No Logging (New Clause)**

ArcSight Wedge MUST NOT:

- produce console output  

- emit debug logs  

- generate telemetry  

- print internal state dumps  

Logging introduces nondeterminism, performance variance, and noise.  

Logging MAY be added in future versions behind deterministic, configuration-controlled flags.

---

## Explicit V1 Goals

- Detect new dependency cycles introduced in a pull request.

- Show the root-cause import edge that closes the cycle.

- Restrict output to cycles sized 2–5.

- Ensure warnings involve PR-changed files.

- Maintain strict determinism and reproducibility.

- Stay silent unless confidence ≥ 0.8.

---

## Explicit V1 Non-Goals

ArcSight Wedge MUST NOT:

- detect god files

- detect forbidden imports

- detect boundaries or layers

- compute architecture metrics

- compute fragility/exposure

- generate recommendations

- analyze module stability

- support monorepos

- support multi-language analysis

- parse ASTs or depend on Babel/ts-morph

These belong to post-wedge phases.

---

## Error Handling Contract

ArcSight Wedge MUST:

- Stop analysis and remain silent upon ANY error.

- Trigger the Safety Switch on unexpected states.

- Never degrade gracefully with partial or speculative output.

---

## Allowed Technologies

- Node.js standard library  

- Deterministic file operations  

- Pure functional TypeScript modules  

Forbidden:

- Babel  

- ts-morph  

- AST libraries  

- code formatters that alter behavior  

- bundlers  

- random or unstable dependencies  

---

## Versioning & Changes

This foundation contract is **frozen for V1**.

Changes require:

1. RFC proposal  

2. ADR approval  

3. SSOT update  

---

## Authority

This contract is authoritative over all implementation and testing.

It is referenced by:

- `v1-safety-wedge-contract.md`

- `api-contract-v1.md`

- `ssot.md`
