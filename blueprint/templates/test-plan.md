---
id: TP-<spec-id>              # TP-FS-foo-012
title: Test plan for <spec title>
type: test-plan
status: draft
version: 0.1.0
owner: verifier
spec: <path to spec>
issue: <number>
---

# Test plan: <spec title>

## 1. Traceability
| AC | Method | Test / scenario ID | Location | Automated? |
|---|---|---|---|---|
| AC-1 | scenario | SCN-foo-placement-01 | scenarios/placement_01.tres | yes |
| AC-2 | unit | test_rejects_overlap | tests/building/test_placement.gd | yes |
| AC-3 | determinism | DET-foo-placement | scenarios/placement_01 (hash run, Win+Linux) | yes |
| AC-4 | playtest | PT-foo-012 | checklist §5 | human |

Every AC appears at least once. Tests carry `# covers <spec-id>/AC-n`.

## 2. Test data and fixtures
- Seeds: …
- Content fixtures: …
- Save fixtures (old versions for migration): …
- Golden hashes: <new | unchanged | changed → justification>

## 3. Scenarios
### SCN-<id>
- Setup: <initial state / save / seed>
- Command script: <tick → command list>
- Duration: <ticks>
- Assertions: <state predicates at ticks>
- Evidence: <report JSON, screenshots at ticks …>

## 4. Negative and edge cases
- …

## 5. Playtest checklist (for owner)
- Build: <how to launch (`gdf play games/<slug> --scenario …`)>
- Questions:
  1. <specific, answerable: "Can you place 10 buildings in under 30 seconds without mis-clicks?">
- Verdict recorded on issue as: `playtest: pass | fail | iterate` + notes

## 6. Performance checks
| Benchmark | Budget | Reference hardware |
|---|---|---|

## 7. Out of scope / known gaps
- …

## 8. Results (filled by verification run)
| Run ID | Date | Pass/Fail | Notes |
|---|---|---|---|
