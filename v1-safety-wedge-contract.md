# ArcSight Wedge — V1 Safety Wedge Contract

This contract defines the exact rules governing the ArcSight V1 Safety Wedge.  

All implementation MUST conform to these constraints.

---

## 1. V1 Output Surface

ArcSight V1 MUST:

- Only emit **new dependency cycles** introduced by a PR.

- Only emit cycles that:

  - include at least one file changed in the PR diff

  - are sized **2–5 nodes** (cycles with >5 nodes MUST be excluded)

  - can be deterministically canonicalized

  - contain a root-cause import edge originating from a **new import line in the PR diff** (DIFF import requirement)

ArcSight MUST output **exactly one PR comment**, updated in-place.

Fingerprint:

<!-- arcsight:pr-comment:v1:cycle-only -->

### 1.1 PR-Changed File Involvement Rules

A cycle MAY be emitted in V1 **only if** all of the following are true:

1. At least one node in the cycle corresponds to a file path listed in `changedFiles` for the PR.

2. At least one edge in the cycle can be mapped to a **new import line** in the PR diff.

3. The selected root-cause edge's `from` file is in `changedFiles`.

**Special cases:**

- If a cycle exists in `head` but **no files** in the cycle are in `changedFiles` → cycle MUST be ignored.

- If a cycle exists and files are in `changedFiles` but **no new import** in the diff closes the cycle → cycle MUST be ignored.

- If the same cycle persists across multiple pushes to the same PR, ArcSight MUST NOT re-warn; the existing comment remains unless the cycle disappears and reappears due to a different root-cause edge.

**Renames:**

- For V1, file renames/moves that introduce cycles without new imports are treated as **non-attributable** and MUST be ignored.

- **Silence on rename-only cycles:** Cycles that appear only due to file renames/moves without new import statements MUST result in silent mode (no PR comment).

**Cycle Size Limits:**

- Cycles with >5 nodes MUST be excluded from V1 output, regardless of other conditions.

**Repeated Cycle Handling:**

- If the same canonical cycle persists across multiple pushes to the same PR, ArcSight MUST NOT re-warn or update the comment unless:
  - The cycle disappears and then reappears, OR
  - The root-cause edge changes (different import line closes the cycle)

---

## 2. Deterministic Cycle Representation

Canonicalization MUST:

1. Normalize all paths to POSIX.  

2. Lowercase paths.  

3. Rotate cycle to lexicographically smallest starting node.  

4. Join nodes with `" → "`.  

5. Sort final cycles alphabetically.

---

## 3. Confidence Gating

ArcSight MUST remain silent unless:

- `confidence ≥ 0.8`

- repo is NOT a monorepo

- repo has ≥ 10 source files

- alias resolution is unambiguous

- segmentation pipeline reports no uncertainty

---

## 4. Silence-First Error Handling

ArcSight MUST remain silent when:

- import graph incomplete

- alias resolution ambiguous

- runtime > 7 seconds

- determinism mismatch across runs

- any unexpected condition occurs

---

## 5. False Negatives Allowed, False Positives Forbidden

ArcSight MAY:

- miss cycles  

- underreport cycles  

ArcSight MUST NOT:

- emit an incorrect cycle  

- infer structure  

- approximate behavior  

---

## 6. Safety Switch

ArcSight MUST disable comments entirely if:

- determinism fails  

- inconsistent canonicalization produced  

- root-cause detection unstable  

---

## 7. PR Comment Format (Developer-Safe)

⚠️ ArcSight Safety Check

This PR introduces a new dependency cycle:

{canonical_cycle}

New import creating the cycle:

{root_cause_edge}

Reply "ignore" to silence ArcSight for this PR.

**Prohibited terms:**

- architecture

- governance

- fragility

- risk

- stability

- blame language

---

## 8. **PR Update Invariance (New Clause)**

ArcSight MUST NOT update its PR comment unless:

- the set of relevant cycles has changed OR

- the associated root-cause edges have changed.

No churn.  

No UI disruption.  

No cosmetic updates.

This ensures consistent developer experience.

---

## 9. Authority

This contract is authoritative over:

- implementation  

- PR surface  

- testing  

- determinism  

It is subordinate only to the Foundation Contract.
