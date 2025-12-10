# ArcSight — Layered Usage Model  

### How Documents 1, 2, and 3 Work Together

ArcSight has three primary architecture documents:

1. **Document 1 — Strategic Architecture (Decision Layer)**  
2. **Document 2 — Physical Architecture (Folder Scaffold)**  
3. **Document 3 — Operational Architecture (System Roadmap — CTO Sign-Off)**  

This document explains how they relate, which one is authoritative in each phase, and how to navigate the architecture without confusion.

It introduces the "four-layer model" used by real-world engineering orgs (Stripe, Google Tricorder, Meta Infer, Snyk).

---

# 🎯 PURPOSE OF THIS DOCUMENT

To answer:

- "Which document is the source of truth right now?"  
- "What should I follow when coding?"  
- "When should we read which document?"  
- "How do these documents relate?"  
- "How do we avoid contradictions or ambiguity?"  

This is the **meta-level guide**, not a design doc.

---

# 🧠 THE FOUR-LAYER MODEL

ArcSight's documentation hierarchy looks like this:

```
┌──────────────────────────────────┐
│ Document 3 — System Roadmap      │
│ (Operational Architecture)      │
│ — What to build & in what order │
└──────────────────────────────────┘
▲
│
┌──────────────────────────────────┐
│ Document 2 — Folder Scaffold     │
│ (Physical Architecture)          │
│ — Where code lives & boundaries │
└──────────────────────────────────┘
▲
│
┌──────────────────────────────────┐
│ Document 1 — Decision Layer      │
│ (Strategic Architecture)         │
│ — When to stay separate vs mono │
└──────────────────────────────────┘
▲
│
┌──────────────────────────────────┐
│ Document 4 — Usage Model         │
│ (Meta Architecture)              │
│ — How to use the documents       │
└──────────────────────────────────┘
```

Each layer zooms into the next.

- Document 1 = **When**  
- Document 2 = **Where**  
- Document 3 = **What + How**  

This document = **How to use all three correctly**.

---

# 📄 DOCUMENT 1 — Strategic Architecture (Decision Layer)

### Summary  
"Stay separate in Phase 1. Migrate to monorepo in Phase 2."

### Purpose  
This governs **high-level structural strategy**, such as:

- When do we reorganize code?
- Why not use a monorepo now?
- When will the monorepo be required?
- What conditions must be met before migration?

### You use it for:

- Planning  
- Onboarding  
- Architecture discussions  
- Long-term decisions  

### You do NOT use it for:

- Deciding where files go  
- Coding  
- Day-to-day engineering  

This is **strategic**, not operational.

---

# 📄 DOCUMENT 2 — Physical Architecture (Folder Scaffold)

### Summary  
Defines the folder trees for:

- arcsight-wedge  
- arcsight-cli  
- arcsight-github-app  

### Purpose  
This is the **developer-facing blueprint**.

It tells you:

- Where every module belongs  
- Allowed imports  
- Forbidden imports  
- Responsibility boundaries  
- What code lives in which package  

### You use it for:

- Writing code  
- PR reviews  
- Creating new modules  
- Enforcing boundaries  
- Checking purity and determinism rules  

### You do NOT use it for:

- Build sequencing  
- Versioning  
- Schema evolution  
- Runtime orchestration  

Document 2 is the **physical architecture**.

---

# 📄 DOCUMENT 3 — Operational Architecture (Full System Roadmap)

### Summary  
This is the **complete build plan** for Phase 1 and early Phase 2.

It defines:

- Build order  
- Canonicalization  
- Determinism constraints  
- Snapshot → envelope pipeline  
- Graph construction  
- Invariants  
- Degraded mode  
- Schema versioning  
- Golden tests  
- Runtime orchestration  
- DLQ & retry logic  
- Phase 2 monorepo migration  
- Shadow analyzers  
- Enterprise roadmap  

### You use it for:

- Implementing features  
- Planning sprints  
- Ensuring correctness  
- Avoiding architectural drift  
- Reviewing complex PRs  
- Coordinating CLI + engine + runtime behavior  

### You **do not** use it for:

- Deciding folder structure (Document 2)  
- Deciding when to migrate repos (Document 1)  

Document 3 is the **operational heart of ArcSight**.

---

# 🔥 WHICH DOCUMENT IS THE SOURCE OF TRUTH DURING IMPLEMENTATION?

### During Phase 1 (now):

**Primary (authoritative):**  
➡️ **Document 3 — Full System Roadmap**  
"Build exactly this, in exactly this order."

**Secondary:**  
➡️ Document 2 — Folder Scaffold  
"Place code here. Follow these rules."

**Reference only:**  
➡️ Document 1 — Strategic  
"When do we migrate to monorepo?"

---

# 🔥 WHICH DOCUMENT DOES NEW ENGINEERS READ FIRST?

1️⃣ Document 4 — Usage Model  
2️⃣ Document 1 — Strategic  
3️⃣ Document 2 — Structure  
4️⃣ Document 3 — Roadmap  

This gives them:

- orientation  
- understanding of the system's philosophy  
- where code goes  
- how the system works end to end  

---

# 🔥 WHEN DO YOU RETURN TO EACH DOCUMENT?

### Every time you write code → Document 2 + Document 3  

### Every time you plan a sprint → Document 3  

### Every time you question folder structure → Document 2  

### Every time you consider monorepo → Document 1  

### Every time a new engineer joins → Document 4  

---

# 🧩 HOW THE DOCUMENTS AVOID CONTRADICTION

Because they sit on different levels:

- Document 1 = "WHEN do we restructure?"  
- Document 2 = "WHERE does the code live?"  
- Document 3 = "WHAT should be implemented and HOW?"  
- Document 4 = "HOW should we use the documents?"  

There is **no overlap**, therefore no conflict.

---

# 🎯 FINAL GUIDANCE FOR DAILY WORKFLOW

### Every morning during Phase 1:

1. Open **Document 3** → choose next subsystem  
2. Open **Document 2** → place files in correct location  
3. Ignore Document 1 completely during coding  
4. Use Document 4 if confused about which doc to consult  

### This keeps execution clean and prevents architectural drift.

---

# 🏁 SUMMARY

Document 4 completes your architecture set by defining:

- How to read the docs  
- When to use each  
- Which is authoritative now  
- How the layers fit together  
- How to onboard new engineers  
- How to avoid decision fatigue  

You now have a full, FAANG-grade documentation hierarchy.

---

## References

- [Strategic Architecture](./01-decision-phase1-vs-phase2.md)
- [Physical Architecture](./02-phase1-folder-scaffold.md)
- [Operational Architecture](./03-full-system-roadmap.md)

