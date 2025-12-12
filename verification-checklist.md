# ArcSight Wedge — Verification Checklist (V1)

This checklist MUST pass before any PR is merged or version released.

---

# 1. Pre-Merge Checklist (Every PR)

### Determinism

- [ ] Identical output across 3 repeated runs  

- [ ] Canonicalization stable  

- [ ] Import graph stable  

- [ ] FS traversal sorted  

### Safety

- [ ] No new signals introduced  

- [ ] No architecture inference  

- [ ] PR Update Invariance enforced  

- [ ] Silent-on-error behavior intact  

### Safety Switch Health (NEW)

- [ ] SafetySwitch did NOT activate unexpectedly  

- [ ] No determinism mismatch flags raised  

- [ ] No performance triggers tripped  

### Code Quality

- [ ] No logs  

- [ ] No forbidden dependencies  

- [ ] No cross-boundary imports  

### Tests

- [ ] Determinism suite passes  

- [ ] Tiny + large repo tests pass  

- [ ] Alias ambiguity cases covered  

---

# 2. Pre-Release Checklist

### Functionality

- [ ] Cycles-only  

- [ ] 2–5 node constraint validated  

- [ ] Root-cause detection correct  

### Confidence

- [ ] Scoring validated  

- [ ] Gating logic respected  

### Performance

- [ ] p95 <3s  

- [ ] p99 <7s  

### Silent Mode

- [ ] Covers all error states  

- [ ] No partial outputs permitted  

---

# 3. Design Partner Readiness

- [ ] 20 historical PRs → 0 false positives  

- [ ] Comment wording approved  

- [ ] PR Update Invariance validated  

- [ ] Latency acceptable  

- [ ] Comment rate <20%  

---

# 4. SSOT Compliance

- [ ] All changes align with contracts  

- [ ] ADRs updated if needed  

Any failure → merge/release blocked.

