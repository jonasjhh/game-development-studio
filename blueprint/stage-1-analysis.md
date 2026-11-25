# Blueprint Stage 1: Analysis, Risks, Target Architecture

Status: DRAFT for owner review · Date: 2026-11-01 · Input: `BOOTSTRAP.md`

This is stage 1 of the process in BOOTSTRAP §16. It analyses the requirements, challenges assumptions, summarises the research, names the risks, and proposes a target architecture. The full *Blueprint v1.0* (22 sections + templates + repository documents) is stage 2. It will be written once the decisions in §8 are settled.

Every position is tagged:

- **[ESTABLISHED]**: decided by you in BOOTSTRAP.md.
- **[RECOMMEND]**: my recommendation. I'll proceed with it unless you object.
- **[DECIDE]**: affects the architecture materially; I need your answer.

---

## 1. Current state of the workspace

| Observation | Consequence |
|---|---|
| The root folder is itself a Godot project (`project.godot`, Godot 4.7, Forward+, Jolt, D3D12). | Conflicts with a monorepo that holds a framework and several games. **Each game must be its own Godot project.** The root project will move into the framework's *workbench* project (§5.3). |
| Godot 4.7.1-stable is at `C:\dev\Godot\` (stable since 2026-06-18; 4.7.2 exists). | The engine version must be pinned and resolved by tooling, not hardcoded per machine. |
| Not a git repository yet. | Phase 0 sets up git + LFS before anything else. |
| No Python or Node installed (only the Windows Store `python` stub). | Repo tooling, gdtoolkit (GDScript linting) and most Godot MCP servers need Python. I recommend installing `uv` (§6). |
| Windows host; CI will most likely run on Linux. | Case-sensitivity, path and rendering differences must be designed for from the start (`.gitattributes` already normalises EOL). |

---

## 2. Challenging the assumptions

These are the places where I think the brief as written would lead to a worse outcome.

### 2.1 "Build a framework simultaneously with games" is the biggest risk in the whole brief

Solo and indie developers who build the engine/framework before the game rarely finish games. With AI agents this gets *worse*, not better: an agent will produce a plausible "generic inventory system" in an afternoon, and nothing in that process tells you whether it is the right abstraction. Generic code built with no real consumer is speculative generality, and it looks like productivity.

**[RECOMMEND] Harvest, don't predict.**
- The framework starts with **foundations only**: things every game provably needs and that carry no design opinion. Examples: bootstrapping, logging, the simulation clock and tick scheduler, save/load plumbing, data/content loading, debug overlay and console, the scenario/test harness.
- Gameplay systems (economy, logistics, needs, combat…) are **born inside a game** under `games/<slug>/` with ordinary game rigor. A system is *harvested* into the framework only when a second consumer (a second game, or a deliberately built framework demo with a real use case) needs it. At that point the shape of the abstraction is known.
- Objective #2 (a reusable library) is then met as a *by-product* of shipping, with a measured reuse record instead of a wish list.

This changes the framework roadmap, so it is listed as a decision (D1).

### 2.2 The proposed folder structure

| Proposal | Issue | Recommendation |
|---|---|---|
| Numeric prefixes `00-governance/`, `01-framework/`… | Leak into every path, every import, every agent instruction ("see 03-games/game-001/source/…"). They imply an ordering that doesn't exist. | **[RECOMMEND]** Plain names: `governance/`, `framework/`, `harness/`, `games/`, `knowledge/`. |
| `05-operations/tasks`, `decisions`, `releases` centralised | Breaks product independence: framework tasks, game tasks and org decisions get mixed. Each product needs its own backlog, ADRs and releases. | **[RECOMMEND]** Each scope owns its `decisions/`, `backlog/`, `releases/`. Org-wide decisions stay in `governance/decisions/`. Cross-cutting run reports go to a gitignored `runs/`. |
| `03-games/game-001/source/` + `assets/` + `tests/` as siblings | A Godot project can only reference files under its own `res://` root. Source, assets and tests must all live *inside* one Godot project folder. | Each game has a `project/` folder that is the Godot project. Design docs, specs and ADRs stay outside it, so docs don't get imported as resources. |
| `01-framework/components/`, `tests/`, `documentation/` split by artefact type | Splitting by type spreads one component over 3–4 trees. Agents then miss the tests or docs when changing code. | **Co-locate per module**: code, tests, spec, README and demo scene together. |
| `02-production-system/agent-definitions` | Fine, but tool-specific copies are needed (`.claude/`, `.agents/`, …) because each CLI reads its own location. | Canonical sources in `harness/`. Tool-specific adapter files are **generated** and never hand-edited (§4.3). |

### 2.3 Eleven agent roles is too many to start with

"A large number of agents" ≠ a good architecture (BOOTSTRAP §17). Most of the listed roles are *modes of the same capability* with different instructions and permissions. Several are not agent work at all:

- **Build & release engineer** → a deterministic script (`gdf release`). An LLM should not be in the release path except to draft release notes.
- **Integration agent** → a merge-queue script: rebase, verify, merge if green.
- **Technical writer** → part of every implementer's definition of done, enforced by doc checks. A writer pass is used only for user-facing docs.

**[RECOMMEND]** Five role definitions at first. Add more only when a measured failure mode justifies it.

| Role | Covers | Key property |
|---|---|---|
| **Planner** | PO assistant, game designer, technical/framework architect (as *modes*) | Writes specs, ADRs, task breakdowns. No source write access. |
| **Implementer** | developer | Writes code and docs in an isolated worktree only. |
| **Verifier** | QA engineer | Writes acceptance tests *from the spec, before or independently of the implementation*. Runs the game and produces evidence. |
| **Reviewer** | code reviewer, architecture compliance | **Different model/vendor from the implementer** where possible. Read-only. |
| **Researcher** | research agent | Web + docs access. Writes to `knowledge/research/` only. |

### 2.4 The 10-step lifecycle: right gates, wrong uniformity

One fixed 10-step pipeline for every change has two failure modes. Either it gets ignored for small changes, or it turns *you* into the bottleneck through approval fatigue. Two corrections:

1. **Risk tiers [RECOMMEND].** Gates scale with risk:
   - **T0 trivial** (typo, docs, a test-only change): implement → verify → merge.
   - **T1 standard** (feature or bug inside one module or game): spec with acceptance criteria → tests → implement → verify → independent review → merge.
   - **T2 structural** (public API, cross-module, architecture, governance, framework release): adds an ADR, architecture review, and **your** approval before *and* after.
   The tier is declared in the task and challenged by the reviewer, so an implementer cannot downgrade its own tier.
2. **Two lifecycle steps are missing:**
   - **Verification-first.** Acceptance tests come from the spec, written by the Verifier, *before* implementation. This is what makes "implementer is not sole authority" (§9) enforceable rather than aspirational.
   - **Prototype lane and playtest gate.** Games are found, not specified. A *prototype* lane with low rigor and explicitly disposable code is needed to test ideas. It needs a *human playtest* gate, because no automated check can verify that something is fun or feels good. Without this lane, framework-grade rigor gets applied to throwaway experiments.

### 2.5 Simulation / presentation separation is *the* central technical decision

Your reference games (Factorio, RimWorld, Timberborn, OpenTTD, Against the Storm) are simulation-first. And you want 3D/iso/FPS/TPS presentations that stay decoupled from gameplay. Both point to the same architecture:

- **Simulation core** in plain typed GDScript objects (`RefCounted`/`Resource`), not in the scene tree. A fixed-step tick is driven by a framework scheduler and is deterministic given seed + inputs.
- **Presentation** as Godot nodes that *observe* simulation state (via events and a read-only view). They issue *commands* back; they never mutate state directly.
- **Benefit for AI development:** the entire simulation can be tested headless, quickly and deterministically, with no rendering. "Agents verify actual behavior" becomes cheap and reliable for most gameplay logic. Rendering-level checks are then reserved for presentation.

**[RECOMMEND]** Adopt this as the framework's architectural spine. **Do not** adopt a generic ECS library: it fights Godot's node model and is premature. Use data-oriented, typed simulation structures instead.

### 2.6 GDScript performance vs. the genres you like

Factorio-, OpenTTD- and RimWorld-scale simulations tick thousands of entities. GDScript is an interpreted language, roughly an order of magnitude slower than C# for tight loops. Options:

| Option | For | Against |
|---|---|---|
| **A. GDScript only** | Simplest; best Godot integration; web export works. | Hard ceiling on simulation scale; may force game redesign later. |
| **B. GDScript + GDExtension (Rust or C++) for measured hot paths** | GDScript stays the default. Hot kernels (pathfinding, flow fields, bulk tick) go native only when profiling proves the need. Works on web. | Native toolchain and cross-compilation in CI. Agents writing Rust/C++ need stronger review. |
| **C. C# as primary** | Fast. Static typing catches more agent errors at compile time. LLMs have far more C# training data than GDScript. Mature testing (gdUnit4Net). | **Godot 4 C# still has no web export** (unchanged per 2026 sources). Heavier toolchain. Two languages in the ecosystem (addons, docs). |

**[RECOMMEND] B.** It's GDScript-first with static typing enforced (warnings-as-errors). In addition, run a **Phase 2 performance spike**: 10k simulated agents ticking, measure the frame budget. The result tells us whether native kernels are ever needed. This is decision D2, because C# would change tooling, testing and distribution.

### 2.7 "Provider independence" needs a precise definition

Total abstraction from vendors would mean building your own agent loop on raw model APIs, which is a large project. Vendor CLIs (Claude Code, Codex CLI, Gemini CLI, opencode…) already provide the agent loop, tool use, sandboxing and context compaction, and they improve monthly. Rebuilding that is waste.

**[RECOMMEND] Independence at the layers that hold *your* investment; vendor CLIs as replaceable executors.**

| Layer | Portable mechanism | Status (2026) |
|---|---|---|
| Persistent instructions | `AGENTS.md` hierarchy (Linux Foundation / AAIF standard; read natively by Codex, Copilot, Cursor, Gemini CLI, etc.). Claude Code reads `CLAUDE.md`, so ours contains `@AGENTS.md`. | Established |
| Reusable procedures | Agent Skills (`SKILL.md`, agentskills.io). Read by Claude Code, Codex, Gemini CLI and others. Portability isn't perfect (e.g. `allowed-tools` is parsed inconsistently). | Established, with caveats |
| Tools | MCP servers (Godot editor, repo tooling) + plain CLI commands. **Every critical capability must also exist as a plain CLI command**, so no agent is stuck without MCP. | Established |
| Roles, workflows, tasks, specs | Our own Markdown + YAML frontmatter in the repo. Owned by us, vendor-neutral. | Ours |
| Execution | Thin adapters: `claude -p`, `codex exec`, `gemini -p`, `opencode run`… invoked headless with role + task + context manifest. | Adapters are ours; the CLIs are swappable |
| Observability | Normalised run reports (our JSON schema) + optional OpenTelemetry. The GenAI semantic conventions are still marked *Development*, so they are not a foundation yet. | Experimental |

The rule: **nothing authoritative may exist only in a vendor-specific file.** `.claude/`, `.codex/`, `.gemini/` and `.agents/` contents are generated from `harness/` by a sync script, and CI checks they're in sync.

### 2.8 Orchestration: no framework, start with scripts

LangGraph, CrewAI, AutoGen and similar frameworks are built for constructing agents on raw APIs. With CLI agents as executors, they add a second agent runtime with no clear benefit. Worktree managers (Claude Squad, Emdash, Vibe Kanban…) are useful for *interactive* parallel work, but they are young. One has already shut down (Conductor, April 2026) and one is reported unmaintained. None should be a foundation.

**[RECOMMEND] Staged orchestration:**
- **O0 (Phase 0–1)**: you drive one interactive CLI session. Workflows are playbooks the agent follows. The independent review is a headless second-vendor call through a script. Parallelism comes from git worktrees.
- **O1 (once O0 workflows are stable)**: `gdf run <workflow> <task>`. This is a small Python runner (a few hundred lines) that executes workflow steps as headless CLI calls, each in its own worktree. It enforces gates, collects run reports, and stops at human-approval gates.
- **O2 (only on evidence)**: queues, scheduled runs, dashboards.

Workflows are defined as data from day one (YAML step lists in `harness/workflows/`), so O0→O1 is automation of existing definitions, not a redesign.

---

## 3. Missing requirements

Each of these has an architectural consequence. Defaults I'll assume unless you say otherwise:

| Gap | Why it matters | Proposed default |
|---|---|---|
| **Multiplayer** | Lockstep multiplayer requires bit-exact determinism, which floating-point GDScript doesn't give across platforms. This dictates the simulation core design. | **[DECIDE] D5.** Default: single-player. The simulation is *reproducible* (same seed + inputs → same result on the same build) but not *cross-platform lockstep-safe*. |
| **Modding** | RimWorld/Factorio-style games live on mods. Mod support requires data-driven content definitions and stable content IDs from the start. | **[RECOMMEND]** Content is data-driven (typed `Resource` definitions + stable string IDs) from day one. A mod *loader* is out of scope until a game needs it. |
| **Art & audio sourcing** | You won't program, and probably won't make art. AI-generated vs licensed vs commissioned affects the asset pipeline, licensing records, Steam disclosure rules, and repository size. | **[DECIDE]** Listed in §8 as open; it doesn't block stage 2. Asset provenance tracking (source, license, AI-generated flag) is mandatory regardless. |
| **Save compatibility** | Long sim games live and die on saves surviving updates. | Versioned save schema + migration chain as a framework foundation. |
| **Localization & accessibility** | Expensive to retrofit. | All user-facing strings via `tr()` keys from day one. Accessibility checklist in game quality standards. |
| **AI cost budget** | Parallel multi-agent runs can burn money quickly. | Per-run and per-day budget caps in the runner (O1). Cost recorded in every run report. |
| **Licensing of the framework** | Affects what it may contain and whether it could be open-sourced. | Proprietary/private by default; third-party addon licenses tracked in `THIRD_PARTY.md`. |
| **Hosting / remote / backup** | CI, PRs, branch protection and backup depend on it. | **[DECIDE] D3.** |
| **Distribution platforms** | Steam integration (achievements, cloud saves, Workshop) affects the platform service layer. | A platform-services interface in the framework; Steam adapter when the first game needs it. |

---

## 4. Governance & instruction inheritance (design sketch)

### 4.1 Three levels mapped onto the directory tree

```
AGENTS.md                         L1 entry: org rules summary + routing
governance/                       L1 authoritative documents
framework/AGENTS.md               L2 entry
framework/governance/             L2 authoritative documents
games/<slug>/AGENTS.md            L3 entry
games/<slug>/governance/          L3 authoritative documents
```

All major CLIs already do hierarchical `AGENTS.md`/`CLAUDE.md` discovery: root plus nearer directories, with nearer files appended. **Location therefore determines scope automatically.** An agent working in `games/foo/project/` gets L1 + L3. An agent in `framework/` gets L1 + L2. A game agent never loads framework *governance*, only the framework's public API docs.

### 4.2 Precedence and conflict rules (machine-checkable)

1. **Every normative rule has an ID and a level**: `ORG-SEC-003`, `FW-API-012`, `GAME-foo-ART-004`. Rules are written as `MUST`/`SHOULD`/`MAY` (RFC 2119).
2. **Lower levels may only *add* or *tighten*.** A lower-level rule that relaxes or contradicts a higher one is invalid unless an **exception record** exists. Exception records (`governance/exceptions/EXC-NNN.md`) name the rule ID, scope, justification, expiry and *your* approval.
3. **Conflict resolution order**: higher level wins → within a level, the more specific document wins (spec > standard > guideline) → otherwise the agent **stops and escalates**. It doesn't guess.
4. **`gdf context <path>`** prints the resolved instruction chain and applicable rule IDs for any file or task. Agents run it at task start, and the run report records which instruction versions applied.
5. **CI lint** (`gdf check governance`) fails on: duplicate rule IDs, references to non-existent rules, lower-level rules using override language without a valid exception, and expired exceptions.

### 4.3 Keeping entry files small

`AGENTS.md` files are **routing documents**, at most ~150 lines. They hold the non-negotiables, the commands, and where to look. Detailed standards live in referenced documents that are loaded on demand (progressive disclosure, the same principle as Skills). This protects the context budget. Long always-loaded instruction files measurably degrade agent performance.

### 4.4 Document authority

Every authoritative document has frontmatter with `id`, `owner` (human or role), `status` (draft/approved/superseded), `version`, `supersedes` and `applies-to`. Agents may *propose* changes to approved governance; only you approve them (protected-path rule, enforced by a hook/CI check, not by trust).

---

## 5. Target architecture (proposal)

### 5.1 Repository layout

```
<repo-root>/
├── AGENTS.md  CLAUDE.md  README.md
├── governance/                     L1: constitution, rules/, decisions/ (org ADRs), exceptions/
├── harness/                        AI production system (a product too; versioned with the repo)
│   ├── roles/                      role definitions (5 to start)
│   ├── workflows/                  workflow definitions (YAML steps + playbook prose)
│   ├── skills/                     canonical SKILL.md sources
│   ├── templates/                  spec, ADR, task, test plan, release notes, …
│   ├── tools/                      `gdf` CLI (Python): verify, context, sync, vendor, run, release
│   ├── adapters/                   vendor CLI adapters + generator for .claude/ .agents/ etc.
│   └── evals/                      regression tasks to measure harness/agent quality
├── framework/
│   ├── AGENTS.md
│   ├── governance/                 L2: architecture rules, API conventions, dependency rules
│   ├── product/                    vision, roadmap, backlog/
│   ├── decisions/                  framework ADRs
│   ├── workbench/                  THE framework Godot project (current root project moves here)
│   │   ├── project.godot
│   │   ├── addons/gdf_<module>/    one folder per module: code, module.toml, README, spec.md, tests/, demo/
│   │   └── scenarios/              cross-module integration scenarios
│   └── releases/                   CHANGELOG, release notes
├── games/<slug>/
│   ├── AGENTS.md
│   ├── governance/                 L3: game constitution, design pillars, constraints
│   ├── product/ design/ specs/ decisions/
│   ├── framework.lock              pinned framework version + module list + checksums
│   └── project/                    the game's Godot project
│       ├── project.godot
│       ├── addons/gdf_*/           VENDORED framework release (read-only, CI-verified)
│       ├── src/  assets/  tests/  scenarios/
├── knowledge/                      godot/, patterns/, research/, lessons/ (non-normative)
└── runs/                           gitignored: agent run reports, logs, screenshots
```

### 5.2 Framework distribution inside the monorepo

Godot cannot load code from outside a project's `res://`, so framework code must physically be inside each game project. Options:

| Option | Version pinning | Isolation | Verdict |
|---|---|---|---|
| Directory junction/symlink into each game | **None.** Every game is always on framework HEAD; one framework edit can break all games. | Poor | Rejected (violates BOOTSTRAP §5) |
| Git submodule per game | Yes | Good | Awkward in a monorepo (submodule of itself), poor agent ergonomics |
| Godot Asset Store / separate repo per addon | Yes | Good | Overhead for one person; breaks "one monorepo" |
| **Vendored release copy + lockfile + checksum** | **Yes.** `gdf vendor games/foo --framework 0.4.0` copies the tagged release's selected modules (tests/demos stripped) and writes `framework.lock` with hashes. | **Good.** CI fails if a vendored file differs from the locked release, which blocks "freely modifying framework internals". | **[RECOMMEND]** |

Consequences:
- **Single release train.** One framework version (SemVer, tag `framework/v0.4.0`) with per-module *maturity* labels. Per-module versioning is unnecessary overhead for a one-person studio.
- **Global namespace hygiene.** GDScript `class_name` is global per project. All framework classes are prefixed (`Gdf…`) and lint-enforced.
- **Downstream canary.** Framework CI tries each active game against framework HEAD and reports breakage early. It is non-blocking and informational, while the games stay pinned.
- **Game-local patches** to framework code are forbidden. A needed fix goes upstream as a framework task, or the game wraps or extends it in its own `src/`.

### 5.3 Module maturity

| Level | Meaning | Promotion criteria |
|---|---|---|
| **Experimental** | Exists in the workbench; API may change any release. Games may use it with an explicit opt-in in `framework.lock`. | Spec + tests + one consumer. |
| **Approved** | API reviewed. Breaking changes need a deprecation cycle of one minor release. | Used by ≥1 game in production code; coverage threshold met; docs + demo scene; T2 review; your approval. |
| **Stable** | SemVer-strict; breaking changes only in a major release. | Used by ≥2 games or ≥1 shipped game; no API change for N releases; performance budget documented. |

`module.toml` declares id, maturity, dependencies (on other modules, acyclic and layer-checked), and supported Godot range. `gdf check deps` enforces the layering: foundation ← simulation ← gameplay ← presentation; never upward; framework never imports from games.

### 5.4 Godot automation & verification stack

| Capability | Tooling | Maturity |
|---|---|---|
| Engine pinning | `.godot-version` per project + `gdf godot` resolver (downloads/locates the exact build) | Ours |
| Parse/type check | `godot --headless --check-only` + project-wide GDScript warnings-as-errors (untyped declarations, unsafe calls…) | Established |
| Lint/format | gdtoolkit 4.x (`gdlint`, `gdformat --check`) | Established |
| Unit + scene tests | **gdUnit4** (v6.2.x supports Godot 4.5–4.7.1; v7 targets 4.7+/4.8). Scene runner with input simulation, mocks/spies, JUnit + HTML reports, GitHub Action. GUT 9.x is the alternative. | Established |
| Import / resource validation | `godot --headless --import` + custom validator (missing deps, broken UIDs, oversized textures, asset provenance metadata) | Ours on established base |
| Scenario runs (behavior verification) | Framework `gdf_testkit`: load scenario → scripted inputs/commands → tick N → assert state → emit JSON report. Headless for simulation; rendered for presentation. | Ours |
| Screenshots / video | Rendered (non-headless) run; `--write-movie` for deterministic frame capture at fixed FPS. On Linux CI: Xvfb + software Vulkan (lavapipe) or the GL compatibility renderer. | Established; CI rendering is fiddly |
| Visual judgement | Multimodal model reviews screenshots against the spec. **Advisory only**, never a gate. | Experimental |
| Performance | Scenario-based benchmarks with budgets (tick ms, frame ms, memory) stored per release; regression thresholds fail CI | Ours on Godot's `Performance` monitors |
| Live editor control | Godot MCP servers (e.g. `godot-editor-mcp`: run, input, screenshots, profiling, ~190 tools, gated toolsets). Useful for interactive debugging. **Not on the critical path**: everything CI needs is plain CLI. | Experimental / fast-moving |

A single entry point, `gdf verify [scope]`, runs the same checks locally and in CI. That one script is the definition of "green".

---

## 6. Tooling recommendations (summary)

| Area | Recommendation | Alternatives considered |
|---|---|---|
| Repo tooling language | **Python 3.12+ via `uv`** (needed anyway for gdtoolkit/MCP; good agent fluency) | GDScript tools via `godot -s` (limited); Node (no ecosystem advantage here) |
| VCS | Git + **Git LFS for binary assets from commit 1** | Plain git (repo bloat is irreversible without history rewrite) |
| Primary interactive agent | Claude Code (already in use) | Codex CLI, Gemini CLI, opencode (multi-provider single CLI) |
| Independent reviewer | A **different vendor's** CLI headless (e.g. Codex CLI) | Same model with a different prompt (weaker independence) |
| Isolation | git worktrees per agent task (native in Claude Code and Codex; scriptable for others) | Containers (later, for unattended runs) |
| CI | **[DECIDE] D3**: GitHub Actions is the natural fit (gdUnit4 action, Godot images, LFS) | GitLab CI; local-only runner |
| Task tracking | **[DECIDE] D3**: files in repo vs GitHub Issues | |

---

## 7. Risk register (top 12)

| # | Risk | Likelihood / impact | Mitigation |
|---|---|---|---|
| R1 | Framework built ahead of games; no game ships | High / critical | Harvest model (§2.1); framework scope is foundations-only until game 1 prototype exists |
| R2 | Building the harness/orchestrator becomes the project | High / high | Staged O0→O1→O2; each stage gated on evidence of need; time-boxed phases |
| R3 | False green: tests pass, game is broken or not fun | High / high | Verification-first; scenario runs; screenshots; human playtest gate; reviewer ≠ implementer |
| R4 | GDScript too slow for sim-heavy designs | Medium / high | Sim/presentation split; perf spike in Phase 2; GDExtension escape hatch |
| R5 | Approval fatigue makes you the bottleneck | High / medium | Risk tiers; batch approvals; concise decision briefs |
| R6 | Instruction/doc rot and context bloat | High / medium | Small routing AGENTS.md; doc ownership + status; doc-lint; lessons-learned are reviewed and folded into rules, not appended forever |
| R7 | Vendor lock-in creeping in via tool-specific files | Medium / medium | Canonical `harness/` + generated adapters + sync check in CI |
| R8 | Godot vendoring pitfalls (UID conflicts, import cache, `class_name` collisions) | Medium / medium | Prefix rule; vendoring tool regenerates import state; workbench CI imports clean |
| R9 | Binary assets bloat git | High if not handled / high | LFS from first commit; asset size lint |
| R10 | Windows dev vs Linux CI divergence (case, paths, rendering) | Medium / medium | Case-sensitivity lint; CI is the authority; rendering tests tolerant (perceptual diff) |
| R11 | AI cost runaway in parallel runs | Medium / medium | Budget caps per run/day; cost in reports |
| R13 | Determinism broken silently (a float or engine call leaks into sim state) → multiplayer desyncs found late | High / critical | Import-ban lint for sim modules; cross-platform hash test in CI; replay-based regression tests |
| R14 | Fixed-point GDScript sim too slow for target scale | Medium / high | Kernel boundaries; perf spike in fixed-point; GDExtension option |
| R12 | Fast-moving ecosystem (MCP servers, orchestrators) invalidates choices | High / low–medium | Only standards (AGENTS.md, SKILL.md, MCP, git) are load-bearing; tools are adapters |

---

## 8. Decisions

### 8.0 Owner answers (2026-11-01) and their consequences

| ID | Owner answer | Consequence for the architecture |
|---|---|---|
| D1 | **Harvest.** Build what we need inside a game first, but always build for reuse. A deliberate, informed decision may set reuse aside. | Game code follows the *same* module structure and rules as the framework (`games/<slug>/project/modules/<name>/` with `module.toml`, maturity `game-local`). Harvesting is then a move plus a review, not a rewrite. Setting reuse aside requires a recorded decision (a "reuse waiver" ADR) so it is never accidental. |
| D2 | **GDScript first**, with the option to refactor to C# where performance requires it. (Owner asked whether they interoperate. See 8.0.1.) | Performance-critical code sits behind narrow, batch-oriented *kernel* interfaces, so its implementation language can change without touching callers. The native option (C# vs GDExtension) is decided by ADR after the performance spike, not now. |
| D3 | **GitHub + Issues.** | Work items live in GitHub Issues/Projects. **Specs, ADRs and acceptance criteria remain in the repo**: an issue *links to* its spec; the spec is the contract. Agents use `gh`. Issue forms mirror the repo templates. Labels per product (`product:framework`, `product:<game>`), milestones = releases. |
| D5 | **Multiplayer is required**: co-op and competitive (RimWorld co-op, OpenTTD multiplayer, Settlers). | These reference games all use **deterministic lockstep**. This is the single most constraining requirement in the system. See 8.0.2. |
| D4 | Not asked. | Proceeding with the recommendation: Claude Code primary + a second vendor (Codex CLI) as independent reviewer, behind adapters. |

#### 8.0.1 GDScript ↔ C# interoperability: the honest answer

**Yes, they can coexist in one project and call each other.** There are caveats that shape the architecture:

- C# requires the **.NET build of Godot** (editor + export templates) for that project. The installed `Godot_v4.7.1-stable_win64.exe` is the standard build.
- **Godot 4 C# still cannot export to Web** (as of 2026 sources; to be re-checked at decision time). Any game that contains C# loses the web platform.
- Cross-language calls go through Godot's Variant layer (`Call()`, signals, properties). Static typing stops at the boundary and each call has marshalling overhead. **Chatty per-entity calls across the boundary can be slower than staying in GDScript.** C# pays off only for *batch* work: whole-tick updates, pathfinding batches, bulk arrays (`PackedInt64Array` etc.).
- If the **framework** contained C#, every game using it would inherit the .NET requirement and lose web export.

Therefore:
- **Framework code stays GDScript.** If framework hot paths need native speed, the candidate is **GDExtension** (Rust via godot-rust/gdext, or C++). It works on web and mobile and avoids the .NET requirement.
- **A game may adopt C#** for its own hot paths through a game-level ADR that accepts the platform trade-off.
- Either way, the design rule that makes the switch cheap is the **kernel boundary**: a narrow interface that passes packed arrays and runs once per tick, not once per entity.

#### 8.0.2 Consequences of deterministic lockstep multiplayer

Lockstep means every client runs the same simulation from the same inputs and must stay **bit-identical** across machines (PC ↔ PC, possibly PC ↔ mobile). That turns several recommendations into hard rules:

1. **Simulation state uses integer / fixed-point math only.** Float results (especially `sin`, `cos`, `pow`, and FMA-contracted operations) can differ across CPUs, compilers and platforms. OpenTTD-style integer simulation is the proven approach. The framework provides `gdf_fixed` (fixed-point scalars and vectors, deterministic trig tables). Floats are allowed in presentation only.
2. **The simulation owns everything that affects outcomes**: its own seeded PRNG (integer-based), its own pathfinding with integer costs, and spatial queries. **Godot physics (Jolt), NavigationServer, `RandomNumberGenerator`, `Time`, and frame delta must never influence simulation state.** These are lint-enforced import bans in simulation modules.
3. **Deterministic iteration order.** Entity IDs are monotonic integers. Collections are iterated in defined order. No iteration over anything whose order depends on hashing or memory addresses.
4. **Command pipeline.** All player and AI actions become serialisable *commands*, scheduled for a future tick (input delay). The presentation layer never mutates simulation state. This is the same rule as §2.5, now a correctness requirement.
5. **State hashing and desync detection.** Every N ticks the simulation produces a canonical state hash, and clients compare them. This gives us, nearly for free:
   - **Replays** (initial state + command log): perfect bug reproduction for agents.
   - **A determinism test gate in CI**: same scenario on Windows and Linux (and ARM when mobile matters), and hashes must match.
6. **Full state serialisation** is needed for late join, reconnect and desync recovery. That is the same code as save/load, so the save system and network sync share one canonical serialiser.
7. **Performance pressure rises.** Every client runs the full simulation, and fixed-point math in GDScript is slower than floats. The Phase 2 performance spike becomes mandatory and must run in fixed-point.
8. **These foundations must exist before the first multiplayer-relevant game code.** Determinism cannot be retrofitted. They belong to framework v0.1 even under the harvest model: they are foundations, not gameplay.
9. **Transport is separate from the lockstep protocol.** Godot ENet for PC and WebRTC for web are the defaults. A Steam transport adapter can be added later. The lockstep layer doesn't care which transport carries the commands.
10. **Not every game must be lockstep.** A game may declare a server-authoritative model in its ADR. The framework's command/tick architecture supports both, but the deterministic core is the default.

### 8.1 Decisions that materially affect the architecture (as originally posed)

| ID | Decision | My recommendation |
|---|---|---|
| **D1** | Framework strategy: *harvest from games* vs *build framework capabilities ahead of games* | Harvest; foundations-only framework v0 |
| **D2** | Implementation language: GDScript only / GDScript + GDExtension escape hatch / C# primary | GDScript + GDExtension escape hatch, perf spike in Phase 2 |
| **D3** | Hosting, CI and task tracking: GitHub (PRs + Actions + Issues) / GitHub + repo-file tasks / local-only | GitHub private repo + Actions; tasks as repo files |
| **D4** | Agent vendor baseline: Claude Code primary + second vendor as reviewer / several vendors equal from day 1 / one multi-provider CLI (opencode) | Claude Code primary + Codex CLI as independent reviewer; adapters keep it swappable |
| **D5** | Multiplayer ever in scope? (determinism requirements) | Single-player; reproducible but not lockstep-safe |

### 8.2 Open but not blocking stage 2

- Art/audio sourcing strategy (AI-generated / licensed / commissioned) and Steam AI-content disclosure.
- Repository / studio name (affects `gdf` prefix and paths).
- Framework license (default: private/proprietary).

### 8.3 Recommendations I'll proceed with unless you object

Plain folder names (§2.2) · five roles (§2.3) · risk-tiered lifecycle with verification-first and a prototype lane (§2.4) · sim/presentation split, no ECS library (§2.5) · AGENTS.md hierarchy with rule IDs and exception records (§4) · vendored framework releases with lockfile (§5.2) · single framework release train with maturity labels (§5.3) · gdUnit4 + gdtoolkit + `gdf verify` (§5.4) · Python/uv tooling, git LFS (§6) · staged orchestration without an agent framework (§2.8).

---

## 9. What stage 2 will produce

1. Blueprint v1.0 (all 22 sections from BOOTSTRAP §15), incorporating your decisions.
2. The initial repository document set, with responsibilities and relationships (constitution, rules, root/scope AGENTS.md files, framework vision and architecture, harness role/workflow definitions).
3. The eight templates (feature spec, component spec, ADR, agent definition, workflow definition, task, test plan, release notes).
4. A phased roadmap with exit criteria per phase. Sketch:
   - **P0 Foundations**: git + LFS, layout migration, Godot pinning, `gdf verify` skeleton, governance v1, AGENTS.md hierarchy.
   - **P1 Harness MVP**: roles, templates, workflows as playbooks, cross-vendor review, worktrees, adapter sync.
   - **P2 Framework v0.1**: foundation modules + testkit + scenario runner + perf spike, in the workbench.
   - **P3 Game 001 prototype lane**: *the first game gets chosen here*; harvesting starts.
   - **P4 Orchestrator O1 + harness evals**, driven by friction observed in P1–P3.

No game design, game mechanics or game code is produced in either stage.

---

## Sources

Research quality note: much of the 2026 coverage is secondary (blogs, aggregators). Claims used above were cross-checked where it mattered (Godot releases, gdUnit4 compatibility, Claude Code AGENTS.md handling). Tool counts and adoption figures are indicative only.

- Godot 4.7 stable / 4.7.1: https://github.com/godotengine/godot/releases/tag/4.7-stable · https://godotengine.org/article/maintenance-release-godot-4-7-1/ · https://godotengine.org/article/release-candidate-godot-4-7-rc-1/
- gdUnit4 (compatibility table, CLI/CI, JUnit/HTML, scene runner, C#): https://github.com/godot-gdunit-labs/gdUnit4
- GUT: https://github.com/bitwes/Gut · https://gut.readthedocs.io/
- gdtoolkit: https://pypi.org/project/gdtoolkit · https://github.com/Scony/godot-gdscript-toolkit
- godot-editor-mcp: https://pypi.org/project/godot-editor-mcp/
- AGENTS.md standard: https://codex.danielvaughan.com/2026/04/07/agents-md-open-standard-cross-tool-portability · https://rywalker.com/research/agents-md-standard
- Claude Code memory / `@AGENTS.md` import: https://code.claude.com/docs/en/memory
- Agent Skills standard: https://codex.danielvaughan.com/2026/05/05/agent-skills-open-standard-portable-skills-codex-cli-cross-agent/ · https://mcp.directory/blog/cross-agent-skills-cursor-codex-cline-antigravity-gemini-mastra-portability
- Worktree orchestrators landscape: https://nimbalyst-website.pages.dev/blog/best-git-worktree-tools-ai-coding-2026/ · https://augmentcode.com/tools/open-source-agent-orchestrators · https://codex.danielvaughan.com/2026/04/11/agentmaxxing-parallel-multi-cli-orchestration/
- OpenTelemetry GenAI conventions status: https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke · https://www.dash0.com/knowledge/opentelemetry-genai-semantic-conventions-explained
- Godot C# web export status: https://forum.godotengine.org/t/is-there-an-update-on-exporting-c-projects-to-web/128821
- Shared addons across Godot projects: https://github.com/bitwes/Gut/issues/472 · https://forum.godotengine.org/t/gd-sync-dock-for-managing-shared-addons/142439
