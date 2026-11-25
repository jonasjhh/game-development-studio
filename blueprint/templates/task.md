---
# Source for the GitHub issue form (.github/ISSUE_TEMPLATE/task.yml is generated from this).
type: task
labels: [product:<framework|slug>, tier:<T0|T1|T2>, workflow:<id>]
---

## Summary
<One sentence: what outcome is wanted.>

## Product / scope
<framework | games/<slug>> — module(s): <names>

## Tier
<T0 | T1 | T2> — justification: <why this tier; reviewer may raise it>

## Workflow
<feature | bugfix | refactor | harvest | research | docs | …>

## Spec / contract
- Spec: <path to FS-/CS- document, or "n/a (T0)">
- Acceptance criteria covered by this task: <FS-foo-012/AC-1, AC-2>

## Context pointers
<Files, ADRs, prior issues the agent should read beyond `gdf context` output.>

## Constraints
- Must not touch: <paths/modules>
- Deadline / budget: <optional; default from workflow>
- Parallelism: <can run in parallel with #… | must follow #…>

## Definition of done
- [ ] All listed ACs have passing verification with evidence in the run report
- [ ] `gdf verify --scope changed --level full` green
- [ ] Independent review approved
- [ ] Docs + changelog fragment updated
- [ ] (if applicable) Playtest verdict recorded
- [ ] (if applicable) Owner approval recorded

## For bugs (delete otherwise)
- Observed: …
- Expected: …
- Reproduction: <replay file / scenario / steps>
- Build / framework version / sim fingerprint: …
