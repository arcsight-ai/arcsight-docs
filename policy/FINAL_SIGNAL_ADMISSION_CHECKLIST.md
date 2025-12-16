This document enforces [philosophy.md](../philosophy.md).

Any conflict must be resolved in favor of the philosophy.

---

# Final Signal Admission Checklist

A signal may appear in ArcSight's default summary only if all are true:

- [ ] Provable via language semantics or structural graphs

- [ ] Irreversible without code change

- [ ] Structurally permanent (not contextual or temporal)

- [ ] Language-scoped (not ecosystem or framework-based)

- [ ] Safe to suppress when uncertain

- [ ] Requires no explanation or interpretation

- [ ] Introduces no ranking, scoring, or prediction

If any box is unchecked, the signal must NOT enter the default summary.

---

## 🔒 Interpretive Clarification: Discover vs Check

This section is interpretive only.
It introduces no new admission criteria, authority, or policy.

All signals admitted through this checklist must already satisfy the existing requirements of being provable, deterministic, structurally permanent, and irreversible.

This section clarifies how those already-admitted signals are classified.

### Discover

A signal is classified as **Discover** if it represents a structural truth that exists in a single snapshot of the codebase.
Its existence does not depend on history, comparison, or change.

### Check

A signal is classified as **Check** if it represents a structural truth that is only knowable through comparison between two states.
Its existence depends on change relative to a baseline.

### Governing Rule

If it would still be true in a fresh clone with no history → **Discover**.
If it only exists because something changed → **Check**.

This classification does not affect whether a signal is admissible.
It determines only how and when an admissible signal may be surfaced.

Reclassifying a signal between Discover and Check after admission would constitute a breaking semantic change.

This section is frozen.

