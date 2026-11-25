---
id: ADR-<SCOPE>-<NNNN>        # ADR-ORG-0001 | ADR-FW-0001 | ADR-<slug>-0001 | ADR-HAR-0001
title: <decision in a short phrase>
type: adr
status: proposed              # proposed | approved | rejected | superseded | deprecated
date: YYYY-MM-DD
deciders: [product-owner]
drafted-by: <role/run-id or human>
scope: <org | framework | games/<slug> | harness>
tier: T2
supersedes: ~
superseded-by: ~
related: []                   # rule IDs changed, specs, research reports
kind: <standard | reuse-waiver | network-model | exception-backing>
---

# <Title>

## Context
<Forces at play: requirements, constraints, evidence (link research, benchmarks, run reports). Facts, not opinions.>

## Decision drivers
- …

## Options considered
### Option A — <name>
- Description · Pros · Cons · Cost · Reversibility

### Option B — <name>
- …

### Option C — Do nothing
- …

## Decision
<The chosen option, stated as a rule: "We will …". Include the scope it applies to.>

## Consequences
- Positive: …
- Negative / accepted costs: …
- Follow-up work (issues): …
- Rules created/changed in this PR: <ORG-…/FW-…; the rule file MUST be updated in the same PR>

## Verification
<How we will know the decision holds / when to revisit. Measurable trigger, e.g. "revisit if tick p99 > 8 ms at 10k entities".>

## Reuse waiver (only for kind: reuse-waiver)
- Module/code concerned: …
- Why reuse investment is not justified now: …
- What generalisation would require later: …
- Review date: …

## Approval
| Name | Date | Verdict | Notes |
|---|---|---|---|
