# Task: Design a Game Development Organization and AI Agent Harness

## 1. Your role

Act as a principal software architect, AI agent systems architect, game technology architect, and engineering operations designer.

Your task is to design a comprehensive, state-of-the-art development environment for building games primarily through AI agents.

Do not give me a generic software development department template or a superficial collection of prompts.

I want a technically sound, practical, maintainable, AI-native development system that I can use for years and across multiple games.

Challenge my assumptions. Identify architectural risks, missing requirements, unnecessary complexity, and better alternatives.

Do not blindly agree with my ideas.

## 2. My background and objectives

I have a software engineering background but very limited game development experience.

I have designed a few simple games, but I will not personally program the games. AI agents will perform the implementation work.

My responsibilities will primarily be:
- Product ownership.
- Creative direction.
- Game design decisions.
- Architecture and technical direction approval.
- Prioritization.
- Reviewing proposed solutions.
- Approving important decisions and deliverables.

My two primary objectives are:

1. Create genuinely excellent, polished indie games.
2. Simultaneously build a reusable library of independent game development components, systems, tools and capabilities that can be used in future games.

I also want a repeatable development workflow that becomes increasingly effective as I gain experience and accumulate reusable technology.

The framework itself must be treated as an independent product, with its own lifecycle, roadmap, architecture, backlog, testing, releases, versioning and documentation.

Individual games are separate products that consume released framework capabilities.

## 3. Game technology direction

The chosen game engine is **Godot**.

Godot 4 is the intended technology generation.

GDScript is the initial preferred implementation language. C# may be reconsidered if justified by concrete technical requirements.

Primary platform:
- PC.

Secondary platforms:
- Mobile.
- Web, if technically practical.

The architecture should support games with:
- 3D worlds.
- Isometric perspectives.
- First-person perspectives.
- Third-person perspectives.

Different presentation and interaction models should be supported without unnecessarily coupling them to the underlying simulation and gameplay systems.

The framework should not become a universal game engine. It should provide reusable, composable capabilities on top of Godot.

## 4. Game references and creative direction

The following games represent the kinds of experiences I find interesting:

- Against the Storm.
- Darkest Dungeon.
- Factorio.
- Timberborn.
- OpenTTD.
- RimWorld.

These are references, not requirements to copy their mechanics.

They demonstrate my interest in:
- Strategy.
- Simulation.
- RPG systems.
- Colony and settlement management.
- Resource management.
- Production chains.
- Logistics.
- Transportation networks.
- Autonomous agents.
- Emergent gameplay.
- Character progression.
- Complex interacting systems.

The first game has deliberately NOT been selected.

Do not turn this task into the design of a particular game. The objective is to establish the development organization and its supporting infrastructure independently of the first game's concept.

## 5. Fundamental organizational structure

Everything should live inside one primary working folder and preferably one Git monorepository.

The environment should contain four independently managed areas:

1. Governance.
2. Game Development Framework.
3. AI Production System.
4. Individual Game Products.

The framework must have its own product identity and lifecycle.

Games must be able to consume released framework versions without freely modifying framework internals.

Framework changes must not accidentally break existing games.

Game-specific requirements and implementations must not contaminate the reusable framework.

The system should support multiple games over time.

## 6. Governance hierarchy

Establish three levels of governance:

### Level 1: Organization-wide governance

Applies to all agents, framework development and games.

Examples:
- Engineering principles.
- Agent behavior.
- Quality requirements.
- Security and permissions.
- Documentation standards.
- Decision-making authority.
- Development policies.

### Level 2: Framework governance

Applies to the reusable framework.

Examples:
- Architecture.
- Component contracts.
- Dependency rules.
- API conventions.
- Reusability requirements.
- Compatibility guarantees.
- Framework release standards.

### Level 3: Game-specific governance

Applies to an individual game.

Examples:
- Game vision.
- Design principles.
- Game architecture.
- Gameplay specifications.
- Art direction.
- Game-specific quality requirements.
- Game-specific technical constraints.

Lower-level governance may introduce additional requirements and restrictions but must not silently override higher-level governance.

Design a precise mechanism for instruction inheritance, precedence, conflict resolution and agent awareness of applicable instructions.

## 7. AI agent architecture

The system must support multiple AI models and providers.

Provider independence should be a foundational architectural requirement.

I do not want the entire development environment tied to one AI vendor.

Consider a CLI-centric architecture with a platform-independent orchestration layer.

Investigate the best current options rather than assuming that this is necessarily the optimal solution.

The system should support specialized agent roles, potentially including:

- Product owner assistant.
- Game designer.
- Technical architect.
- Framework architect.
- Implementation developer.
- QA engineer.
- Code reviewer.
- Technical writer.
- Build and release engineer.
- Research agent.
- Integration agent.

Do not assume every role requires a permanently running agent.

Distinguish between:
- Agent definitions.
- Agent instances.
- Workflows.
- Orchestration.
- Tools and permissions.
- Shared knowledge and persistent instructions.

Agents should be instantiated when needed and receive the context relevant to their tasks.

The system should support parallel execution where appropriate.

## 8. Persistent instructions and knowledge

Agents must have persistent instructions and access to authoritative project knowledge.

Avoid relying on conversation history as the source of truth.

Design a hierarchical documentation and instruction system.

Consider:
- Root agent instructions.
- Organization-wide governance.
- Framework instructions.
- Game-specific instructions.
- Agent-specific role definitions.
- Workflow instructions.
- Architecture documentation.
- Technical specifications.
- Architecture Decision Records.
- Glossaries.
- Research references.
- Lessons learned.

Context should be supplied selectively rather than dumping the entire repository into every agent's prompt.

Design mechanisms for agents to discover relevant documentation and verify which instructions apply.

Documentation must have ownership, versioning and clear authority.

## 9. Development workflows

Design repeatable workflows for:

- New framework capabilities.
- New reusable components.
- New game projects.
- New game features.
- Bug fixing.
- Refactoring.
- Architecture changes.
- Research and technical evaluation.
- Code reviews.
- Testing.
- Releases.
- Documentation updates.

A proposed default feature lifecycle is:

1. Request.
2. Analysis.
3. Requirements and specification.
4. Architecture review.
5. Human approval where required.
6. Implementation.
7. Automated verification.
8. Independent review.
9. Integration.
10. Documentation and reporting.

Challenge and improve this process.

The implementing agent must not be the sole authority deciding whether its own work is correct.

Specifications should be treated as contracts with explicit acceptance criteria.

## 10. Quality and verification

Quality is critical.

Design an automated quality system that can verify:

- Code correctness.
- Architecture compliance.
- Dependency boundaries.
- Test coverage.
- Performance.
- Build integrity.
- Game runtime behavior.
- Asset integrity.
- Documentation completeness.
- Compatibility between framework releases and games.

Consider Godot-specific tools and workflows, including:
- Headless execution.
- Automated tests.
- GDScript static analysis.
- Scene and resource validation.
- Runtime error detection.
- Screenshot and video capture.
- Performance profiling.
- Continuous integration.

Investigate relevant testing frameworks such as GUT and GdUnit4.

The system must not equate successful code generation with successful implementation.

Agents should be able to run the game, inspect failures and verify actual behavior.

## 11. Framework architecture and component strategy

The framework should provide independent, composable systems.

Potential areas include:

- Core infrastructure.
- Events and messaging.
- Data management.
- Persistence.
- Simulation time and scheduling.
- Entity and component architecture.
- World representation.
- Spatial systems.
- Navigation and pathfinding.
- Autonomous agents.
- Needs and relationships.
- Resource management.
- Economy.
- Production and crafting.
- Logistics.
- Inventory.
- Combat.
- Progression.
- Quests.
- UI.
- Camera and presentation.
- Audio and animation.

These are candidate capabilities, not a commitment to build all of them.

Establish principles for:
- Module boundaries.
- Interfaces.
- Dependencies.
- Reusability.
- Testing.
- Documentation.
- Versioning.
- Compatibility.
- Experimental versus stable components.

Avoid premature abstraction and framework development without demonstrated needs.

A reusable component should ideally have:
- A clear responsibility.
- A defined interface.
- Minimal dependencies.
- Automated tests.
- Documentation.
- A demonstration project.
- Versioning and compatibility rules.

Consider three maturity levels:
- Experimental.
- Approved.
- Stable.

Design the promotion process between them.

## 12. Framework and game product lifecycles

The framework must have its own:
- Vision.
- Roadmap.
- Backlog.
- Requirements.
- Architecture.
- Specifications.
- Development process.
- Testing.
- Releases.
- Versioning.
- Documentation.
- Maintenance.

Individual games should have corresponding product management structures.

Games must explicitly declare their framework version.

Investigate suitable approaches for framework distribution and dependency management inside a monorepository.

## 13. Initial repository structure

Use the following as a starting point, not a rigid requirement:

game-development/
|
|-- AGENTS.md
|-- README.md
|
|-- 00-governance/
|   |-- constitution.md
|   |-- engineering-standards.md
|   |-- quality-standards.md
|   |-- agent-policy.md
|   |-- decision-making.md
|
|-- 01-framework/
|   |-- architecture/
|   |-- specifications/
|   |-- components/
|   |-- tools/
|   |-- documentation/
|   |-- tests/
|
|-- 02-production-system/
|   |-- agent-definitions/
|   |-- workflows/
|   |-- templates/
|   |-- automation/
|   |-- integrations/
|
|-- 03-games/
|   |-- game-001/
|   |   |-- AGENTS.md
|   |   |-- game-constitution.md
|   |   |-- design/
|   |   |-- architecture/
|   |   |-- specifications/
|   |   |-- assets/
|   |   |-- source/
|   |   |-- tests/
|   |   |-- documentation/
|
|-- 04-shared-knowledge/
|   |-- godot/
|   |-- patterns/
|   |-- research/
|   |-- lessons-learned/
|
|-- 05-operations/
    |-- tasks/
    |-- decisions/
    |-- reports/
    |-- releases/

Evaluate whether this structure is optimal and propose improvements.

## 14. Your research requirements

This must be a state-of-the-art design, not a naive default answer.

Research current tools, platforms and approaches.

In particular, investigate:

- Current AI coding agents and CLI tools.
- Multi-agent orchestration.
- Model/provider abstraction.
- Persistent agent instructions.
- Context engineering.
- MCP and tool integration.
- Git-based agent workflows.
- Parallel execution and isolated workspaces.
- Godot automation.
- Testing and CI/CD.
- Agent evaluation and observability.

Use current primary documentation and credible technical sources.

Do not recommend products merely because they are popular.

Compare alternatives and explain trade-offs.

Separate established capabilities from experimental approaches.

Avoid introducing unnecessary infrastructure, microservices or complex orchestration frameworks without a concrete justification.

## 15. Expected deliverables

Produce a comprehensive **Game Development Organization Blueprint v1.0**.

It should contain:

1. Executive summary.
2. Design principles.
3. Architectural overview.
4. Proposed working-folder structure.
5. Governance hierarchy.
6. Instruction inheritance and conflict resolution.
7. Framework product architecture.
8. Game product architecture.
9. AI agent architecture.
10. Agent role definitions.
11. Orchestration architecture.
12. Knowledge and context management.
13. Development workflows.
14. Specification and documentation standards.
15. Tooling and technology recommendations.
16. Godot integration strategy.
17. Testing and quality assurance.
18. Security and permission model.
19. Observability and reporting.
20. Versioning and release strategy.
21. Risks and trade-offs.
22. Phased implementation roadmap.

Also produce a proposed initial set of actual repository documents, with their responsibilities and relationships.

Include templates for:
- Feature specifications.
- Component specifications.
- Architecture Decision Records.
- Agent definitions.
- Workflow definitions.
- Task descriptions.
- Test plans.
- Release notes.

Distinguish between:
- Decisions already established by me.
- Your recommendations.
- Decisions that still require my approval.

## 16. How to approach the task

Do not immediately generate hundreds of files or an enormous speculative architecture.

Work in deliberate stages.

First:
- Analyze the requirements.
- Identify missing decisions.
- Research current alternatives.
- Identify architectural risks.
- Propose the target architecture.

Then:
- Produce the complete blueprint.
- Identify all necessary supporting documents.
- Establish the initial repository structure.
- Define the implementation roadmap.

Where decisions are not essential, make a well-reasoned recommendation rather than asking endless questions.

Where a decision materially affects the architecture, explain the alternatives and ask me.

Do not design the first game yet.

Do not start implementing game mechanics.

Do not write actual game code.

The immediate objective is to design the organization, framework product structure and AI development harness that will support many future games.

## 17. Quality expectations

I want something that could serve as the foundation for a serious long-term indie game development operation.

Be specific and technically grounded.

Prefer explicit interfaces, machine-verifiable rules, documented decisions and automated validation over vague instructions.

Design for maintainability, modularity, provider independence and gradual evolution.

Do not confuse a large number of agents with a good agent architecture.

Do not confuse a large documentation repository with good knowledge management.

Do not confuse reusable code with genuinely reusable components.

The result should be practical enough to implement incrementally, but architecturally sound enough that I will not need to redesign everything after my first game.

Your job is to help me build a development system that gets better with every game I make.