---
id: ROLE-<name>               # ROLE-implementer
title: <Role name>
type: role
status: draft
version: 0.1.0
owner: product-owner
permission_profile: <harness/permissions/<profile>.yaml>
default_executor:
  adapter: <claude | codex | gemini | opencode | …>
  model_class: <strong-reasoning | coding | fast | web-research>   # resolved to a concrete model in harness config
independence: <e.g. "must differ in vendor from the implementer of the change under review">
modes: [<mode-a>, <mode-b>]
---

# Role: <Name>

## Purpose
<One paragraph: why this role exists and which failure it prevents.>

## Responsibilities
- …

## Inputs (always provided by the run contract)
- Role definition (this file), workflow step, issue, spec, context manifest from `gdf context`

## Outputs
| Output | Location | Format |
|---|---|---|
| Run report | `runs/<run-id>/report.json` | `harness/schemas/run-report.schema.json` |
| … | … | … |

## Allowed actions
- Files it may write: …
- Commands it may run: …
- Network: …

## Forbidden actions
- … (each tied to a rule ID, e.g. "Edit verifier-owned acceptance tests — ORG-AGT-007")

## Procedure
1. Run `gdf context <issue>`; read the manifest; cite applicable rule IDs.
2. …
n. Write the run report; set status `done | blocked | failed`.

## Modes
### <mode-a>
- Differences from the base procedure: …

## Definition of done
- [ ] Every AC addressed is listed with evidence
- [ ] `gdf verify --scope changed` green (if role writes code/tests)
- [ ] Docs/changelog fragment updated
- [ ] No open instruction conflicts

## Escalation
Stop and escalate (issue comment + label `needs-owner`) when:
- instructions conflict (cite both sources),
- the spec is ambiguous or missing ACs,
- the task requires a protected path or a tier above the declared one,
- the budget is ≥80% consumed without completion.

## Quality signals (tracked in reports)
- <e.g. first-pass review acceptance rate, escaped defects>
