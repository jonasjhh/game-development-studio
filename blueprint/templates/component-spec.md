---
id: CS-<module>               # e.g. CS-gdf_sim
title: <module name> component specification
type: component-spec
status: draft
version: 0.1.0                # spec version (module version = framework version)
owner: <planner (framework-architect mode) → product-owner>
scope: <framework | games/<slug>>
module: <gdf_xxx | game-local module name>
maturity: <game-local | experimental | approved | stable>
layer: <L0-foundation | L1-simulation | L2-session | L3-presentation | tooling>
deterministic: <true | false>
issue: <number>
related: []
---

# <Module> — Component Specification

## 1. Responsibility
<One paragraph. One responsibility. What it explicitly does NOT do.>

## 2. Consumers
| Consumer | Uses | Since |
|---|---|---|
| games/<slug> | … | v0.x |

(Experimental requires ≥1 real consumer; Stable requires ≥2 games or 1 shipped game.)

## 3. Public interface (`api/`)
For each public class:

### `Gdf<Name>`
- Purpose: …
- Lifecycle/ownership: <who creates it, who frees it>
- Methods:
  ```gdscript
  func schedule(command: GdfCommand, tick: int) -> GdfResult
  ```
  Pre-conditions · post-conditions · errors (GdfResult codes) · complexity
- Signals/events emitted: <presentation-facing only; sim uses gdf_events>
- Thread-safety: …

### Kernel boundary (if performance-sensitive)
- Interface: `Gdf<Name>Kernel` — inputs/outputs as packed arrays, called once per tick
- Reference implementation: GDScript (required)
- Native implementations allowed: <GDExtension | none>; must pass the same tests and produce identical hashes

## 4. Dependencies
| Depends on | Layer | Why |
|---|---|---|
| gdf_core | L0 | logging, results |

Forbidden (determinism, if `deterministic: true`): physics, NavigationServer, RandomNumberGenerator, Time, OS, SceneTree, float state.

## 5. Data and persistence
- State owned: …
- Serialisation (gdf_serial schema id + version): …
- Migration policy: …

## 6. Acceptance criteria
| ID | Criterion | Method |
|---|---|---|
| AC-1 | … | unit |
| AC-2 | Identical hash sequence over 10 000 ticks on Windows and Linux | determinism |
| AC-3 | `tick()` ≤ <budget> ms p99 at <N> entities on reference hardware | perf |

## 7. Performance budget
<Reference hardware, scenario, numbers. Recorded per release in scenarios/benchmarks.>

## 8. Compatibility and versioning
- Breaking-change policy at current maturity: …
- Deprecated APIs and removal version: …

## 9. Demo
`demo/<name>.tscn` — what it shows, how to run (`gdf demo <module>`).

## 10. Harvest record (if harvested from a game)
- Origin: games/<slug>/project/modules/<name> @ <commit>
- Generalisations made: …
- Game-specific parts left behind: …

## 11. Promotion history
| Date | From → To | Evidence | Approved by |
|---|---|---|---|
