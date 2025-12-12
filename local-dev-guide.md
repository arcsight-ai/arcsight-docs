# ArcSight Wedge — Local Development Guide (V1 • Final)

**Status:** Stable  

**Audience:** Developers contributing to ArcSight Wedge V1  

**Purpose:** Guarantees all local development preserves determinism, wedge purity, and V1 safety guarantees.

---

# 1. Local Development Principles

ArcSight Wedge is a deterministic safety engine. Local development MUST protect:

- determinism  

- silent fallback  

- cycles-only purity  

- confidence gating behavior  

- module-boundary guarantees  

- zero false positives  

Developers MUST optimize for **minimal change**, not abstraction quality.

---

# 2. Required Tooling (Deterministic Tooling Lock)

Developers MUST use:

- Node version specified in `.nvmrc`  

- Yarn or pnpm (npm lockfiles forbidden)  

- Local TypeScript config (`tsconfig.json`) ONLY  

❗ Mandatory Tooling Lock  

You MUST run Node using the exact version in `.nvmrc`.  

No other Node version is permitted for development, tests, or CI.

Before committing code:

```
node -v
cat .nvmrc
```

If they do not match → determinism is NOT guaranteed → DO NOT COMMIT.

Forbidden tools:

- global TS configs  

- Babel  

- SWC / ts-morph  

- Jest transformers  

- ESLint / Prettier  

---

# 3. Determinism Checks Before Commit

Before committing code, you MUST validate:

### 3.1 Determinism Suite (Run 3 Times)

```
yarn test:determinism
yarn test:determinism
yarn test:determinism
```

All outputs MUST match exactly.

### 3.2 No Logging

```
grep -R "console." -n src
```

Must return **0 results**.

### 3.3 No Hidden State

Check modules for:

- module-level mutable variables  

- implicit caches  

- memoized maps/sets  

- retained state across calls  

All state MUST be:

- readonly  

- constant  

- recreated per invocation  

### 3.4 Performance Budget

Large-repo test MUST complete in:

- p95 < 3s  

- p99 < 7s  

If >7s → treat analysis as uncertain → wedge MUST remain silent.

---

# 4. Filesystem Behavior

Never assume filesystem traversal is ordered.  

ALL FS results MUST be lexicographically sorted before use.

Path rules:

- MUST use POSIX normalization  

- MUST lowercase  

- MUST apply normalization pipeline (see determinism-requirements.md)  

---

# 5. Working on Modules

Follow module-boundaries.md strictly.

When modifying modules:

- preserve module purity  

- never merge modules  

- never introduce cross-boundary imports  

- keep functions pure and stateless  

Developers MUST NOT:

- introduce abstractions  

- reorganize architecture  

- create shared utilities  

- add any new signals  

---

# 6. Determinism Manual Testing (Optional but Recommended)

```
node tools/run-wedge.js repoA > out1.json
node tools/run-wedge.js repoA > out2.json
diff out1.json out2.json
```

Diff MUST be empty.

---

# 7. Alias Resolution Testing

You MUST verify:

- alias maps resolve deterministically  

- ambiguous alias → silent mode  

- root-cause edge detection respects alias mapping  

---

# 8. Safety Switch Testing

Wedge MUST fall silent when:

- alias ambiguity detected  

- cycle set nondeterministic across runs  

- performance >7s  

- import graph incomplete  

- normalization fails  

---

# 9. Developer Behavior Expectations (NEW)

Contributors MUST:

- favor **minimal diffs**  

- avoid architectural redesigns  

- avoid refactors unless explicitly approved  

- keep functions tiny, pure, and specific  

- prefer "local repair" over "structural change"  

- reject AI or human suggestions that expand scope  

If in doubt:

**"Minimal, deterministic, cycles-only."**

---

# 10. Pre-PR Checklist

Before submitting a PR:

- determinism validated  

- performance within SLO  

- module boundaries preserved  

- no new imports added  

- no logging added  

- no hidden state  

- tests adjusted only if explicitly needed  

- no API contract drift  

---

# 11. Authority

This guide is binding until wedge validation + wedge freeze concludes.

