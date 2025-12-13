# ArcSight — Trying It Safely

ArcSight is a deterministic architectural analyzer.

It reports only dependency cycles that are mathematically provable.

If it prints nothing actionable, that means no provable architectural violations were detected.

ArcSight prefers silence to uncertainty.

If it cannot prove a result, it does not report it.

---

## How to try it safely

### 1. Run on a repository you already trust:

```bash
npx arcsight check .
```

### 2. Try replaying history (recommended):

```bash
npx arcsight replay . <PR_NUMBER>
```

Replay mode shows how ArcSight would have behaved in the past, without touching live workflows.

---

## What to expect

- **Silence** = no provable architectural violations detected
- **Low confidence** = analysis suppressed to avoid false positives
- **Output is deterministic** (same input → same output every run)

Silence does not mean ArcSight did nothing. It means no provable architectural violations were found.

---

## What ArcSight does not do

- No opinions
- No refactor advice
- No architecture scoring
- No AI suggestions

If something surprises you — good or bad — that feedback is valuable.

