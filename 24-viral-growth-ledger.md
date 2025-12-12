# Viral Growth Ledger

**Status:** Planning-Only  
**Implementation:** Explicitly forbidden until Phase 2.5  
**Purpose:** Strategic planning document to prevent scope creep and preserve Phase 1/2 determinism

---

## 1. Purpose

This document tracks future viral and sharing mechanisms for ArcSight.

**Critical:** These features are **explicitly excluded** from Phase 1 and Phase 2 implementation.

Their inclusion is **time-gated** to preserve:
- Determinism guarantees
- Contract stability
- Engine correctness
- User trust

This document serves as:
- A guardrail against scope creep
- A strategic roadmap for growth features
- A reference for investor/advisor conversations
- Proof of intentional growth strategy vs. accidental discovery

**Why This Works:**

- Keeps Phase 1 & 2 pure and deterministic
- Prevents "just slipping this in" scope creep
- Provides long-term competitive advantage
- Strengthens investor/advisor conversations
- Demonstrates intentional growth strategy

---

## 2. Scope

This document governs:

### Included

- Viral feature catalog and phase-based activation plan
- Guardrails and hard rules for viral features
- Phase gates and implementation restrictions
- Review and ownership requirements

### Not Included

- Implementation details (deferred to Phase 2.5+)
- Technical specifications (deferred to Phase 2.5+)
- Engine or contract changes (viral features are additive only)

---

## 3. Definitions & Terms

**Viral Feature**  
Any mechanism that encourages sharing, attribution, or organic growth of ArcSight usage.

**Phase Gate**  
A milestone that must be reached before certain features can be implemented.

**Passive Viral**  
Viral mechanisms that do not require network calls, user interaction, or affect analysis results.

**Active Viral**  
Viral mechanisms that require infrastructure, user interaction, or external services.

**Opt-In**  
All viral features must be explicitly enabled by users; no default viral behavior.

---

## 4. Rules / Contract

### 4.1 Principles

All viral features MUST:

- ✅ Be opt-in (explicit user consent required)
- ✅ Never affect analysis results
- ✅ Never affect determinism
- ✅ Never mutate snapshots
- ✅ Never block CI
- ✅ Be removable without breaking contracts
- ✅ Be additive only (no breaking changes)

### 4.2 Phase-Based Activation Plan

#### 🔒 Phase 1 — Forbidden

**Status:** ❌ **Do not implement**

**Forbidden Features:**
- No branding
- No sharing
- No external links
- No telemetry
- No badges
- No prompts
- No attribution in outputs

**Reason:** Engine trust > growth.

**Enforcement:** Any PR attempting to add viral features in Phase 1 MUST be rejected.

#### ⚠️ Phase 2 — Preparation Only

**Status:** ⚠️ **Document, do not ship**

**Allowed (Structure Only):**
- CLI output structure that can later host footers
- PR summary layout that can later host attribution
- Snapshot metadata fields reserved but unused
- Extension fields reserved for future viral data

**Explicitly Forbidden:**
- Any user-visible viral output
- Any network calls
- Any external service integration
- Any branding or attribution display

**Purpose:** Prepare structure without implementing behavior.

#### ⭐ Phase 2.5 — Passive Viral Hooks

**Status:** ✅ **Allowed**

**Low-risk, non-intrusive features:**
- PR comment footer (passive, non-blocking)
- CLI "share hint" (no network calls, local only)
- README badge templates (static, user-added)
- Attribution in PR summaries (opt-in)

**Requirements:**
- Must be opt-in
- Must not change analysis behavior
- Must not affect determinism
- Must not require network calls
- Must be removable without breaking contracts

#### 🚀 Phase 3 — Active Viral Loops

**Status:** ✅ **Allowed**

**Active viral mechanisms:**
- `npx arcsight share` command
- Shareable report links
- GitHub Issue creation
- Slack notifications
- Repo architecture score pages
- Public dashboard links

**Requirements:**
- All opt-in
- All require explicit user consent
- All must not affect analysis results
- All must be removable

#### 🌐 Phase 4 — Organizational Virality

**Status:** ✅ **Allowed**

**Organizational growth features:**
- Org dashboards
- Monthly reports
- Rulepack marketplace
- Team invites
- Cross-repo analytics
- Enterprise features

**Requirements:**
- All opt-in
- All require authentication
- All must preserve determinism
- All must not affect core engine

### 4.3 Viral Mechanisms Catalog

| Feature | Phase | Risk | Notes |
|---------|-------|------|-------|
| PR comment footer | 2.5 | Low | Passive, non-blocking, opt-in |
| CLI share hint | 2.5 | Low | Local only, no network calls |
| README badge | 2.5 | Low | Static template, user-added |
| Shareable links | 3 | Medium | Requires infrastructure, opt-in |
| Slack alerts | 3 | Medium | Opt-in, requires webhook setup |
| Repo score pages | 3 | Medium | Snapshot-backed, public opt-in |
| Org dashboards | 4 | High | Multi-repo, requires auth |
| Marketplace | 4 | High | Monetization, requires payment infra |
| Team invites | 4 | High | Requires org management |

### 4.4 Guardrails (Hard Rules)

Viral features MUST NOT:

- ❌ Affect analyzer output
- ❌ Change envelopes
- ❌ Add fields to frozen contracts (Phase 1)
- ❌ Introduce nondeterminism
- ❌ Require login for CLI use
- ❌ Block CI execution
- ❌ Modify snapshots
- ❌ Change analysis results
- ❌ Break backward compatibility

**Violations of this section block merge.**

### 4.5 Implementation Requirements

All viral features MUST:

- Be documented in this ledger before implementation
- Reference this document in PRs
- Pass phase gate review
- Include opt-in mechanism
- Be removable without breaking contracts
- Preserve determinism guarantees

---

## 5. Determinism Impact

Viral features MUST NOT affect determinism:

- No timestamps in viral outputs (unless explicitly opt-in and excluded from signatures)
- No random values in viral features
- No network-dependent behavior in core engine
- No user-specific data in deterministic outputs

Viral features are **presentation layer only** and must not touch core analysis.

---

## 6. Runtime / Engine Boundary Impact

**Engine MUST:**

- Remain completely free of viral features
- Never include branding, attribution, or sharing logic
- Never make network calls for viral purposes
- Never depend on viral infrastructure

**Runtime MAY:**

- Add viral features in presentation layer (Phase 2.5+)
- Include opt-in attribution in PR comments
- Provide shareable links (external to engine)
- Display badges and branding (presentation only)

**Boundary Rule:**

Viral features live **outside** the engine. They are runtime/presentation concerns only.

---

## 7. Versioning Impact

Viral features MUST NOT:

- Require `schemaVersion++` (unless adding optional extension fields in Phase 2+)
- Require `analyzerVersion++` (unless changing analysis behavior, which is forbidden)
- Break backward compatibility

Viral features are **additive only** and must not affect versioning.

---

## 8. Testing Requirements

Viral features MUST have:

- Opt-in/opt-out tests
- Removal tests (verify removal doesn't break contracts)
- Determinism tests (verify no nondeterminism introduced)
- Integration tests (verify no engine impact)

---

## 9. Ownership & Review

**Review Requirements:**

- Any viral feature PR MUST reference this document
- Any earlier-than-allowed implementation is **rejected**
- Phase gates are enforced by review, not discretion
- Violations of phase gates block merge

**Ownership:**

- Product team owns viral feature roadmap
- Engineering team enforces phase gates
- CTO/Architect approves phase transitions

**Phase Gate Enforcement:**

- Phase 1: Zero viral features allowed
- Phase 2: Structure preparation only, no user-visible features
- Phase 2.5: Passive viral hooks only
- Phase 3: Active viral loops (requires Phase 2.5 complete)
- Phase 4: Organizational virality (requires Phase 3 stable)

---

## 10. Cross-References

- [Contract Authority Index](./00A-phase1-contract-authority-index.md) — Phase 1 contract classification
- [Phase 1 Implementation Plan](./23-phase1-implementation-plan.md) — Phase 1 implementation roadmap
- [Determinism Contract](./07-determinism-contract.md) — Determinism requirements
- [Runtime ↔ Engine Contract](./10-runtime-and-engine-contract.md) — Boundary rules
- [Envelope Format Spec](./15-envelope-format-spec.md) — Envelope structure (viral features must not modify)

---

## 11. Change Log (Append-Only)

**v1.0.0** — Initial viral growth ledger

- Established phase-based activation plan
- Defined guardrails and hard rules
- Cataloged viral mechanisms by phase
- Enforced Phase 1/2 restrictions
- Defined review and ownership requirements

---

## 12. Strategic Context

### Why This Is a Power Move

**Most dev tools:**
- ❌ Add virality ad hoc
- ❌ Break trust early
- ❌ Confuse users
- ❌ Pollute outputs
- ❌ Lose credibility

**ArcSight approach:**
- ✅ Engine first
- ✅ Trust first
- ✅ Growth later
- ✅ Intentional compounding

**Historical Precedent:**

This is exactly how successful dev tools became defaults:
- **Sentry:** Reliability first, growth features later
- **ESLint:** Correctness first, ecosystem later
- **Prettier:** Determinism first, adoption later
- **Renovate:** Trust first, viral features later

**Investor/Advisor Value:**

This document demonstrates:
- Strategic thinking
- Discipline in execution
- Long-term vision
- Intentional growth strategy
- Trust-first approach

**Competitive Advantage:**

By deferring viral features until engine trust is established, ArcSight:
- Builds stronger user trust
- Avoids early credibility issues
- Creates intentional growth loops
- Differentiates from ad-hoc tools
- Positions for sustainable growth

---

## 13. Implementation Checklist (Future)

### Phase 2.5 Readiness

- [ ] Phase 1 complete (zero drift, golden stable)
- [ ] Phase 2 complete (monorepo migrated, stable)
- [ ] Engine trust established
- [ ] User base validated
- [ ] Contract stability confirmed

### Phase 2.5 Implementation

- [ ] PR comment footer (opt-in)
- [ ] CLI share hint (local only)
- [ ] README badge templates
- [ ] Attribution in PR summaries (opt-in)

### Phase 3 Readiness

- [ ] Phase 2.5 passive features validated
- [ ] Infrastructure for shareable links ready
- [ ] User consent mechanisms in place
- [ ] Opt-in/opt-out flows tested

### Phase 3 Implementation

- [ ] `npx arcsight share` command
- [ ] Shareable report links
- [ ] GitHub Issue creation
- [ ] Slack notifications
- [ ] Repo score pages

### Phase 4 Readiness

- [ ] Phase 3 active features validated
- [ ] Org management infrastructure ready
- [ ] Authentication system in place
- [ ] Multi-repo analytics validated

### Phase 4 Implementation

- [ ] Org dashboards
- [ ] Monthly reports
- [ ] Rulepack marketplace
- [ ] Team invites

---

**Remember:** This document is a **guardrail**, not a roadmap. Phase gates are **enforced**, not suggestions.

