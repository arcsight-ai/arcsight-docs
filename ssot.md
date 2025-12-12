# ArcSight Wedge — Single Source of Truth (SSOT)

This document defines the authoritative sources of truth for V1.  

Only the documents listed here have authority over implementation and behavior.

---

## Batch 1 — Constitutional Documents (Authoritative NOW)

### 1. Foundation Contract  

`00-foundation-contract.md`  

Defines determinism, silence, constraints, and non-goals.

### 2. Safety Wedge Contract  

`v1-safety-wedge-contract.md`  

Defines V1 output rules, cycle handling, PR surface, and gating.

### 3. API Contract  

`api-contract-v1.md`  

Defines the frozen public API.

### 3.1 Types Definition  

`types.md`  

Defines all shared types used across API contract, design doc, and implementation.

All type definitions are authoritative and frozen for V1.

### 4. Project Philosophy  

`project-philosophy.md`  

Explains purpose, intent, and wedge purity.

### 5. Future Integration Contract  

`future-integration-contract.md`  

Ensures clean future evolution without wedge corruption.

### 6. SSOT  

`ssot.md`  

This file.

### 7. **Architectural Decision Records (New)**

Located in `/docs/adrs/`.  

Each ADR captures binding historical precedent.  

No implementation may contradict an ADR.

---

## Authority Ordering

1. **Foundation Contract**  

2. **Safety Wedge Contract**  

3. **API Contract**  

4. **Project Philosophy**  

5. **Future Integration Contract**  

6. **ADRs**  

7. **All implementation**  

Batch 2 and beyond will extend SSOT after generation.

---

## Cursor Instruction

Cursor MUST treat SSOT as the **ultimate truth index**.  

If a document is not referenced here, it MUST NOT influence code generation.
