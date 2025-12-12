# ArcSight Wedge Philosophy

**Version:** 1.0  
**Status:** GUIDING PRINCIPLES  
**Effective Date:** Phase 1

---

## Purpose

This document articulates the core philosophy behind the ArcSight Wedge engine. It explains **why** we make certain design decisions and helps prevent product dilution and accidental contract breaks.

---

## Core Principles

### 1. Determinism First

**Why it matters:**

Determinism is not a feature—it's a **requirement**. Without determinism:
- Results cannot be trusted
- Debugging becomes impossible
- Reproducibility is lost
- CI/CD integration fails

**What it means:**

- Identical inputs → Identical outputs
- No timestamps, random numbers, or environment dependencies
- Byte-for-byte identical outputs across runs
- Golden tests enforce determinism

**How we enforce it:**

- All outputs are sorted explicitly
- All dates are injected (never read from system)
- All random operations are forbidden
- Golden tests verify determinism (3× runs)

---

### 2. Adapters Are Thin

**Why it matters:**

The engine is the **source of truth**. Adapters (CLI, GitHub PR) are **presentation layers** that format engine output. Keeping adapters thin:
- Prevents logic duplication
- Ensures consistency across adapters
- Makes testing easier
- Preserves engine purity

**What it means:**

- Engine produces canonical output
- Adapters format output (JSON, Markdown)
- No business logic in adapters
- Adapters are deterministic formatters

**How we enforce it:**

- Engine output is fully canonicalized
- Adapters use pure formatting functions
- No adapter-specific logic
- Cross-adapter consistency tests

---

### 3. Waivers Are Annotations

**Why it matters:**

Waivers are **audit trails**, not suppressions. They:
- Preserve violation history
- Enable governance oversight
- Support compliance reporting
- Maintain transparency

**What it means:**

- Waivers **annotate** violations (never remove them)
- Waived violations are visible in summaries
- Waivers are excluded from risk scoring (via `effectiveViolations`)
- Waiver application is fully auditable

**How we enforce it:**

- Violations are never deleted
- Waiver metadata is stored in `extension_data`
- Waiver application is deterministic
- Waiver history is preserved

---

### 4. Contracts Are Immutable

**Why it matters:**

Contracts define **guarantees**. Breaking contracts:
- Breaks consumer trust
- Causes integration failures
- Creates maintenance burden
- Violates determinism

**What it means:**

- Phase 1–3.3 contracts are **frozen**
- Changes require version bumps
- Backward compatibility is maintained
- Breaking changes are explicitly documented

**How we enforce it:**

- Frozen boundaries document (`docs/frozen-boundaries.md`)
- Contract validation in tests
- Version bump requirements
- Integration audit verifies contracts

---

### 5. Fail-Safe by Default

**Why it matters:**

Safety is more important than features. The engine:
- Fails closed (silent mode) on ambiguity
- Never mutates inputs
- Never throws unhandled errors
- Never breaks determinism

**What it means:**

- Errors trigger silent mode (no output)
- Invalid inputs are rejected gracefully
- Waiver failures don't break engine
- Determinism violations are caught

**How we enforce it:**

- Silent mode on errors
- Input validation
- Error handling in all paths
- Determinism tests catch violations

---

## Design Decisions

### Why No Timestamps?

Timestamps break determinism. The same code run at different times produces different outputs. We use **injected evaluation dates** instead.

### Why No Random Numbers?

Random numbers break determinism. We use **deterministic hashing** (SHA-256) for IDs and fingerprints.

### Why Explicit Sorting?

Implicit sorting (e.g., `Object.keys()` iteration order) is not guaranteed. We **explicitly sort** all arrays and objects before output.

### Why Thin Adapters?

Thick adapters duplicate logic and create inconsistencies. We keep adapters **thin** and move all logic to the engine.

### Why Annotate, Not Suppress?

Suppressing violations hides problems. We **annotate** violations with waiver metadata, preserving audit trails.

---

## Anti-Patterns

### ❌ Adding Features to Frozen Components

**Problem:** Breaks contracts, violates determinism

**Solution:** Create new components or version bump

### ❌ Using System Time

**Problem:** Breaks determinism

**Solution:** Inject evaluation date as parameter

### ❌ Mutating Inputs

**Problem:** Breaks immutability, causes side effects

**Solution:** Always clone inputs before modifying

### ❌ Logic in Adapters

**Problem:** Duplicates logic, creates inconsistencies

**Solution:** Move logic to engine, keep adapters thin

### ❌ Suppressing Violations

**Problem:** Hides problems, breaks audit trail

**Solution:** Annotate violations, exclude from scoring

---

## For New Contributors

### Before You Code

1. **Read the contracts** (`docs/module-contracts/`)
2. **Understand determinism** (`docs/determinism-rules.md`)
3. **Check frozen boundaries** (`docs/frozen-boundaries.md`)
4. **Review philosophy** (this document)

### When You Code

1. **Inject dates** (never use `Date.now()`)
2. **Sort explicitly** (never rely on iteration order)
3. **Clone inputs** (never mutate)
4. **Write golden tests** (verify determinism)

### After You Code

1. **Run determinism tests** (3× repeated runs)
2. **Check contract compliance** (validate schemas)
3. **Verify no mutations** (deep clone checks)
4. **Update documentation** (if contracts change)

---

## Long-Term Vision

### Phase 1–3.3: Core Engine (FROZEN)

- Deterministic engine
- Thin adapters
- Waiver system
- Contract compliance

### Phase 4+: Adoption Features

- New features in new components
- No changes to frozen components
- Version bumps for breaking changes
- Backward compatibility maintained

---

## Related Documents

- `docs/frozen-boundaries.md` — Frozen component boundaries
- `docs/determinism-rules.md` — Determinism requirements
- `docs/phase3.1-waiver-audit.md` — Waiver system audit
- Contract documents in `docs/module-contracts/`

---

**Last Updated:** Phase 3.4  
**Status:** Guiding Principles

