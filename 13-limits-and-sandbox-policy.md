# Limits & Sandbox Policy (FINAL UPDATED VERSION, Phase 1)

## 1. Purpose

This document defines ArcSight's canonical limit model and sandbox guarantees, which ensure that the engine:

- remains deterministic
- cannot be exploited via pathological repositories
- operates within fixed CPU/memory/time constraints
- produces safe, valid envelopes even under limit pressure
- never uses nondeterministic fallback modes

This policy applies to all ArcSight analysis executions:

- PR analysis
- CLI analysis
- shadow analyzers (Phase 2)
- drift detection comparisons
- historical replays

Limits exist to protect deterministic behavior, runtime stability, and user safety.

---

## 2. Scope

### Included

- Snapshot-size limits (runtime)
- Engine graph limits
- Cycle/truncation limits
- Rulepack runtime limits
- Memory/CPU time sandbox
- Hard failure vs degraded behavior
- Deterministic truncation rules
- Limit immutability rules
- Forbidden fallback logic
- Rulepack limit interaction

### Excluded

- Snapshot format ([RepoSnapshot Contract](./14-repo-snapshot-contract.md))
- Envelope format ([Envelope Format Spec](./15-envelope-format-spec.md))
- Determinism rules ([Determinism Contract](./07-determinism-contract.md))
- Schema evolution ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- Rulepacks ([Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))

---

## 3. Definitions & Terms

**Runtime Snapshot Limits**  
Constraints enforced by runtime before creating a RepoSnapshot.

**Engine Analysis Limits**  
Constraints enforced by the engine after snapshot creation.

**Hard Failure**  
Runtime rejection; engine is not invoked.

**Degraded Mode**  
Engine returns a valid deterministic envelope with:

```
core.status = "degraded"
meta.degraded_reason = <enum>
```

**Sandbox**  
The deterministic environment where engine code executes.

---

## 4. Rules / Contract

### 4.1 Two-Layer Limit Architecture

ArcSight uses two independent layers of protection:

| Layer | Enforced By | Failure Type | Engine Runs? |
|-------|-------------|--------------|--------------|
| Snapshot Limits | Runtime | Hard Failure | ❌ No |
| Analysis Limits | Engine | Degraded Mode | ✔ Yes |

This strict separation ensures no nondeterministic interactions between runtime and engine.

### 4.2 Runtime Snapshot Limits (Hard Failure)

Runtime MUST reject a repository without creating a snapshot if ANY of the following exceed configured thresholds:

#### 4.2.1 File Count

`MAX_FILE_COUNT` (default: 10,000)

#### 4.2.2 Total Bytes

`MAX_TOTAL_BYTES` (default: 50 MB)

#### 4.2.3 Maximum Path Length

`MAX_PATH_LENGTH` (default: 300 chars)

#### 4.2.4 Maximum Directory Depth

`MAX_DIRECTORY_DEPTH` (default: 20)

#### 4.2.5 Symlinks

Any symlink MUST cause a hard failure.

#### 4.2.6 Submodules

Submodules MUST be treated as empty directories OR rejected deterministically.

#### 4.2.7 Invalid/Escaping Paths

Any normalized path escaping repo root MUST cause a hard failure.

#### 4.2.8 Binary File Thresholds

Binary detection must follow [RepoSnapshot Contract](./14-repo-snapshot-contract.md).  
Runtime MUST NOT reinterpret binary as text.

**Hard Failure Consequence**

If any rule is violated:

- Snapshot MUST NOT be constructed
- Engine MUST NOT be invoked
- Runtime MUST return deterministic error metadata

### 4.3 Engine Analysis Limits (Soft Failure → Degraded Mode)

Once snapshot creation succeeds, the engine enforces analysis limits.

If ANY of the following exceed limits, engine MUST NOT error — it MUST degrade deterministically.

#### 4.3.1 Graph Size Limits

**Node Limit**

`MAX_GRAPH_NODES` (default: 40,000)

**Edge Limit**

`MAX_GRAPH_EDGES` (default: 60,000)

If exceeded:

- Engine MUST stop graph expansion
- MUST return `status = "degraded"`
- MUST set `meta.degraded_reason = "graph-node-limit-exceeded"` or `"graph-edge-limit-exceeded"`

#### 4.3.2 Cycle Output Limits

- `MAX_CYCLE_COUNT` (default: 2,000)
- `MAX_TOTAL_CYCLE_LENGTH` (default: 20,000)

If exceeded:

- Engine MUST truncate cycles deterministically
- MUST degrade
- MUST use canonical truncation rules (see §4.4)

#### 4.3.3 Rulepack Runtime Limits

`MAX_RULEPACK_RUNTIME_MS` (default: 200ms per namespace)

**Rules:**

- Limit applies per rulepack namespace, NOT per file
- Engine MUST NOT terminate analysis early
- If timeout occurs → rulepack output = empty, envelope = degraded
- `degraded_reason` MUST be `"rulepack-timeout"`

### 4.4 Deterministic Truncation Rules (NEW — REQUIRED)

If any limit requires truncation, then:

#### 4.4.1 Canonical Ordering Required Before Truncation

Engine MUST:

1. Compute cycles
2. Canonicalize cycles
3. Sort cycles lexicographically
4. THEN truncate via prefix

Never truncate before canonicalization.

#### 4.4.2 Prefix Truncation Only

Truncation MUST be:

```
cycles = cycles.slice(0, MAX_CYCLE_COUNT)
```

No rulepack may see or modify intermediate unsorted cycles.

#### 4.4.3 Deterministic Ordering Across Machines

Ordering MUST be identical on:

- Linux
- macOS
- Windows
- All Node LTS versions

### 4.5 Limit Immutability (NEW — REQUIRED)

All limit thresholds MUST:

- be evaluated once at engine initialization
- remain constant for the entire run
- NOT depend on repo content
- NOT depend on runtime heuristics
- NOT vary with environment variables
- NOT be adjusted by runtime

Dynamic or adaptive limit tuning is forbidden.

This guarantees shadow analyzers always run under identical conditions.

### 4.6 Degraded Mode Rules

If any engine limit is exceeded, engine MUST:

- produce a full, structurally valid envelope
- set `core.status = "degraded"`
- set `meta.degraded_reason` to a canonical enum
- still compute deterministic signature
- preserve all required fields
- use canonical truncation rules

**Allowed degraded reasons:**

- `"graph-node-limit-exceeded"`
- `"graph-edge-limit-exceeded"`
- `"cycle-limit-exceeded"`
- `"rulepack-timeout"`
- `"memory-limit"`

**Forbidden:**

- free-form strings
- dynamic values ("exceeded by 500")
- timestamps or contextual metadata

### 4.7 Hard Failure vs Degraded Mode Table

| Scenario | Example | Response |
|----------|---------|----------|
| Hard Failure | symlink found | No snapshot; engine not invoked |
| Hard Failure | repo > 50 MB | No snapshot |
| Soft Failure | too many nodes | Degraded envelope |
| Soft Failure | rulepack timeout | Degraded envelope |
| Forbidden | runtime retries analysis | MUST NEVER HAPPEN |

### 4.8 Forbidden Sandbox Behaviors

**Engine MUST NOT:**

- retry analysis
- change limits dynamically
- terminate before signature computed
- read system clock for timing
- access filesystem, network, env vars
- perform nondeterministic concurrency

**Runtime MUST NOT:**

- retry analysis with modified limits
- apply fallback analyzer configurations
- rewrite or mutate envelope
- drop or rewrite extensions

This rule permanently forbids GitHub-style "fallback mode" semantics.

### 4.9 Memory & CPU Limits (Sandbox Requirements)

Engine MUST execute inside a deterministic sandbox:

- fixed max heap
- no dynamic thread count
- no parallelism unless fully deterministic (Phase 2 only)
- no system-level timing APIs
- deterministic timeout enforcement

If sandbox kills engine for memory/time:

→ MUST produce degraded envelope via deterministic fallback path.

### 4.10 Stability & Evolution Rules

**Changing limit thresholds:**

- MUST require `analyzerVersion` bump
- MUST require golden updates
- MUST NOT require `schemaVersion` bump

**Changing truncation semantics:**

- MUST require `analyzerVersion` bump
- MUST update docs

---

## 5. Determinism Impact

Limits interact directly with:

- signature determinism
- cycle ordering
- golden tests
- shadow analyzer comparisons
- future enterprise rulepack outputs
- envelope drift detection

Nondeterministic limit handling would break the entire ArcSight pipeline.

This contract ensures:

- reproducibility
- stability across OS/machines
- enforcement clarity
- long-lived analyzer behavior compatibility

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- enforce snapshot limits
- treat RepoSnapshot as immutable once built
- NEVER retry analysis
- NEVER adjust limits dynamically
- never mutate envelope

**Engine MUST:**

- enforce analysis limits
- degrade deterministically
- compute signature even in degraded/error states

---

## 7. Versioning Impact

| Change | Requires analyzerVersion bump? | Requires schemaVersion bump? |
|--------|-------------------------------|------------------------------|
| Limit value change | ✔ Yes | ❌ No |
| New deterministic truncation category | ✔ Yes | ❌ No |
| New `degraded_reason` enum | ✔ Yes | ❌ No |
| New snapshot limit | ✔ Yes | ❌ No |
| Limit enforcement bug fix | ✔ Yes if output changes | ❌ No |

---

## 8. Testing Requirements

### 8.1 Hard Limit Tests

Test repos must:

- exceed limit
- meet limit
- be just under limit

Snapshot MUST behave deterministically.

### 8.2 Degraded Mode Tests

Test:

- deterministic `degraded_reason`
- deterministic cycle truncation
- deterministic signature
- stable output across OS/Node versions

### 8.3 Limit Immutability Tests

Simulate multiple engine runs where:

- limit constants must NOT drift
- signature must remain stable

### 8.4 Rulepack Limit Interaction Tests

Rulepacks MUST:

- drop output on timeout
- NOT attempt fallback logic
- NOT bypass degraded mode

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Schema Evolution Contract](./11-schema-evolution-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Major Limit Model Clarification

Added:

- deterministic truncation rules
- limit immutability
- canonical `degraded_reason` enums
- forbidden runtime fallback rules
- rulepack limit interaction rules
- memory/time sandbox clarity

**v1.0.0** — Initial Phase-1 Limit & Sandbox Policy

