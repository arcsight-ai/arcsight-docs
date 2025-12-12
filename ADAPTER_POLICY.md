# ArcSight Adapter Policy

**Version:** v1.0.0  
**Date:** 2024-12-19  
**Status:** Active Policy

---

## Overview

This document defines the policy for adapter repositories (e.g., `arcsight-github-app`, `arcsight-cli`) that consume the core contract layer from `@arcsight-ai/wedge/core`.

---

## 1. Adapter Definition

An **adapter** is a repository that:
- Consumes the core contract layer from `@arcsight-ai/wedge/core`
- Wraps or transforms wedge output for a specific runtime (GitHub, CLI, etc.)
- Does **NOT** define core contracts locally
- May define adapter-specific contracts (e.g., `src/contracts/github-*`)

---

## 2. Core Contract Import Policy

### 2.1 Required Imports

All core contract types MUST be imported from `@arcsight-ai/wedge/core`:

```typescript
// ✅ CORRECT
import {
  Violation,
  Analyzer,
  Rulepack,
  Snapshot,
  EnvelopeSchema,
  PRSummary,
  CONTRACT_VERSION
} from "@arcsight-ai/wedge/core";

// ❌ INCORRECT - Local core contracts
import { Violation } from "../src/core/violation";
import { Violation } from "../contracts/core/violation";
```

### 2.2 Forbidden Patterns

Adapters MUST NOT:
- Define core contracts in `src/core/` or `src/contracts/core/`
- Duplicate contract type definitions locally
- Import from `docs-submodule` for runtime contract types (docs-submodule is for documentation only)

### 2.3 Allowed Adapter Contracts

Adapters MAY define adapter-specific contracts:
- `src/contracts/github-*` (GitHub App specific)
- `src/contracts/cli-*` (CLI specific)
- Other adapter-specific contract namespaces

---

## 3. Contract Version Alignment

### 3.1 Version Declaration

Adapters MUST declare their supported contract version by importing `CONTRACT_VERSION`:

```typescript
// src/contractSupport.ts
import { CONTRACT_VERSION } from "@arcsight-ai/wedge/core";

export const SUPPORTED_CONTRACT_VERSION = CONTRACT_VERSION;
```

### 3.2 CI Enforcement

CI MUST verify:
1. No rogue core contracts in `src/core/` or `src/contracts/core/`
2. `SUPPORTED_CONTRACT_VERSION` matches `CONTRACT_VERSION` from `@arcsight-ai/wedge/core`
3. All core contract imports reference `@arcsight-ai/wedge/core`

---

## 4. Adapter Responsibilities

### 4.1 Wrapping & Transformation

Adapters may:
- Transform wedge output for their runtime (e.g., GitHub Check Runs, CLI output)
- Add adapter-specific metadata
- Format output for their target platform

### 4.2 Contract Compliance

Adapters MUST:
- Preserve contract semantics when transforming
- Maintain determinism guarantees
- Not mutate core contract data structures

---

## 5. Enforcement

### 5.1 Pre-commit Hook

Local pre-commit hooks MUST block:
- New files in `src/core/` or `src/contracts/core/`
- Imports from local core contract paths

### 5.2 CI Checks

GitHub Actions MUST:
- Check for rogue core contracts on every PR
- Verify contract version alignment
- Fail builds if violations are detected

---

## 6. Examples

### GitHub App Adapter

```typescript
// ✅ CORRECT - Import from wedge/core
import { Violation, EnvelopeSchema, CONTRACT_VERSION } from "@arcsight-ai/wedge/core";
import { SUPPORTED_CONTRACT_VERSION } from "./contractSupport";

// ✅ CORRECT - Adapter-specific contract
import { GitHubCheckRunOutput } from "./contracts/github-checkrun";

// ❌ INCORRECT - Local core contract
import { Violation } from "./core/violation";
```

### CLI Adapter

```typescript
// ✅ CORRECT - Import from wedge/core
import { Rulepack, Snapshot, Analyzer } from "@arcsight-ai/wedge/core";

// ✅ CORRECT - Adapter-specific contract
import { CLIOutputFormat } from "./contracts/cli-output";
```

---

## 7. Related Documents

- [Contract Layer Spec v1](./21-contract-layer-spec-v1.md)
- [Phase 1 Core Contract Layer](./21A-phase1-core-contract-layer.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)

---

## 8. Policy Updates

This policy is versioned and must be updated when:
- New adapter patterns emerge
- Contract import requirements change
- Enforcement mechanisms are updated

