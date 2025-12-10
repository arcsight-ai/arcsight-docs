# Analyzer Promotion Policy (FINAL UPDATED, Phase 2)

## 1. Purpose

This document defines the formal, deterministic promotion lifecycle for ArcSight analyzer versions, enforcing:

- correctness
- safety
- drift stability
- deterministic gating
- controlled rulepack evolution
- non-regressive analyzer behavior

The promotion pipeline:

```
shadow → canary → production (live)
```

is governed by Sentinel and ensures new analyzers do not introduce regressions in graph structure, cycles, rulepack behavior, extension fields, or deterministic guarantees.

This document governs all `analyzerVersion` changes across all environments.

---

## 2. Scope

### Included

This policy defines:

- `analyzerVersion` lifecycle rules
- allowed transitions between states
- Sentinel's role in drift gating
- manual approval requirements
- deterministic promotion constraints
- canary rollout rules
- freeze windows
- forbidden fallback behaviors

### Not Included

- rulepack versioning rules (see [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md))
- schema evolution rules (see [Schema Evolution Contract](./11-schema-evolution-contract.md))
- drift comparison rules (see [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md))
- envelope format (see [Envelope Format Spec](./15-envelope-format-spec.md))
- invariants and determinism rules (see [Invariants Contract](./06-invariants-contract.md), [Determinism Contract](./07-determinism-contract.md))

---

## 3. Definitions & Terms

**AnalyzerVersion**  
The version identifier for the ArcSight engine and bundled rulepacks.

**Shadow Analyzer**  
Candidate `analyzerVersion` run alongside live, invisible to users.

**Canary Analyzer**  
Narrow rollout of candidate version to a small subset of repos for controlled monitoring.

**Live Analyzer**  
The currently deployed, production `analyzerVersion`.

**Drift**  
Any structural or semantic difference between shadow and live envelopes.

**Blocker Drift**  
Drift that prohibits promotion (see [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)).

**Benign Drift**  
Drift allowed by Sentinel and not requiring human intervention.

---

## 4. Rules / Contract

### 4.1 Analyzer Version Lifecycle

Each `analyzerVersion` MUST progress through the following states:

1. shadow
2. canary
3. production (live)

No `analyzerVersion` may skip a stage.

No `analyzerVersion` may regress in version number.

### 4.2 Shadow Analyzer Execution Requirements

Shadow analyzer MUST:

- run on identical RepoSnapshot as live analyzer
- run with identical rulepack set
- run under identical limits (graph, cycles, rulepacks, timeouts)
- operate under identical sandbox configuration
- run with identical config snapshot
- remain completely invisible to end users

**NEW RULE (MUST):**

Shadow analyzer MUST match sandbox and limit configuration exactly with live analyzer.  
Any divergence invalidates drift analysis and MUST block promotion.

Shadow outputs MUST NEVER:

- modify PR results
- replace envelopes
- influence CheckRuns
- override degraded/error statuses

### 4.3 Sentinel Drift Gate

Promotion from shadow → canary requires:

- zero BLOCKER drift
- documented handling of WARNING drift
- optional BENIGN drift permitted
- structural compliance with [Envelope Format Spec](./15-envelope-format-spec.md)
- full schema normalization
- invariant satisfaction ([Invariants Contract](./06-invariants-contract.md))

If shadow analyzer:

- introduces new cycles
- deletes fields
- changes semantics without `rulepackVersion` bump
- degrades when live succeeds
- errors on repositories where live succeeds

→ Promotion MUST FAIL.

### 4.4 Canary Execution Rules

Once promoted to canary:

- candidate `analyzerVersion` runs on a restricted set of repositories
- live analyzer remains the authoritative source of truth
- canary results MUST NOT affect PR statuses
- canary MUST NOT retroactively modify shadow drift results

**NEW RULE (MUST):**

Canary results MUST NOT alter, reinterpret, or retroactively update any Sentinel drift classifications.  
Canary is observational only.

If canary produces:

- errors
- regressions
- new degraded outputs
- unexpected extension changes

→ Promotion MUST PAUSE until human evaluation.

### 4.5 Manual Approval Requirement (NEW — REQUIRED)

Even if Sentinel reports:

- zero BLOCKER drift
- zero WARNING drift
- no regressions
- deterministic improvements

**AnalyzerVersion promotion MUST STILL require:**

**Explicit human approval.**  
Automatic promotion is forbidden.

This prevents:

- semantic regressions
- rule misunderstanding
- overly aggressive cycle reduction
- silent correctness issues
- limit misconfiguration

No automated system may advance `analyzerVersion` without human approval.

### 4.6 Promotion Freeze Window (Optional but Recommended)

Organizations MAY enforce:

**No `analyzerVersion` promotions between Friday 12:00 UTC → Monday 12:00 UTC.**

**Rationale:**

- static analyzers often impact CI/CD
- regressions can break PR workflows
- reduces weekend outage risks

This aligns with freeze windows in:

- Google Tricorder
- Meta code analysis systems
- Stripe CI analyzers

### 4.7 Forbidden Promotion Behaviors

The following are strictly forbidden:

- skipping shadow or canary stages
- promoting `analyzerVersion` automatically
- modifying RepoSnapshot during shadow/canary runs
- relaxing sandbox limits for shadow analyzers
- discarding or rewriting drift reports
- applying fallback heuristics to "fit" expected drift
- comparing raw envelopes instead of normalized schema
- adjusting analyzer behavior based on environment variables
- promoting an analyzer that produces nondeterministic signatures

### 4.8 Allowed Promotion Behaviors

The following ARE allowed:

- reducing cycles if change is deterministic and approved
- adding new extension namespaces
- rulepack minor version bumps that pass Sentinel
- performance improvements
- improved canonicalization as long as schema remains valid

---

## 5. Determinism Impact

This policy ensures:

- drift classification remains reproducible
- promotion decisions are auditable
- environments do not diverge between analyzers
- limits remain fixed across all versions
- analyzer behavior does not change unpredictably
- deterministic comparison of live vs shadow vs canary

Any violation here compromises:

- drift detection
- rulepack stability
- CI correctness
- long-term reproducibility

Determinism is the foundation of the entire promotion process.

---

## 6. Runtime / Engine Boundary Impact

**Runtime MUST:**

- run live + shadow analyzers using identical inputs
- never rerun or "fix up" shadow results
- never modify envelopes
- never alter limits or sandbox parameters
- record canary outputs but never expose them
- ensure promotion actions are explicit and logged

**Engine MUST:**

- produce deterministic envelopes
- adhere to all limit and sandbox rules
- avoid environment-dependent behaviors
- produce degraded but valid envelopes under limit stress

---

## 7. Versioning Impact

Promotion to a new `analyzerVersion` requires:

- `analyzerVersion` bump (mandatory)
- `rulepackVersion` bump if rulepacks changed
- `schemaVersion` bump only if structure changed ([Schema Evolution Contract](./11-schema-evolution-contract.md))
- golden test updates if outputs differ
- updated drift baselines for Sentinel

Version bumps MUST reflect:

- semantic changes
- limit changes
- rulepack updates
- truncation behavior changes
- canonicalization improvements

---

## 8. Testing Requirements

### 8.1 Promotion Simulation Tests

CI MUST test:

- shadow vs live drift classification
- canary vs live regressions
- deterministic drift mapping

### 8.2 Limit and Sandbox Consistency Tests

Test infrastructure MUST ensure:

- identical environment across analyzer versions
- identical limits and sandbox constraints

### 8.3 Manual Approval Flow Tests

Promotion pipeline MUST validate:

- human approval required
- automated promotion impossible

### 8.4 Freeze Window Tests (If Enabled)

Test systems MUST reject promotion requests during freeze.

---

## 9. Cross-References

- [Invariants Contract](./06-invariants-contract.md)
- [Determinism Contract](./07-determinism-contract.md)
- [Schema Evolution & Adapters](./08-schema-evolution-and-adapters.md)
- [Golden Test Governance](./09-golden-test-governance.md)
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md)
- [Rulepack Versioning Contract](./12-rulepack-versioning-contract.md)
- [RepoSnapshot Contract](./14-repo-snapshot-contract.md)
- [Envelope Format Spec](./15-envelope-format-spec.md)
- [Sentinel Shadow Analysis Contract](./16-sentinel-shadow-analysis-contract.md)

---

## 10. Change Log (Append-Only)

**v1.1.0** — Added mandatory rules:

- identical sandbox + limits for shadow analyzer
- canary results MUST NOT modify shadow drift
- human approval required for promotion
- optional freeze window section

Integrated clean deterministic language and clarified gating semantics.

**v1.0.0** — Initial analyzer promotion policy.

