> This document records observed behavior.  
> It is not a proposal and must not be retrofitted to justify features.

# Discover — Trust-Based Resolution Under Unprovable Impact

## Status

**Frozen Discover Evidence**  
Not a proposal. Not a roadmap. Not a feature spec.

---

## Goal

Understand how engineering teams resolve merge decisions when correctness, safety, or impact cannot be conclusively proven.

This document does not evaluate whether decisions were correct.  
It records how decisions were made under uncertainty.

---

## Method

We analyzed high-discussion Pull Requests in mature, safety-critical codebases where reviewers explicitly acknowledged unresolved risk or incomplete reasoning.

### Repositories examined

- Apache Spark
- Apache Kafka

### Selection criteria

- PRs touching core or cross-cutting system behavior
- Extended review threads with unresolved questions
- Explicit acknowledgement of uncertainty
- Eventual merge without full resolution

### What we examined

- How reviewers framed uncertainty
- What evidence was considered “good enough”
- What arguments ended the discussion
- What conditions allowed the merge to proceed

---

## Evidence (Representative PRs)

| Repo  | PR     | Unresolved Concern                                      | Resolution Mechanism          | Deciding Language                     | Notes                               |
|-------|--------|---------------------------------------------------------|-------------------------------|---------------------------------------|-------------------------------------|
| spark | #19468 | Scheduler abstraction correctness across cluster managers | Scope reduction + deferral    | “Proceed as-is and revisit later”     | Architectural uncertainty acknowledged |
| spark | #28708 | Shuffle lifecycle correctness on node shutdown          | Data duplication + flags      | “Don’t want to block the PR longer”   | Correctness deferred                |
| kafka | #9001  | Feature versioning behavior under failover & upgrade   | Existing failures accepted    | “Failures are not new”                | Merge despite known failures        |
| kafka | #2929  | Disk failure semantics across log & replica manager    | IOException-only handling     | “Follow-up patch”                     | Failure modes intentionally incomplete |
| kafka | #3874  | Replica state races                                    | Incremental guards            | “Optimize later”                      | Known complexity accepted           |

---

## Observed Pattern

Across repositories and years, the same decision structure appears:

1. **Uncertainty is explicitly named**  
   Reviewers articulate scenarios they cannot fully reason about (failover, race conditions, downstream consumers, partial upgrades).

2. **No decisive counterexample is found**  
   The discussion does not converge on a proof of correctness or incorrectness.

3. **A stopping argument is introduced**

   Examples:

   - “This is no worse than existing behavior”  
   - “These failures already exist”  
   - “The API is rarely used”  
   - “We can follow up later”  

4. **Merge authority substitutes for proof**  
   A trusted reviewer or maintainer explicitly accepts responsibility:

   - “LGTM”  
   - “I’ll merge this”  
   - “We’ll revisit if needed”  

   The merge occurs without eliminating the uncertainty.

---

## Key Observation

When impact cannot be proven, merge decisions are resolved socially, not technically.

Trust replaces certainty.

This is not accidental — it is the only available mechanism once analytical proof is exhausted.

---

## What This Does Not Claim

It does **not** claim these merges were mistakes.

It does **not** claim tests or reviews were insufficient.

It does **not** claim that better discipline would have resolved uncertainty.

It does **not** claim that trust is bad.

It claims only that trust is used as a substitute for impact proof when proof is unavailable.

---

## Relationship to Other Discoveries

This document is complementary to:

- **Discover — Refactor Avoidance in Mature Codebases**

Refactor Avoidance explains **why changes stop**.  
Trust-Based Resolution explains **how decisions still proceed**.

Together, they describe the full decision loop under uncertainty.

---

## Open Question (Deferred)

What form of system knowledge would have reduced reliance on trust at merge time?

This question is intentionally unanswered here.

---

## Status

Discover complete.  
Additional examples add redundancy, not clarity.


