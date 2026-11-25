---
id: FS-<scope>-<NNN>          # e.g. FS-foo-012 (game) or FS-fw-007 (framework tooling)
title: <short imperative title>
type: feature-spec
status: draft                 # draft | proposed | approved | superseded
version: 0.1.0
owner: <product-owner | planner>
scope: <games/<slug> | framework>
tier: <T1 | T2>
issue: <GitHub issue number>
supersedes: ~
related: []                   # ADR IDs, rule IDs, other specs
---

# <Title>

## 1. Problem and intent
<What player/user problem or opportunity does this address? Why now? 2–5 sentences.>

## 2. Scope
**In scope:**
- …

**Out of scope (explicitly):**
- …

## 3. Design
<Behaviour description from the player's/consumer's perspective. Link design docs. For game features: which pillar this serves.>

### 3.1 Simulation impact
- New/changed commands: …
- New/changed sim state (and save-format impact): …
- Determinism considerations: …   <!-- FW-DET rules that apply -->

### 3.2 Presentation impact
- Scenes/UI/camera/input changes: …
- Perspectives affected: …

### 3.3 Content/data
- New content definitions and stable IDs: …

## 4. Acceptance criteria
Each AC is observable, has concrete values and names its verification method
(`unit | scenario | determinism | perf | screenshot | playtest | review`).

| ID | Given / When / Then | Method |
|---|---|---|
| AC-1 | **Given** … **When** … **Then** … | scenario |
| AC-2 | … | unit |
| AC-3 | **Given** two clients in lockstep… **Then** state hashes are identical for 10 000 ticks | determinism |
| AC-4 | … feels responsive: placement preview follows cursor without visible lag at 60 fps | playtest |

## 5. Non-functional requirements
- Performance budget: <e.g. ≤0.3 ms per tick at 2 000 entities on reference PC>
- Accessibility / localization: <all strings via tr(); colour-blind safe indicators …>
- Save compatibility: <migration needed? old-save fixture?>
- Multiplayer: <commands serialisable; no host-only logic …>

## 6. Dependencies and risks
- Framework modules/version required: …
- Open questions (must be empty before `approved`): …
- Risks: …

## 7. Reuse assessment (D1)
- Is any part a harvest candidate? <module name, likely second consumer>
- If built game-local: module boundary planned as `modules/<name>/`
- Reuse waiver needed? <no | yes → ADR-<slug>-NNNN>

## 8. Approval
| Role | Name | Date | Verdict |
|---|---|---|---|
| Planner | | | |
| Owner (T2 / creative) | | | |
