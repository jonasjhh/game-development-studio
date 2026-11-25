---
id: BLUEPRINT
title: Game Development Organization Blueprint
version: 1.0.0-draft
status: draft            # becomes "approved" after owner sign-off
owner: product-owner     # the human owner
date: 2026-11-01
supersedes: none
rationale: blueprint/stage-1-analysis.md
---

# Game Development Organization Blueprint v1.0

> This document defines the **target design**. The *reasons* behind it are in [stage-1-analysis.md](stage-1-analysis.md): alternatives considered, research and sources, and the owner's answers. This document refers to it rather than repeating it.
>
> Tags: **[E]** established by the owner · **[R]** recommendation (in force unless the owner objects) · **[A]** requires owner approval before it takes effect.

---

## 1. Executive summary

We are building a **one-person studio where AI agents do the implementation** and the human owner holds product, creative and architectural authority. The studio consists of four independently managed areas in one Git monorepo:

1. **Governance**: the organisation's law. Numbered, machine-checkable rules in three levels (organisation → framework → game).
2. **Framework (GDF)**: a versioned product of reusable Godot 4 capabilities. Games consume it as **vendored, pinned, checksum-verified releases**.
3. **Harness**: the AI production system. Roles, workflows, templates, skills, tooling (`gdf` CLI), vendor adapters and evaluations.
4. **Games**: independent products, each a separate Godot project with its own governance, backlog and releases.

The design has six load-bearing decisions:

- **Harvest, don't predict [E].** Gameplay systems are born inside games, written to framework standards from the start, and harvested into the framework when reuse is demonstrated. The framework begins with *foundations only*.
- **Deterministic simulation core [E→derived].** Multiplayer (co-op and competitive, lockstep-style) is required. Simulation state is therefore integer/fixed-point, command-driven, hashable and replayable. It is strictly separated from presentation (which handles 3D, isometric, first- and third-person). The same property makes agent verification cheap: headless, deterministic and reproducible.
- **GDScript first, with kernel boundaries [E].** Hot paths sit behind batch-oriented interfaces, so they can move to GDExtension or C# later (by ADR) without touching callers.
- **Provider independence by owning the durable layers [R].** Instructions (`AGENTS.md`), skills (`SKILL.md`), tools (MCP + plain CLI), roles, workflows and specs are ours and vendor-neutral. Vendor CLIs (Claude Code, Codex CLI, Gemini CLI…) are **replaceable executors** behind thin adapters.
- **Independent verification [E/R].** Acceptance tests are derived from the spec before implementation. A reviewer from a different model/vendor reviews. `gdf verify` is the single definition of "green". Human playtest gates whatever automation cannot judge.
- **Staged, evidence-driven infrastructure [R].** No agent framework and no services. Scripts first; automation only where friction has been observed.

GitHub hosts the repo, CI (Actions) and work items (Issues/Projects) **[E]**. Specs and ADRs, the contracts, stay in the repo.

---

## 2. Design principles

Each principle has an ID. The rule files in `governance/rules/` cite these principles and turn them into checkable rules.

| ID | Principle | How it is enforced |
|---|---|---|
| P1 | **The repository is the source of truth.** Not conversation history, not issue comments, not an agent's memory. Decisions that matter are written down in the repo. | Workflows end by writing artefacts; reviewer rejects "decided in chat" |
| P2 | **Specs are contracts.** Every T1+ change traces to a spec with numbered acceptance criteria (`AC-n`). | PR template + `gdf check trace` |
| P3 | **No self-certification.** The author of a change is never the only judge of its correctness. | Reviewer role ≠ implementer instance (different vendor/model preferred); CI gates |
| P4 | **Verification over generation.** "It compiles" and "the agent says done" are not evidence. Evidence is test results, scenario reports, hashes, screenshots. | Run report must include evidence; PR gate |
| P5 | **Harvest, don't predict.** Abstractions are extracted from real consumers. Reuse-ready structure always; reuse investment by decision. | Module rules apply in games too; reuse-waiver ADRs |
| P6 | **Simulation is sovereign.** Simulation state changes only through commands on ticks, using deterministic math. Presentation observes. | Import-ban lint; determinism CI gate |
| P7 | **Dependencies point one way.** Games → framework; presentation → simulation → foundation. Never upward, never sideways between games. | `gdf check deps` |
| P8 | **Small always-on context, deep on-demand knowledge.** Entry instruction files are routing documents; detail is loaded when relevant. | Size lint on `AGENTS.md` files |
| P9 | **Own the durable layers, rent the executors.** Nothing authoritative lives only in vendor-specific files. | Generated adapters + sync check |
| P10 | **Gates scale with risk.** T0/T1/T2 tiers; the human approves what matters, not everything. | Tier field in issue/PR; reviewer can raise tier |
| P11 | **Every decision is recorded with its scope.** ADRs live with the product they govern. | ADR template; doc lint |
| P12 | **Infrastructure must earn its place.** New tooling, services or agents require an observed problem and a recorded decision. | T2 for harness architecture changes |

---

## 3. Architectural overview

```
                         ┌──────────────────────────────────────────┐
                         │                 OWNER                    │
                         │  product · creative · approval · playtest│
                         └───────────────┬──────────────────────────┘
                                         │ issues, approvals, ADR sign-off
┌────────────────────────────────────────▼─────────────────────────────────────────┐
│ HARNESS (AI production system)                                                   │
│  roles · workflows · templates · skills · gdf CLI · vendor adapters · evals      │
│  executes agent runs in git worktrees  →  PRs + run reports                      │
└───────┬───────────────────────────────┬────────────────────────────────┬─────────┘
        │ reads/obeys                   │ develops                       │ develops
┌───────▼────────┐            ┌─────────▼──────────┐   vendored   ┌──────▼──────────────┐
│  GOVERNANCE    │  L1 rules  │  FRAMEWORK (GDF)   │──release────▶│  GAMES               │
│  L1 org law    │──────────▶ │  L2 rules          │  + lockfile  │  L3 rules            │
│  rules, ADRs,  │            │  workbench project │              │  one Godot project   │
│  exceptions    │            │  modules, releases │◀─harvest─────│  each; game-local    │
└────────────────┘            └────────────────────┘   (PR)       │  modules             │
                                                                  └─────────────────────┘
                       KNOWLEDGE (non-normative): godot notes, patterns, research, lessons
```

**Runtime architecture inside a game:**

```
 Presentation layer (Nodes, scenes, float math, rendering, audio, UI, camera rigs per perspective)
        ▲ events / read-only views                     │ commands (serialisable)
        │                                              ▼
 Simulation layer (RefCounted/typed data, fixed-point, ticks, owned PRNG/pathfinding, state hash)
        ▲                                              ▲
        │ uses                                         │ lockstep command exchange
 Foundation (gdf_core, gdf_fixed, gdf_rng, gdf_sim, gdf_serial, gdf_content)   gdf_net
```

---

## 4. Working-folder structure

Repository name is **[A]** (placeholder `studio/`). Prefix `gdf` = "game development framework" and is used for the CLI and module names. Class prefix `Gdf`.

```
studio/
├── AGENTS.md                      L1 entry: non-negotiables, routing, commands (≤150 lines)
├── CLAUDE.md                      "@AGENTS.md" + Claude-specific notes (generated)
├── README.md                      human orientation
├── .github/
│   ├── ISSUE_TEMPLATE/            issue forms: feature, component, bug, research, task (generated from harness/templates)
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── CODEOWNERS                 governance/**, harness/roles/** → owner
│   └── workflows/                 CI: verify.yml, determinism.yml, release-framework.yml, canary.yml
├── .agents/ .claude/ .codex/      GENERATED vendor adapter output (skills, settings). Never hand-edited.
├── tools/godot/                   gitignored: engine binaries resolved by `gdf godot`
├── governance/                    L1
│   ├── constitution.md
│   ├── rules/                     ORG-*.md rule files (engineering, quality, agents, security, docs, decisions)
│   ├── decisions/                 ORG ADRs (ADR-ORG-0001…)
│   ├── exceptions/                EXC-NNN.md approved deviations with expiry
│   └── glossary.md                canonical terms (org-wide)
├── harness/
│   ├── AGENTS.md
│   ├── roles/                     planner.md implementer.md verifier.md reviewer.md researcher.md
│   ├── workflows/                 <workflow>.yaml + <workflow>.md (playbook)
│   ├── skills/                    canonical SKILL.md packages (godot-testing, write-spec, …)
│   ├── templates/                 the 8 templates (+ issue form sources)
│   ├── permissions/               permission profiles per role (vendor-neutral YAML)
│   ├── adapters/                  claude/ codex/ gemini/ … : invocation + config generators
│   ├── tools/gdf/                 Python package for the `gdf` CLI (uv project)
│   ├── schemas/                   JSON Schemas: run-report, module.toml, framework.lock, frontmatter
│   ├── evals/                     harness regression tasks + expected outcomes
│   └── decisions/                 harness ADRs
├── framework/
│   ├── AGENTS.md                  L2 entry
│   ├── governance/                FW-*.md rules: architecture, API conventions, deps, determinism, compat
│   ├── product/                   vision.md, roadmap.md (backlog = GitHub Project "Framework")
│   ├── architecture/              overview.md, layers.md, module-catalogue.md
│   ├── decisions/                 ADR-FW-NNNN
│   ├── workbench/                 the framework's Godot project (current root project moves here)
│   │   ├── project.godot  .godot-version
│   │   ├── addons/gdf_<module>/   module = code + module.toml + README.md + spec.md + tests/ + demo/
│   │   ├── addons/gdUnit4/        test framework (third-party, pinned)
│   │   └── scenarios/             cross-module scenarios, perf benchmarks, golden hashes
│   ├── releases/                  CHANGELOG.md, notes/vX.Y.Z.md, compat-matrix.md
│   └── THIRD_PARTY.md
├── games/
│   └── <slug>/
│       ├── AGENTS.md              L3 entry
│       ├── governance/            game-constitution.md (vision, pillars), GAME-<slug>-*.md rules
│       ├── product/               roadmap.md (backlog = GitHub Project "<Game>")
│       ├── design/                GDD sections, design pillars, references, playtest notes
│       ├── specs/                 feature specs (FS-<slug>-NNN)
│       ├── decisions/             ADR-<slug>-NNNN (incl. reuse waivers, network model)
│       ├── art-direction/         style guides, asset provenance register
│       ├── releases/              CHANGELOG.md, notes
│       ├── framework.lock         pinned GDF version, module list, checksums
│       └── project/               the game's Godot project
│           ├── project.godot  .godot-version  export_presets.cfg
│           ├── addons/gdf_*/      VENDORED (read-only; CI-verified against lock)
│           ├── modules/<name>/    game-local modules (same anatomy as framework modules)
│           ├── src/               game composition: scenes, presentation, glue
│           ├── content/           data definitions (.tres / data files) with stable IDs
│           ├── assets/            art/audio (LFS once adopted, §15), each with provenance metadata
│           ├── scenarios/         game scenarios, replays, golden hashes
│           └── tests/
├── knowledge/                     non-normative: godot/, patterns/, research/, lessons/
│   └── INDEX.md                   generated catalogue with one-line summaries
├── blueprint/                     this document + stage 1 (moves to governance/ on approval)
└── runs/                          gitignored: run reports, logs, screenshots, videos
```

Placement rules (enforced by `gdf check layout`):
- Normative content (`MUST`/`SHOULD`) lives only under `governance/`, `*/governance/`, specs and ADRs. `knowledge/` is advisory.
- A Godot project directory contains no planning documents. Planning documents contain no code.
- Vendored and generated directories carry a `.generated` marker file. CI rejects hand edits.

---

## 5. Governance hierarchy

| Level | Scope | Location | Rule ID prefix | Owner | Change approval |
|---|---|---|---|---|---|
| L1 Organisation | Everything | `governance/` | `ORG-<AREA>-NNN` | Owner | Owner (always T2) |
| L2 Framework | `framework/**` | `framework/governance/` | `FW-<AREA>-NNN` | Owner (Planner drafts) | Owner (T2) |
| L3 Game | `games/<slug>/**` | `games/<slug>/governance/` | `GAME-<slug>-<AREA>-NNN` | Owner (Planner drafts) | Owner for pillars/constraints; T1 for minor clarifications |
| (H) Harness | `harness/**` | `harness/` roles/workflows | `HAR-<AREA>-NNN` | Owner | Owner (T2) |

The harness is governed by L1 directly. It is a peer of the framework, not beneath it.

Initial L1 rule areas: `ENG` (engineering), `QUA` (quality), `AGT` (agent behaviour), `SEC` (security & permissions), `DOC` (documentation), `DEC` (decision authority), `GIT` (branching, commits, PRs), `AST` (assets & provenance).

Initial L2 rule areas: `ARCH` (layers, modules), `API` (naming, signals, errors), `DEP` (dependencies), `DET` (determinism), `CMP` (compatibility), `REL` (release), `TST` (testing).

Rule file format, one file per area:

```markdown
---
id: ORG-AGT
level: L1
status: approved
version: 1.2.0
owner: product-owner
---
## ORG-AGT-001 — Agents work only in isolated worktrees
**MUST.** An agent instance that writes files operates in its own git worktree on a branch `agent/<issue>-<slug>`.
Rationale: parallel safety; reviewable diffs.
Check: `gdf check worktree` (pre-run), branch protection (server).
```

---

## 6. Instruction inheritance and conflict resolution

### 6.1 Discovery (who sees what)

1. **Positional inheritance.** Agent CLIs load `AGENTS.md` from the repo root down to the working directory. The scope of an instruction is its directory. An agent in `games/foo/project/src/` gets root + `games/foo/`. An agent in `framework/workbench/` gets root + `framework/`.
2. **Entry files route; they don't legislate at length.** Each `AGENTS.md` contains: scope statement, top non-negotiables (rule IDs + one line each), commands, and links to the governing documents. Hard cap: 150 lines (`gdf check docs`).
3. **Explicit resolution.** `gdf context <path|issue>` prints:
   - the applicable instruction chain (files + versions);
   - all rule IDs in force for that scope;
   - active exceptions;
   - the role definition and workflow step;
   - the recommended context manifest (spec, ADRs, module READMEs, API docs).

   Every workflow begins with this command. The run report records an `instructions_digest` (hash of the resolved chain) so we can later tell which rules an agent was operating under.

### 6.2 Precedence

1. **L1 > L2/L3 > task instructions > agent judgement.** The harness is L1-adjacent: role definitions cannot relax L1.
2. **Lower levels may add or tighten only.** A lower-level rule must declare its relation to the rule above: `refines: ORG-QUA-004` (stricter or more specific) or `adds:` (new). Never `overrides:`.
3. **Deviations require an exception record** (`governance/exceptions/EXC-NNN.md`). It contains the rule ID, scope, justification, compensating controls, expiry date and owner approval. Expired exceptions fail CI.
4. **Within a level**: ADR (most recent approved) > spec > rule file > guideline. A newer ADR that changes a rule must also update the rule file in the same PR.
5. **Task instructions** (issue text) cannot authorise violating a rule. An issue that asks for a violation is escalated.

### 6.3 Conflict handling by agents

On detecting conflicting instructions, an agent **MUST stop the affected work**. It records the conflict in the run report (`status: blocked`, `blocker: instruction-conflict`, both sources cited) and comments on the issue. It must not choose silently. The Planner may propose a resolution (rule edit, exception or ADR). Only the owner approves.

### 6.4 Machine checks (`gdf check governance`)

- Unique rule IDs; every `refines:` target exists and is at a higher or equal level.
- Lower-level documents contain no override language (`override`, `instead of ORG-`, `ignore`) unless they cite a valid exception.
- Exceptions not expired; superseded documents not referenced as authority.
- Every approved document has an owner, status and version.

---

## 7. Framework product architecture

### 7.1 Product identity

GDF has its own vision (`framework/product/vision.md`), roadmap, backlog (GitHub Project "Framework" + label `product:framework`), ADRs, release train and changelog. Its customers are the games. Its success metric is **demonstrated reuse**: the number of games consuming each module, and harvests completed. Lines of framework code are not the metric.

### 7.2 Layers

| Layer | Purpose | May depend on | Determinism |
|---|---|---|---|
| **L0 Foundation** | Bootstrap, logging, config, results/errors, fixed-point, PRNG, serialisation, content registry | Godot core types only | Deterministic where it touches sim |
| **L1 Simulation** | Tick scheduler, commands, entity IDs, state hashing, sim-side spatial/pathfinding, (harvested) gameplay systems | L0 | **Strict**: no floats in state, no engine services |
| **L2 Session/Net** | Lockstep sessions, transports, replays, save/load orchestration | L0, L1 | Protocol-deterministic |
| **L3 Presentation** | Camera rigs per perspective, input→command mapping, interpolation, UI kit, debug overlay, audio helpers | L0–L2 (read-only views of L1) | None required |
| **Tooling** | Testkit, scenario runner, perf probes, editor plugins | Any (never shipped in release builds) | n/a |

### 7.3 Framework v0.1 module catalogue (foundations only)

| Module | Responsibility | Why it's a foundation (not speculative) |
|---|---|---|
| `gdf_core` | Bootstrap sequence, explicit service registry (a single autoload), structured logging, config, `GdfResult` errors | Every game; needed by all other modules |
| `gdf_fixed` | Fixed-point scalar/vector types, deterministic trig/sqrt tables, conversions to float for presentation | Required by lockstep [E] |
| `gdf_rng` | Seeded, integer-based PRNG streams, forkable per system | Required by lockstep [E] |
| `gdf_sim` | Fixed tick scheduler, ordered system pipeline, command queue (tick-scheduled), entity ID allocator, canonical state hash | The sovereign-simulation spine (P6) |
| `gdf_serial` | Canonical serialiser for sim state, schema versions, migration chain (saves + net state transfer) | Saves + late join share it |
| `gdf_content` | Data-driven definitions with stable string IDs, registry, validation, load order (mod-ready) | All target genres are content-heavy |
| `gdf_events` | Typed sim→presentation event stream (buffered per tick), no presentation→sim path | Decoupling of presentation |
| `gdf_debug` | In-game console, overlay, sim pause/step/speed, state inspector | Agents and owner need to see the sim |
| `gdf_testkit` | Scenario runner, scripted commands, replay record/playback, golden-hash assertions, screenshot capture, perf probes, JSON reports | Agent verification (P4) |
| `gdf_net` *(experimental, P3)* | Lockstep session (input delay, turn exchange, desync detection, resync via `gdf_serial`), ENet/WebRTC transports | Required by multiplayer [E]; spiked before game code depends on it |

Everything else from BOOTSTRAP §11 is a **harvest candidate**, for example navigation, agents/needs, economy, production, logistics, inventory, combat, progression, quests, UI kit and camera rigs. Each enters the framework only through the harvest workflow (§13.4).

### 7.4 Module anatomy

```
addons/gdf_sim/
├── module.toml        id, version (= framework version), maturity, layer, depends, godot = ">=4.7,<4.8", deterministic = true
├── README.md          purpose, public API summary, usage example, limitations
├── spec.md            component specification (template CS) incl. acceptance criteria
├── api/               public classes (class_name Gdf*)  ← the ONLY files other modules/games may reference
├── internal/          private implementation (lint: no external references)
├── tests/             gdUnit4 tests (stripped on vendoring)
└── demo/              minimal demo scene (stripped on vendoring)
```

### 7.5 API conventions (FW-API, summary)

- Public types are `class_name Gdf<Name>`, under `api/`. Everything under `internal/` is private (lint-enforced; GDScript has no visibility modifiers).
- **Static typing everywhere.** The GDScript warnings `untyped_declaration`, `unsafe_*` and `inferred_declaration` are configured as errors in the project settings baseline.
- Errors return `GdfResult` for expected failures. `assert` is for programmer errors. There are no silent `null` returns from public APIs.
- Signals are allowed in presentation. Simulation uses explicit ordered calls and the `gdf_events` buffer, because signal emission order is an implementation detail that can't be relied on for determinism.
- Autoloads: only `GdfCore`. Everything else is obtained through the service registry, which keeps test isolation possible.
- **Kernel boundary** for performance-sensitive code: a `Gdf*Kernel` interface operates on packed arrays once per tick. The GDScript reference implementation is required. Native implementations (GDExtension, or C# in games) must pass the same tests and produce identical hashes.

### 7.6 Maturity and promotion

| From → To | Criteria | Approval |
|---|---|---|
| (game-local) → **Experimental** | Harvest workflow complete: module anatomy, spec, tests, demo, no game-specific references, ≥1 consumer | T2 review, owner |
| Experimental → **Approved** | Used in production code by ≥1 game; coverage ≥ 80% lines for `api/` + `internal/`; docs complete; API review passed; perf budget documented | T2, owner |
| Approved → **Stable** | ≥2 consuming games or ≥1 shipped game; no breaking API change for 2 minor releases; deprecation policy applied | T2, owner |
| Any → **Deprecated** | Replacement exists or no consumers; removal no earlier than the next major | T2, owner |

Games opt into experimental modules explicitly in `framework.lock` (`allow_experimental = ["gdf_net"]`).

---

## 8. Game product architecture

### 8.1 Product structure

Each game has: a game constitution (vision, pillars, audience, non-goals), L3 rules, design documents, feature specs, ADRs, art direction with an asset provenance register, a roadmap, a GitHub Project and its own releases. Game creation follows the *New game* workflow (§13.3).

### 8.2 Inside the Godot project

- `addons/gdf_*`: **vendored framework**. Read-only. CI verifies checksums against `framework.lock`.
- `modules/<name>/`: **game-local modules**, with the same anatomy and rules as framework modules plus `maturity = "game-local"`. This is where harvest candidates grow, built reuse-ready per [E] D1. A module that will knowingly *not* be reusable carries a **reuse-waiver ADR** (cost/benefit, what would be needed to generalise later).
- `src/`: composition and game-specific glue (scenes, presentation, UI). Not intended for reuse.
- `content/`: data definitions with stable IDs.
- `scenarios/`: scenario definitions, recorded replays, golden hashes, perf benchmarks.

### 8.3 Simulation / presentation contract

| Concern | Simulation | Presentation |
|---|---|---|
| Representation | Typed `RefCounted` data, packed arrays, entity IDs | Nodes, scenes, resources |
| Math | `gdf_fixed` integers | floats, interpolated between ticks |
| Time | Ticks (fixed rate per game, e.g. 10–30 Hz) | Frames |
| Mutation | Only via commands executed on ticks | Never mutates sim; emits commands |
| Engine services | Forbidden: physics, navigation server, `RandomNumberGenerator`, `Time`, `OS`, scene tree | Free to use |
| Output | Events (per tick) + read-only views | Rendering, audio, UI |

**Perspectives** (3D, isometric, first-person, third-person) are *presentation adapters*. Each is a camera rig plus an input-mapping profile that translates input into commands. One simulation can be shown through several perspectives.

**Network model is a per-game ADR.** Lockstep fits strategy, colony and transport games. Direct-control perspectives in competitive play (e.g. first person) suffer from lockstep input delay. Such a game may choose server-authoritative with client prediction. The command/tick architecture serves both.

### 8.4 Framework dependency declaration

```toml
# games/<slug>/framework.lock  (written by `gdf vendor`, never by hand)
framework = "0.3.1"
godot = "4.7.1"
modules = ["gdf_core", "gdf_fixed", "gdf_rng", "gdf_sim", "gdf_serial", "gdf_content", "gdf_events", "gdf_debug", "gdf_testkit"]
allow_experimental = ["gdf_net"]
source_commit = "<sha of tag framework/v0.3.1>"
[checksums]
"addons/gdf_sim/api/gdf_tick_scheduler.gd" = "sha256:…"
```

Upgrading is a T1 task (T2 across a major version): `gdf vendor games/<slug> --framework 0.4.0`, run the game's full verify, record the result in the PR.

---

## 9. AI agent architecture

Six concepts, kept strictly separate:

| Concept | What it is | Where it lives | Lifetime |
|---|---|---|---|
| **Agent definition (role)** | Purpose, responsibilities, allowed outputs, permission profile, required context, definition of done, escalation rules | `harness/roles/<role>.md` | Versioned; changes are T2 |
| **Agent instance (run)** | One execution of a role on one task step, by one vendor CLI + model, in one worktree, with a budget | `runs/<run-id>/` (report, logs) | Minutes to hours; disposable |
| **Workflow** | Ordered steps → role per step, inputs/outputs, gates, human approvals, tier rules | `harness/workflows/<name>.yaml` + `.md` playbook | Versioned |
| **Orchestration** | The mechanism that executes a workflow: O0 human-driven, O1 `gdf run` | `harness/tools/gdf/` | Code |
| **Tools & permissions** | Capabilities available to an instance: CLI commands, MCP servers, file-scope and network rules | `harness/permissions/<profile>.yaml` → generated vendor configs | Versioned |
| **Knowledge & instructions** | Governance, AGENTS.md chain, specs, ADRs, skills, knowledge base | Repo (see §12) | Versioned docs |

**Run contract.** Every instance, whatever the vendor, receives the same inputs and must produce the same outputs:

```yaml
# input (written by orchestrator to runs/<id>/input.yaml)
run_id: 2026-11-03T1012-fw-42-implement
role: implementer
workflow: feature
step: implement
tier: T1
issue: 42
spec: framework/workbench/addons/gdf_sim/spec.md
context_manifest: [AGENTS.md, framework/AGENTS.md, framework/governance/FW-DET.md, …]
worktree: ../wt/fw-42
branch: agent/42-tick-scheduler
permission_profile: implementer
executor: { adapter: claude, model: <configured> }
budget: { max_usd: 5, max_minutes: 60 }
```

Output: `runs/<id>/report.json` validated against `harness/schemas/run-report.schema.json`. Fields: status (`done | blocked | failed`), summary, changed files, ACs addressed, checks executed with results, evidence paths, open questions, `instructions_digest`, executor, model, tokens/cost, duration.

---

## 10. Agent role definitions

Five roles **[R]**. Modes are parameters of a role, not separate agents. New roles require a harness ADR backed by observed failures.

| Role | Modes | Writes | Must not | Default executor |
|---|---|---|---|---|
| **Planner** | product (backlog, issue refinement), design (game design docs; proposes, never decides), architect (ADRs, module design), framework-architect (API review, harvest assessment) | Specs, ADRs (as `proposed`), design docs, issue updates, task breakdowns | Write source code; approve its own ADRs; change approved governance | Strong reasoning model |
| **Implementer** | feature, fix, refactor, harvest, docs | Source, tests for internal behaviour, module docs, changelog fragments, in its worktree only | Edit acceptance tests authored by the Verifier (can propose changes); touch vendored/generated/governance paths; merge | Primary coding CLI |
| **Verifier** | test-design (from spec, before implementation), verification (run everything, collect evidence), playtest-prep (build + checklist for owner) | Acceptance tests, scenarios, golden hashes (new ones need review), verification reports | Modify implementation code | Different model than implementer where practical |
| **Reviewer** | code, architecture compliance, spec conformance, security | Review comments, verdict (`approve / request-changes / escalate`), tier challenges | Write to branch (read-only) | **Different vendor** than implementer [R] |
| **Researcher** | tech evaluation, tool scouting, Godot investigation | `knowledge/research/*.md` with sources and confidence levels | Change code or governance; present secondary sources as established | Model with web access |

Human-only responsibilities (never delegated): approving T2 changes, governance, ADRs, framework releases, game pillars and creative direction, playtest verdicts, budgets, exceptions, credentials.

Each role file follows template **AD** (`harness/templates/agent-definition.md`).

---

## 11. Orchestration architecture

### 11.1 Stages

| Stage | Trigger to enter | Mechanism |
|---|---|---|
| **O0 Manual** (P0–P1) | Start | Owner drives an interactive CLI session per task. Workflow playbooks guide steps. `gdf review <pr> --executor codex` runs the independent review headless. Parallel work = parallel sessions in separate worktrees (`gdf wt new <issue>`). |
| **O1 Runner** (≥P4) | Workflows stable over ≥10 tasks; observed time lost to manual sequencing | `gdf run <workflow> --issue N`: executes YAML steps as headless CLI calls via adapters, one worktree per writing step, collects reports, enforces gates, **pauses at human gates** (issue comment + label `needs-owner`), resumes on approval. Concurrency limit and budget caps. |
| **O2 Scheduled / fleet** | Evidence that O1 throughput is the constraint | Scheduled runs (nightly canary, doc audits), GitHub-triggered runs, possibly containers for unattended execution. |

No agent framework (LangGraph, CrewAI, AutoGen…) and no long-running services. Vendor CLIs already provide the agent loop, tool use and sandboxing. Our layer is orchestration *of* them, roughly 500–1500 lines of Python at O1.

### 11.2 Adapters

`harness/adapters/<vendor>/` holds two things:
1. **Invocation**: how to run headless with a prompt file, working directory, model, permission flags and output capture (e.g. `claude -p`, `codex exec`, `gemini -p`, `opencode run`).
2. **Generators**: render canonical sources into the vendor's native locations: `CLAUDE.md`, `.claude/skills/`, `.claude/settings.json` permissions, `.agents/skills/`, `.codex/config.toml`, etc. `gdf sync adapters` writes them; CI runs `gdf sync adapters --check`.

Adding a vendor = adding an adapter directory + passing the harness evals (§19.4).

### 11.3 Parallelism rules

- One writing agent per worktree per branch. Read-only roles (reviewer, researcher) may share the main checkout.
- Parallel tasks must touch disjoint modules, or be explicitly sequenced in the issue. `gdf run` checks module overlap from the spec.
- Integration is a merge queue: rebase → `gdf verify` → merge. No agent resolves a semantic conflict in another task's code. Conflicts go back to the Implementer of the later task.
- Godot import cache (`.godot/`) is per worktree. `gdf wt new` warms it with a headless import.

---

## 12. Knowledge and context management

### 12.1 Document classes and authority

| Class | Examples | Authority | Lifecycle |
|---|---|---|---|
| Normative | constitution, rule files, approved ADRs, approved specs | Binding at its level | draft → approved → superseded |
| Descriptive | architecture overviews, module READMEs, API docs | Must match code (doc checks) | Updated in the same PR as code |
| Procedural | workflows, skills, playbooks | Binding for the procedure | Versioned with harness |
| Advisory | `knowledge/**`: research, patterns, Godot notes, lessons | Informative; cite sources and confidence | Reviewed quarterly; stale → archived |
| Operational | issues, PRs, run reports | Record, not authority | GitHub / `runs/` |

### 12.2 Context supply (selective, never "the whole repo")

1. **Always on** (via CLI auto-loading): the `AGENTS.md` chain for the working directory. Kept small.
2. **Per run**: the *context manifest* from `gdf context`, consisting of the spec, referenced ADRs, the rule files whose IDs apply, READMEs and `api/` of touched or depended-on modules, and the role definition.
3. **On demand**: skills (loaded by description match), `knowledge/INDEX.md` (generated one-line summaries, so agents can search before reading), and code search.
4. **Never**: other games' code or docs (a game agent has no reason to read another game). Raw run logs.

### 12.3 Discovery and verification of applicable instructions

- `gdf context` (§6.1) is authoritative. Agents cite rule IDs in reports and PRs ("complies with FW-DET-002…"). The reviewer checks the citations.
- The `knowledge/INDEX.md` and `framework/architecture/module-catalogue.md` are generated from frontmatter and `module.toml`, so they can't drift from the content.

### 12.4 Lessons-learned loop

Lessons are not an ever-growing file. A lesson enters `knowledge/lessons/` with a trigger (incident, review finding). At each milestone retrospective the Planner proposes: **promote** (into a rule, skill or template; the lesson then links there), **keep**, or **archive**. Lessons that change behaviour must end up in a normative or procedural document, because agents don't reliably act on advisory text.

### 12.5 Glossary

`governance/glossary.md` (org) and `games/<slug>/design/glossary.md` (game terms). Terms are used consistently across spec, code and UI. The reviewer flags synonyms.

---

## 13. Development workflows

### 13.1 Default lifecycle (improved)

```
Request ─▶ Triage (tier T0/T1/T2, product, workflow) ─▶ Analysis & Spec (Planner; ACs)
   ─▶ [T2: Architecture review + ADR ─▶ OWNER APPROVAL 1]
   ─▶ Test design (Verifier writes failing acceptance tests/scenarios from ACs)
   ─▶ Implementation (Implementer, worktree)
   ─▶ Automated verification (gdf verify: static, unit, scenario, determinism, perf, docs)
   ─▶ Independent review (Reviewer, other vendor; may raise tier)
   ─▶ [Game features: PLAYTEST GATE (owner) when feel/fun/UX is in the ACs]
   ─▶ [T2: OWNER APPROVAL 2]
   ─▶ Integration (merge queue) ─▶ Docs/changelog fragment/report ─▶ Close issue (+ lesson if any)
```

Changes from the proposed 10-step lifecycle: triage with tiers; verification-first test design; playtest gate; approval before *and* after for T2 only; the merge queue replaces an "integration agent"; reporting is a by-product of the run reports.

| Tier | Examples | Spec | Test design step | Review | Owner |
|---|---|---|---|---|---|
| **T0** | Typo, comment, docs fix, test-only, dependency patch bump | Issue text | No | Automated + light review | No (auto-merge allowed [A]) |
| **T1** | Feature or bug inside one module/game, internal refactor | Feature/bug spec with ACs | Yes | Full independent review | No, unless playtest gate applies [A] |
| **T2** | Public framework API, cross-module, architecture, governance, harness roles/workflows, releases, network protocol, save format | Spec + ADR | Yes | Full + architecture | Yes, before and after |

### 13.2 Workflow catalogue

Each workflow is a `harness/workflows/<id>.yaml` (template WF) with a playbook.

| ID | Workflow | Key steps / notes |
|---|---|---|
| `feature` | New game feature | Lifecycle above; prototype features use `prototype` instead |
| `prototype` | Throwaway exploration in a game | Time-boxed; lives in `games/<slug>/project/prototypes/`; no coverage requirement; **cannot be merged into `src/`/`modules/`**. Promotion = re-implement via `feature`. Output: playtest notes + decision |
| `fw-capability` | New framework capability (non-harvest) | Only for foundations or tooling; requires a demonstrated consumer need in the issue; T2 |
| `harvest` | Game-local module → framework | Planner assessment (consumers, game-specific leakage, API shape) → framework spec → move + generalise → tests in workbench → demo → original game re-vendors and replaces its local copy → T2 approval |
| `new-game` | New game product | Scaffold (`gdf new game <slug>`), game constitution workshop with owner, L3 rules, framework vendoring, network-model ADR, first prototype issues. No production code before the constitution is approved |
| `bugfix` | Bug | Reproduce first (failing test or replay) → fix → verify → review. Desyncs: always attach replay + hashes |
| `refactor` | Behaviour-preserving change | Golden hashes and tests must be unchanged; any golden-hash change makes it a feature |
| `architecture-change` | ADR-driven change | Always T2; ADR first; migration plan; framework major-version check |
| `research` | Tech evaluation | Question → Researcher → report with sources, confidence, recommendation → Planner → optional ADR |
| `review` | Independent review | Checklist: spec conformance per AC, rule citations, tests meaningful (not tautological), determinism bans, API/doc updates, tier correctness |
| `release-framework` | Framework release | Freeze → full verify + determinism + canary across games → changelog → release notes → owner approval → tag `framework/vX.Y.Z` → artefact |
| `release-game` | Game release | Version bump → export all platform presets → smoke scenarios on exports → playtest sign-off → tag `games/<slug>/vX.Y.Z` |
| `docs` | Documentation update | T0 unless normative; normative docs follow the doc's own approval level |

### 13.3 Testing workflow

Covered in §17. Test design is a step inside other workflows, not a separate pipeline.

---

## 14. Specification and documentation standards

### 14.1 Frontmatter (all governed docs)

```yaml
id: FS-foo-012            # unique; prefix by doc type and scope
title: Building placement
type: feature-spec        # constitution|rule|adr|feature-spec|component-spec|test-plan|workflow|role|research|lesson
status: draft             # draft|proposed|approved|superseded|archived
version: 0.2.0
owner: product-owner      # or a role for drafts
scope: games/foo
tier: T1
issue: 57
supersedes: ~
related: [ADR-foo-0003, FW-DET-002]
```

### 14.2 ID scheme

| Type | Pattern |
|---|---|
| Rules | `ORG-AREA-NNN`, `FW-AREA-NNN`, `GAME-<slug>-AREA-NNN`, `HAR-AREA-NNN` |
| ADRs | `ADR-ORG-NNNN`, `ADR-FW-NNNN`, `ADR-<slug>-NNNN`, `ADR-HAR-NNNN` |
| Specs | `FS-<scope>-NNN` (feature), `CS-<module>` (component) |
| Acceptance criteria | `<spec-id>/AC-n` (referenced from tests by tag) |
| Exceptions | `EXC-NNN` |

### 14.3 Acceptance criteria rules

- Each AC is **observable** and **verifiable by a named method**: `unit`, `scenario`, `determinism`, `perf`, `screenshot` (advisory), `playtest` (human), or `review`.
- Format: Given / When / Then, with concrete values. "Works well", "fast" and "intuitive" are not allowed.
- Tests reference ACs (`# covers FS-foo-012/AC-3`). `gdf check trace` reports ACs without tests and tests without ACs.

### 14.4 Documentation completeness (machine checks)

- Every module: README, spec, `module.toml` valid, every `api/` class has a doc comment header.
- Every public API change in a PR comes with a changelog fragment (`framework/releases/unreleased/*.md`).
- Links resolve; frontmatter valid against schema; `AGENTS.md` ≤150 lines; no `TODO` in approved normative docs.

---

## 15. Tooling and technology recommendations

| Area | Choice | Status | Alternatives / notes |
|---|---|---|---|
| Engine | Godot 4.7.x, pinned per project via `.godot-version` | [E]/[R] | Engine upgrades are T2, framework-first, then games |
| Language | GDScript (static typing enforced); native escape via GDExtension (framework) or C# (game ADR) | [E]/[R] | See stage 1 §8.0.1 |
| Test framework | **gdUnit4** (v6.2.x for Godot 4.7; v7 when stable) | [R] | GUT 9.x: simpler, also mature; gdUnit4 chosen for scene runner, input simulation, mocks, JUnit/HTML, C# support (future-proof for game-level C#) |
| Lint/format | gdtoolkit 4.x (`gdlint`, `gdformat`) + custom `gdf` lints (imports, prefixes, determinism bans, layout) | [R] | Godot's own warnings as errors are the first line |
| Repo tooling | Python ≥3.12 managed by **uv**; `gdf` CLI (Typer or argparse) | [R] | Must be installed (not present on this machine) |
| VCS | Git now; **Git LFS deferred** until the first binary asset (`*.png *.jpg *.psd *.blend *.glb *.fbx *.wav *.ogg *.mp3 *.ttf *.otf *.exr *.hdr`) | [E] | Repo is text-only today. LFS rules must be added to `.gitattributes` *before* the first binary commit, otherwise `git lfs migrate` (history rewrite) is needed. Free GitHub quota: 10 GiB storage + 10 GiB bandwidth/month; CI and agent checkouts count against bandwidth, so CI fetches LFS only in jobs that need assets |
| Hosting/CI/work items | GitHub private repo, Actions, Issues + Projects; `gh` CLI | [E] | `gh` must be installed |
| Primary agent CLI | Claude Code | [R] | |
| Independent reviewer | Codex CLI (different vendor) | [R] | Gemini CLI; opencode with a different provider |
| Godot MCP | `godot-editor-mcp` (inspection toolset only by default) for interactive debugging | [R] experimental | Nothing in CI depends on it |
| Observability | Run reports (own schema) + JSONL aggregation; OTel export optional later | [R] | OTel GenAI conventions still "Development" |
| Worktree GUI managers | Not adopted | [R] | Re-evaluate at O2 |

---

## 16. Godot integration strategy

- **Engine pinning.** `.godot-version` contains the exact version (e.g. `4.7.1-stable`). `gdf godot` resolves or downloads the matching binary (standard or .NET) into `tools/godot/` and is the only way scripts invoke Godot. CI uses the same resolver.
- **Project settings baseline.** Framework rules define a baseline `project.godot` fragment. It sets GDScript warnings-as-errors for typing and unsafe access, physics tick rate, the rendering method per platform target, and `application/config/features`. `gdf check project` verifies each project against it.
- **Headless verbs** (wrapped by `gdf`):
  - `--headless --import`: clean import; fail on errors.
  - `--headless --check-only -s <script>` / project-wide parse: syntax and type errors.
  - `--headless -s addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://…`: tests with JUnit output. The exact path is pinned per gdUnit4 version in `gdf`.
  - `--headless --main-pack`/scenario entry: `gdf_testkit` runs a scenario, then writes a JSON report and exits with status.
  - Rendered capture: `--write-movie out.png --fixed-fps 30 --quit-after N` for deterministic frame dumps.
- **Runtime error detection.** Every headless or rendered run captures stdout/stderr. `gdf` fails the run on any `ERROR:`/`SCRIPT ERROR:` line, or on any warning not in the per-project allowlist. Most agent-introduced bugs show up first as runtime errors.
- **Scenes and resources.** Prefer text formats (`.tscn`, `.tres`). Binary resources are only for imported assets. Validate UID references and missing dependencies after import. Scenes that sim modules depend on are forbidden (sim is scene-free).
- **Vendoring and UIDs.** `gdf vendor` copies `.uid` files alongside scripts. The same module across different game projects keeps the same UIDs, which is harmless because projects are separate. Within one project, a module exists once only.
- **Exports.** `export_presets.cfg` committed, with no secrets in it. CI builds Windows and Linux on every main merge. Web and Android builds are nightly once relevant. Smoke scenarios run on the exported binary, not just the editor build.
- **Linux CI rendering.** Headless for most checks. Rendered checks use Xvfb + Mesa lavapipe (Vulkan) or the Compatibility renderer. Rendered checks are tolerant (perceptual diff thresholds), and screenshots are evidence for review rather than strict gates.
- **Editor-level automation.** The Godot MCP server is used for interactive inspection by the owner and agents during O0 debugging only.

---

## 17. Testing and quality assurance

### 17.1 Test layers

| Layer | What | Tool | Speed | Gate |
|---|---|---|---|---|
| Static | Parse, types, lint, format, deps, layout, determinism bans, docs, governance | Godot `--check-only`, gdtoolkit, `gdf check *` | seconds | Every PR |
| Unit | Pure sim/foundation logic | gdUnit4 headless | seconds | Every PR |
| Scenario | Sim behaviour over many ticks from scripted commands; asserts on state | `gdf_testkit` headless | seconds–minutes | Every PR touching sim |
| Determinism | Same scenario → identical state-hash sequence: (a) twice on one machine, (b) Windows vs Linux in CI, (c) replay playback = recorded hashes | `gdf_testkit` + CI matrix | minutes | Every PR touching sim; required for release |
| Scene/integration | Scenes load, nodes wired, input → commands, presentation reacts to events | gdUnit4 scene runner | seconds–minutes | PRs touching presentation |
| Rendered | Screenshots/frames at checkpoints; perceptual diff vs baseline; multimodal review advisory | `--write-movie` + diff tool | minutes | Advisory; required evidence for UI ACs |
| Performance | Benchmarks: tick time p50/p99 at N entities, frame time, memory; budgets per game | `gdf_testkit` perf probes | minutes | Regression > threshold fails (nightly + release) |
| Export smoke | Exported build launches, runs smoke scenario, exits clean | CI | minutes | Main merges + release |
| Asset integrity | Import clean, size budgets, naming, **provenance metadata present** | `gdf check assets` | seconds | PRs touching assets |
| Compatibility | Each game against framework HEAD (canary); each game against its pinned version (regular) | CI matrix | minutes | Canary informational; pinned required |
| Playtest | Feel, fun, readability, UX | Owner, with Verifier-prepared build + checklist | human | Game ACs marked `playtest` |

### 17.2 Coverage and quality thresholds (initial, FW-TST / ORG-QUA)

- Framework `api/` + `internal/`: ≥80% line coverage. Game-local modules: ≥70%. `src/`: no threshold, scenario coverage per AC instead. Coverage tooling for GDScript is immature. If unavailable, use the AC-trace report plus a reviewer check as the substitute. **[R]** re-evaluated in P2.
- Zero runtime errors in any test or scenario run.
- Flaky tests are quarantined within 24 h with an issue. Quarantine > 14 days blocks the module's next release.

### 17.3 Against false green

- Verifier writes acceptance tests from the spec *before or independent of* implementation.
- Reviewer checks tests for tautologies (asserting what the code does instead of what the AC says), mocked-away logic and skipped assertions.
- Mutation-style spot checks (O1+): the Verifier deliberately breaks the implementation and confirms the tests fail.
- Golden hashes change only with an explicit, reviewed justification in the PR.

### 17.4 `gdf verify`

`gdf verify [--scope framework|games/<slug>|changed] [--level fast|full|release]` runs the applicable layers and writes `runs/verify/<timestamp>.json`. CI calls exactly this command. The PR is green only if `gdf verify --scope changed --level full` passes.

---

## 18. Security and permission model

### 18.1 Permission profiles (vendor-neutral, generated into each CLI's config)

| Profile | File write | Shell | Network | Git | Secrets |
|---|---|---|---|---|---|
| `planner` | Docs/spec/ADR paths of its scope | `gdf`, `gh issue/pr view`, read-only git | Docs sites allowlist | Commit to own branch | None |
| `implementer` | Own worktree, excluding protected paths | `gdf`, Godot via `gdf godot`, `uv run` | Package registries via lockfiles only | Commit + push own branch | None |
| `verifier` | Tests, scenarios, reports | `gdf`, Godot | None | Commit + push own branch | None |
| `reviewer` | None (comments via `gh pr review`) | Read-only + `gdf verify` | None | None | None |
| `researcher` | `knowledge/research/` | Read-only | Web | Commit to own branch | None |

### 18.2 Protected paths (CODEOWNERS = owner; plus pre-commit/hook checks)

`governance/**`, `*/governance/**`, `harness/roles/**`, `harness/permissions/**`, `harness/workflows/**`, `.github/**`, vendored `addons/gdf_*` in games, `framework.lock`, `*.generated`, `export_presets.cfg`.

### 18.3 Controls

- **Branch protection on `main`**: PR required, required checks (`verify`, `determinism` where applicable), CODEOWNERS review for protected paths, no force-push, linear history.
- **Agent GitHub identity**: a fine-grained token (or a GitHub App) with contents and PR write on this repo only. No admin, no settings, no secrets access. The owner's personal token is never available to agents.
- **Secrets** (Steam, signing keys, store credentials) live only in GitHub Actions environments with required owner approval. They are never on agent-accessible disk and never in the repo (`gitleaks` in CI).
- **Third-party code**: Godot addons and Python packages are pinned by version and hash and recorded in `THIRD_PARTY.md`. A new dependency is T2 at framework level, and T1 in a game with a license check.
- **MCP servers**: allowlist only, pinned versions, minimal toolsets enabled. No MCP server gets write access outside the worktree.
- **Prompt injection**: content from the web, issues by others, assets and downloaded files is data, not instructions (ORG-SEC rule). The Researcher has no write access to code or governance.
- **Execution sandbox**: use each CLI's sandbox/permission mode. Unattended O1/O2 runs move to containers before they get network access.

---

## 19. Observability and reporting

### 19.1 Run reports

Every agent instance writes `runs/<run-id>/report.json` (schema-validated, §9). Reports are attached to the PR as a summary comment by `gdf` and aggregated into `runs/index.jsonl`.

### 19.2 Metrics (computed by `gdf report`)

| Metric | Why |
|---|---|
| First-pass review acceptance rate (per role, executor, model) | Quality of implementers and the effect of prompt and role changes |
| Escaped defects (bugs found after merge, linked to the PR) | Real quality signal |
| Rework rate (review cycles per PR) | Spec clarity |
| Cost and time per merged PR, per tier | Budget control |
| Blocked runs by reason (instruction conflict, missing spec, tool failure) | Harness friction, which drives the O1 decision |
| Determinism failures, flaky tests | Platform health |
| Framework reuse: modules × consuming games; harvests completed | Objective #2 |

### 19.3 Reports to the owner

- **Per PR**: summary comment with ACs, evidence links, cost and reviewer verdict.
- **Weekly digest** (`gdf report weekly`, posted as a GitHub issue labelled `report`): merged work, open decisions waiting on the owner, budget, metrics trends, top friction.
- **Milestone retrospective**: lessons → promote/keep/archive (§12.4).

### 19.4 Harness evaluations

`harness/evals/` holds a set of fixed tasks on pinned repo snapshots, each with expected outcomes and graders: tests pass, rule compliance, reviewer catches a seeded bug. They run when changing role definitions, workflows, adapters or models (a harness change is T2). Results are recorded with the harness ADR. This is how we know a model swap or prompt change helped rather than hurt.

---

## 20. Versioning and release strategy

| Artefact | Scheme | Tag | Notes |
|---|---|---|---|
| Framework | SemVer, single release train. `0.x` = API unstable (minor may break, documented) | `framework/vX.Y.Z` | Maturity per module; deprecations announced one minor ahead |
| Game | SemVer-ish for player builds | `games/<slug>/vX.Y.Z` | Declares framework + Godot version in release notes |
| Harness | Version in `harness/VERSION`; changes via ADR | `harness/vX.Y.Z` | Run reports record harness version |
| Save format | Integer schema version per game | n/a | Migration chain in `gdf_serial`; old-save fixtures tested in CI |
| **Sim fingerprint** | Hash of sim module code + content definitions + framework version | n/a | **Multiplayer clients refuse to connect on mismatch.** Shown in the debug overlay |
| Network protocol | Integer version | n/a | Bumping it is T2 |
| Content IDs | Stable forever once released; renames via alias table | n/a | Required for saves and mods |

**Framework release process:** `release-framework` workflow (§13.2). The changelog is assembled from fragments. Release notes follow template RN. `compat-matrix.md` lists which game versions use which framework version. Hotfix releases (`X.Y.Z+1`) are branched from the tag when a game needs a fix without upgrading.

**Engine upgrades:** Framework first (workbench passes everything on the new Godot), then a framework release declaring the new range, then each game upgrades in its own T2 task. Games are never forced.

---

## 21. Risks and trade-offs

The full risk register (R1–R14) is in [stage 1 §7](stage-1-analysis.md#7-risk-register-top-12). The main trade-offs accepted in this blueprint:

| Trade-off | We chose | We pay | We gain |
|---|---|---|---|
| Fixed-point deterministic sim vs float sim | Fixed-point | Verbose math, slower GDScript, more framework work up front | Lockstep multiplayer, replays, reproducible tests, desync detection |
| Vendoring vs live link | Vendoring + lock | Duplicate files, explicit upgrade tasks | Games never break by accident; clear version per game |
| Single release train vs per-module versions | Single train | Coarser versioning | Much less release overhead for one person |
| Own thin orchestration vs agent framework | Thin scripts | Fewer built-in features (memory, graphs, UI) | No second agent runtime; vendor CLIs improve for free |
| GitHub Issues vs repo-file tasks | Issues [E] | Work items outside git; GitHub dependency | Better UI, Projects, mobile triage, PR linking. Mitigation: specs/ADRs in repo; periodic issue export |
| Different-vendor reviewer | Yes | Two subscriptions, adapter upkeep | Real independence; less correlated blind spots |
| Five roles vs specialist roster | Five | Role files with modes are longer | Less orchestration, clearer accountability |
| GDScript-first | Yes [E] | Possible perf ceiling | Fastest iteration, web export, single language |

---

## 22. Phased implementation roadmap

Each phase ends with explicit **exit criteria**, verified by the owner. No phase starts game design except P3.

### P0: Foundations
- `git init`, `.gitattributes`/`.gitignore` extended, GitHub private repo, branch protection. Git LFS is deferred (§15): it is set up when the first binary asset is about to be committed.
- Install and pin tooling: uv/Python, `gh`, gdtoolkit. Engine resolver `gdf godot`.
- Migrate the current root Godot project to `framework/workbench/`; create the layout of §4.
- Governance v1: constitution, L1 rule files (minimum set), decision-making rules, glossary.
- Root, framework and harness `AGENTS.md`; `CLAUDE.md` import.
- `gdf verify` (fast level: import, parse, lint, format, gdUnit4 smoke test) locally and in Actions on Linux + Windows.
- **Exit:** CI green on an empty workbench; one T0 change made by an agent end-to-end via PR with a reviewer from a second vendor.

### P1: Harness MVP
- 5 role definitions, 8 templates (+ GitHub issue forms), workflows `feature`, `bugfix`, `research`, `review`, `docs` as YAML + playbooks.
- Permission profiles → generated adapter configs (Claude Code, Codex CLI); `gdf sync adapters --check` in CI.
- `gdf context`, `gdf wt new`, `gdf review`, run report schema and PR summary comment.
- `gdf check governance|docs|layout`.
- **Exit:** three T1 framework tooling tasks completed through the full lifecycle with run reports; all checks enforced in CI.

### P2: Framework v0.1 (foundations)
- Modules `gdf_core`, `gdf_fixed`, `gdf_rng`, `gdf_sim`, `gdf_serial`, `gdf_content`, `gdf_events`, `gdf_debug`, `gdf_testkit` (§7.3).
- Determinism CI (Windows vs Linux), replay round-trip, golden hashes.
- **Performance spike:** a synthetic workbench sim (not a game) with 10k fixed-point agents. Measure tick time. ADR on the native kernel strategy (GDExtension vs none; C# remains a per-game option).
- **Lockstep spike:** two local clients run the synthetic sim over ENet with injected latency. Desync detection works.
- `gdf vendor`, `framework.lock`, vendored-checksum CI check. Tag `framework/v0.1.0`.
- **Exit:** v0.1.0 released; spike results recorded as ADRs; the synthetic sim is deterministic across OSes.

### P3: Game 001 concept and prototype
- **First game selection happens here** (owner). `new-game` workflow: constitution, pillars, network-model ADR.
- `prototype` workflow loops with playtest gates. `gdf_net` matures from the spike if the game needs it.
- Set up Git LFS (§15) before the first art/audio asset is committed, if that has not already happened earlier.
- **Exit:** owner go/no-go on a playable prototype. Prototype learnings recorded. Production backlog created.

### P4: Game 001 production + harvest + O1
- Production through `feature` workflows; game-local modules built reuse-ready.
- First harvest candidates assessed (not necessarily executed).
- O1 runner if P1–P3 metrics show sequencing friction. Harness evals established.
- **Exit:** vertical slice shipped internally; metrics dashboard (weekly digest) in use.

### P5: Ship and second consumer
- Game 001 release workflow; first harvests executed when a second consumer (game 002 or a deliberate framework demo with a real use case) exists; first modules reach Approved.
- Framework 1.0 when ≥1 game has shipped on it and core foundations reach Stable criteria.

---

## Appendix A: Initial repository documents

Created in P0–P1. "Authority" = what the document can bind.

| Path | Purpose | Owner | Authority | Relationships |
|---|---|---|---|---|
| `AGENTS.md` | L1 entry: non-negotiables, commands, routing | Owner | Routing (cites rules) | Imports nothing; points to governance/ |
| `CLAUDE.md` | Claude Code bridge (`@AGENTS.md`) | Generated | None of its own | Generated from harness/adapters/claude |
| `README.md` | Human orientation | Owner | None | Links to blueprint, governance |
| `governance/constitution.md` | Mission, objectives, authority model, principles P1–P12, governance levels | Owner | Highest | Parent of all rule files |
| `governance/rules/ORG-ENG.md` | Engineering standards | Owner | L1 | Refined by FW-*, GAME-* |
| `governance/rules/ORG-QUA.md` | Quality, definition of done, gates per tier | Owner | L1 | §13, §17 |
| `governance/rules/ORG-AGT.md` | Agent behaviour, escalation, worktrees, reporting | Owner | L1 | harness/roles |
| `governance/rules/ORG-SEC.md` | Security, permissions, secrets, prompt injection | Owner | L1 | harness/permissions |
| `governance/rules/ORG-DOC.md` | Doc standards, frontmatter, IDs | Owner | L1 | §14, schemas |
| `governance/rules/ORG-DEC.md` | Decision authority matrix, ADR process, exceptions | Owner | L1 | decisions/, exceptions/ |
| `governance/rules/ORG-GIT.md` | Branching, commits, PRs, merge queue | Owner | L1 | .github/ |
| `governance/rules/ORG-AST.md` | Asset provenance, licensing, AI-content disclosure | Owner | L1 | games/*/art-direction |
| `governance/glossary.md` | Canonical terms | Owner | L1 (terminology) | Game glossaries extend |
| `governance/decisions/ADR-ORG-0001..` | Initial decisions (monorepo, GitHub, harvest, determinism, vendoring, vendor strategy) | Owner | L1 | Cites stage 1 analysis |
| `framework/AGENTS.md` | L2 entry | Owner | Routing | → framework/governance |
| `framework/product/vision.md` | Framework purpose, customers, success metrics, non-goals ("not an engine") | Owner | L2 | roadmap |
| `framework/product/roadmap.md` | Now/next/later, linked to GitHub Project | Planner→Owner | Plan | Issues |
| `framework/governance/FW-ARCH.md` | Layers, module anatomy, maturity | Owner | L2 | §7 |
| `framework/governance/FW-API.md` | API conventions | Owner | L2 | §7.5 |
| `framework/governance/FW-DEP.md` | Dependency rules | Owner | L2 | `gdf check deps` |
| `framework/governance/FW-DET.md` | Determinism rules and bans | Owner | L2 | `gdf check det`, CI |
| `framework/governance/FW-CMP.md` | Compatibility, deprecation, SemVer | Owner | L2 | releases |
| `framework/governance/FW-TST.md` | Testing requirements | Owner | L2 | §17 |
| `framework/architecture/overview.md` | Descriptive architecture | Planner | Descriptive | Generated catalogue |
| `framework/architecture/module-catalogue.md` | Generated from `module.toml` | Generated | Descriptive | — |
| `framework/releases/CHANGELOG.md` | Release history | Generated from fragments | Record | RN template |
| `harness/AGENTS.md` | Harness entry | Owner | Routing | — |
| `harness/roles/*.md` (5) | Role definitions | Owner | Procedural (HAR) | permissions/, workflows/ |
| `harness/workflows/*.yaml/.md` | Workflow definitions + playbooks | Owner | Procedural | roles, templates |
| `harness/permissions/*.yaml` | Permission profiles | Owner | Procedural (security) | ORG-SEC |
| `harness/templates/*` (8) | Templates (Appendix C) | Owner | Form | Issue forms generated |
| `harness/schemas/*.json` | Machine contracts | Owner | Technical | gdf |
| `knowledge/INDEX.md` | Generated index | Generated | Advisory | — |
| `games/<slug>/…` | Created by `new-game` workflow (P3) | Owner | L3 | — |

## Appendix B: Decision register

| Decision | Status |
|---|---|
| Godot 4, GDScript initial, PC primary / mobile + web secondary, 3D/iso/FPS/TPS support | [E] |
| Monorepo, four areas, framework as independent product, games consume releases | [E] |
| Three governance levels; lower levels cannot silently override | [E] |
| Provider independence; specialised roles not permanently running | [E] |
| Harvest model, reuse-ready by default, reuse waivers by decision | [E] (D1) |
| GDScript first; C#/native later where performance demands | [E] (D2) |
| GitHub + Issues | [E] (D3) |
| Multiplayer (co-op and competitive) required → deterministic lockstep-capable sim core | [E] (D5) + derived |
| Plain folder names; per-scope ADRs/backlogs; `harness/` naming | [R] |
| Five roles; risk tiers; verification-first; prototype lane; playtest gate | [R] |
| Vendored framework + lockfile; single release train; maturity levels | [R] |
| gdUnit4, gdtoolkit, Python/uv `gdf` CLI | [R] |
| Git LFS deferred until the first binary asset; rules added to `.gitattributes` before that commit | [E] (2026-11-25) |
| Claude Code primary + Codex CLI reviewer, via adapters | [R] (D4) |
| Staged orchestration O0→O1→O2, no agent framework | [R] |
| **Approval of this blueprint (status → approved)** | [A] |
| Repository/studio name; final `gdf` prefix | [A] |
| Auto-merge for T0 (and T1 without playtest ACs) after green + reviewer approval | [A] |
| Monthly AI budget ceiling and per-run caps | [A] |
| Art/audio sourcing strategy and AI-content disclosure policy | [A] (not blocking until P3) |
| Framework license (default private/proprietary) | [A] (default applies) |

## Appendix C: Templates

Located in `blueprint/templates/` until P1, then moved to `harness/templates/`.

| Code | File | Used by |
|---|---|---|
| FS | `feature-spec.md` | Planner (feature, bugfix for large bugs) |
| CS | `component-spec.md` | Planner/framework-architect (modules, harvest) |
| ADR | `adr.md` | Planner (architect), owner approves |
| AD | `agent-definition.md` | Owner (harness) |
| WF | `workflow-definition.yaml` | Owner (harness) |
| TK | `task.md` (also GitHub issue form source) | Planner/owner |
| TP | `test-plan.md` | Verifier |
| RN | `release-notes.md` | release workflows |
