# ArcSight Wedge — Dependency Policy (V1)

This policy guarantees determinism, purity, minimalism, and safety.

---

# 1. Allowed Dependencies

### Node.js Standard Library

Permissible modules include:

- fs (sorted traversal required)

- path

- url

- os

- crypto (hash only; no randomness)

Restrictions:

- MUST explicitly **sort** all outputs from fs.readdir/fs.readdirSync  

- MUST normalize paths to POSIX  

- MUST lowercase paths  

### Optional Small Utilities

Allowed only if:

- deterministic  

- no global state  

- no internal async workers  

- lexicographically ordered outputs  

All optional dependencies require RFC + ADR.

---

# 2. Forbidden Dependencies

### AST Tools (Forbidden)

- Babel  

- @babel/parser  

- swc  

- ts-morph  

- TS compiler API  

### Build Tools / Bundlers

- esbuild  

- parcel  

- webpack  

### Formatting/Parsing Tools

- prettier  

- eslint  

- magic comment processors  

### Any Library With:

- nondeterministic ordering  

- randomness  

- concurrency hazards  

- plugin loading  

- environment-dependent execution  

---

# 3. Dependency Governance

To add a dependency:

1. RFC  

2. ADR  

3. SSOT update  

4. Determinism validation  

CI MUST block dependency drift.

---

# 4. Philosophy

ArcSight Wedge is intentionally **dependency-light** to protect:

- determinism  

- performance  

- reproducibility  

- simplicity  

- auditability  

No new dependencies until wedge success + v1 freeze ends.

