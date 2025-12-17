> This document records observed behavior.  
> It is not a proposal and must not be retrofitted to justify features.

# Discover — Refactor Avoidance in Mature Codebases

## Status

**Frozen Discover Evidence**  
Not a proposal. Not a roadmap. Not a feature spec.

---

## Goal

Understand why experienced engineering teams avoid refactoring or modifying core parts of large systems, even when those changes are logically necessary or architecturally cleaner.

This document does not propose solutions.  
It records observable behavior only.

---

## Method

We analyzed high-risk, high-comment Pull Requests across mature open-source repositories.

### Repositories examined

- Apache Spark
- Apache Kafka

### Selection criteria

- Mature codebases (multi-year, many contributors)
- Large PRs (hundreds to thousands of LOC)
- Core system areas (scheduler, shuffle, storage, metadata, messaging)
- Extensive review discussion

### What we examined

- What did not change
- Where refactors explicitly stopped short
- What mechanisms replaced deeper changes
- How uncertainty was resolved at merge time

### Artifacts extracted per PR

- Avoided change (file / module / architectural boundary)
- Safety substitution (tests, flags, wrappers, scoped logic)
- Trust-based resolution language (“merge and revisit”, etc.)

---

## Evidence (Representative PRs)

| Repo          | PR        | Avoided Change                               | Safety Substitution                          | Trust-Based Resolution           | Notes                         |
|---------------|-----------|----------------------------------------------|----------------------------------------------|----------------------------------|-------------------------------|
| apache/spark  | PR #19468 | Core scheduler abstractions (YARN / Mesos / K8s) | Scoped implementation, heavy tests           | Yes — “Proceed as-is and revisit” | Scheduler refactor avoided    |
| apache/spark  | PR #28708 | Shuffle recomputation lifecycle              | Data copying, feature flags                  | Yes — “Merge, don’t block longer” | Lifecycle correctness deferred |
| apache/kafka  | PR #2929  | Log lifecycle & disk failure semantics       | Scoped IO handling, tests                    | Yes — “Follow-up patch”          | Core storage model preserved  |
| apache/kafka  | PR #9001  | Unified feature versioning                   | Versioned APIs, limited write path           | Yes — “Merge despite failures”   | End-to-end impact unclear     |
| apache/kafka  | PR #2476  | Offset lifecycle refactor                    | New API layered over old                     | Yes — “Rare corner case accepted” | Edge cases knowingly shipped  |
| apache/kafka  | PR #2614  | Message abstraction unification              | Legacy paths preserved                        | Yes — “Merge and follow up”      | Complexity retained           |
| apache/kafka  | PR #3874  | Replica lifecycle redesign                   | Incremental logic, flags                     | Yes — “Optimize later”           | Race complexity accepted      |

---

## Observed Pattern (Cross-Repo)

Across all cases, the same structure appears:

1. **A logically complete change is identified**  
   Reviewers explicitly acknowledge that a deeper refactor or unification would be cleaner.

2. **The change stops at a boundary**  
   A core abstraction, lifecycle, or module is intentionally not modified.

3. **Progress is achieved via substitution**

   - Tests instead of refactors  
   - Feature flags instead of deletion  
   - Wrappers instead of modification  
   - Incremental logic instead of simplification  

4. **Uncertainty is resolved by trust, not proof**

   - “Merge and revisit later”  
   - “Follow-up PR”  
   - “LGTM despite edge cases”  
   - “Not blocking this PR further”  

This pattern repeats across teams, years, and domains.

---

## Conclusion (Discover Nucleus)

Engineers avoid refactoring or modifying core abstractions because they cannot confidently prove the downstream impact of a change across the system.

This behavior is rational, not cultural.

It is **not** caused by:

- Laziness
- Lack of skill
- Process failure
- Aversion to refactoring

When impact cannot be proven, teams ship around the problem.

---

## What This Does Not Claim

It does **not** claim refactors are always correct.

It does **not** claim a single tool would have changed every decision.

It does **not** prescribe a solution.

It establishes a forced absence:

> The inability to prove change impact prevents engineers from acting, even when they know the system would be better if they did.

---

## Open Question (Deferred)

What concrete proof or visibility would have been required for engineers to safely cross the avoided boundary?

This is a synthesis question and is intentionally unanswered here.

---

## Status

Discover complete.  
Further Discover requires a new question, not more examples.


