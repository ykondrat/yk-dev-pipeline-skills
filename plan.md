# Agent-per-Phase Refactoring — Implementation Plan

## Summary
- **Total tasks**: 16
- **Estimated phases**: 4
- **Spec reference**: [spec.md](spec.md)
- **Design reference**: [design doc](docs/plans/2026-04-16-agent-refactoring-design.md)

## File Structure (New/Modified Files)

```
plugins/yk-dev-pipeline/
├── .claude-plugin/plugin.json              # MODIFIED: version 3.0.0
├── agents/                                 # NEW: 7 agent definitions
│   ├── brainstorm.md                       # ~450 lines
│   ├── investigation.md                    # ~430 lines
│   ├── planning.md                         # ~350 lines
│   ├── implementation.md                   # ~530 lines
│   ├── code-review.md                      # ~550 lines
│   ├── testing.md                          # ~520 lines
│   └── documentation.md                    # ~380 lines
├── references/                             # NEW: shared reference library
│   ├── brainstorm/
│   │   └── creativity-techniques.md        # COPY from skills/brainstorm/references/
│   ├── investigation/
│   │   └── investigation-patterns.md       # COPY from skills/investigation/references/
│   ├── implementation/
│   │   ├── clean-code-principles.md        # COPY from skills/implementation/references/
│   │   ├── js-ts-best-practices.md
│   │   ├── implementation-standards.md
│   │   ├── security-patterns.md
│   │   ├── databases-sql.md
│   │   ├── databases-nosql.md
│   │   ├── databases-redis.md
│   │   ├── frameworks-web.md
│   │   └── frameworks-frontend.md
│   ├── code-review/
│   │   └── review-checklists.md            # COPY from skills/code-review/references/
│   ├── testing/
│   │   └── test-patterns.md                # COPY from skills/testing/references/
│   └── documentation/
│       └── doc-templates.md                # COPY from skills/documentation/references/
├── skills/                                 # KEPT: backward compatible
│   ├── SKILL.md                            # MODIFIED: orchestrator delegates to agents
│   └── {phase}/SKILL.md                    # KEPT AS-IS (all 7 phase skills)
├── CLAUDE.md                               # MODIFIED: document new structure
├── TESTING.md                              # MODIFIED: add agent testing
├── .claude-plugin/marketplace.json         # MODIFIED: version 3.0.0
└── examples/
    └── pipeline-state.example.json         # MODIFIED: add v3.0 fields
```

## Phase 1: Infrastructure

### Task 1: Create shared references directory
- **Files**: `plugins/yk-dev-pipeline/references/` (entire tree)
- **Depends on**: none
- **Test mode**: no-test
- **Description**: Create the `references/` directory at the plugin root. Copy all reference files from their current `skills/{phase}/references/` locations. Preserve exact content — no modifications to reference file contents. Directory structure mirrors current skill structure: `references/brainstorm/`, `references/investigation/`, `references/implementation/`, `references/code-review/`, `references/testing/`, `references/documentation/`.
- **Acceptance criteria**:
  - All 14 reference files copied to new locations
  - File contents identical to originals
  - Original files in `skills/{phase}/references/` untouched
  - Directory structure: `references/{phase}/{file}.md`

### Task 2: Update plugin.json manifest
- **Files**: `plugins/yk-dev-pipeline/.claude-plugin/plugin.json`
- **Depends on**: none
- **Test mode**: no-test
- **Description**: Update version to 3.0.0 and description to mention agents. The plugin.json is minimal — Claude Code auto-discovers agents from the `agents/` directory.
- **Acceptance criteria**:
  - Version is "3.0.0"
  - Description mentions agent-based pipeline

## Phase 2: Agent Definitions (Core)

Each agent file follows this structure:
```markdown
---
name: {phase-name}
description: {when to use this agent}
model: {opus|sonnet}
tools: {scoped tool list}
effort: {high|medium}
maxTurns: {N}
memory: project
color: {color}
---

# {Phase Name} Agent

{Role description}

## Your Process
{Adapted from current SKILL.md — pipeline context removed, standalone capable}

## References
{How to load reference files using Read tool — paths will be discovered via Glob}

## Improvements
{New capabilities from research}

## Self-Verification Gate
{Checklist to verify before completing}

## Output Artifacts
{What to create/update on disk}
```

### Task 3: Create a brainstorm agent
- **Files**: `plugins/yk-dev-pipeline/agents/brainstorm.md`
- **Depends on**: Task 1 (references must exist for path references)
- **Test mode**: no-test
- **Description**: Create the brainstorm agent definition. Adapt content from `skills/brainstorm/SKILL.md` (413 lines). Remove pipeline context section. Add agent frontmatter (model: opus, tools: Read/Grep/Glob/WebSearch/WebFetch/Bash/Edit/Write, effort: high, maxTurns: 50, color: blue, memory: project). Add reference discovery instructions (Glob for creativity-techniques.md). Add three new improvements: (1) constraint-based exploration — systematically vary constraints after generating approaches, (2) multi-perspective rewriting — rewrite selected approach from security/performance/maintenance/UX perspectives, (3) completeness scoring — numeric score across 5 dimensions before generating spec. Add self-verification gate. Keep extended thinking markers.
- **Acceptance criteria**:
  - Valid YAML frontmatter with all required fields
  - Full process adapted from current SKILL.md (Steps 1-6)
  - Pipeline-specific sections removed (standalone capable)
  - Three new improvements integrated into process
  - Self-verification gate before completion
  - Reference loading instructions use Glob to find files
  - Produces: spec.md, design doc, pipeline-state.json update

### Task 4: Create investigation agent
- **Files**: `plugins/yk-dev-pipeline/agents/investigation.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the investigation agent definition. Adapt from `skills/investigation/SKILL.md` (398 lines). Remove pipeline context. Frontmatter: model opus, tools Read/Grep/Glob/WebSearch/WebFetch/Bash (NO Write/Edit — investigate only), effort high, maxTurns 50, color purple, memory project. Add reference discovery for investigation-patterns.md. Add four new improvements: (1) git bisect integration — explicit instruction to use git bisect for regressions, (2) hypothesis-evidence matrix — structured table for each hypothesis with supporting/contradicting evidence, (3) evidence threshold — minimum evidence before declaring root cause, (4) dependency changelog analysis — check changelogs when issue correlates with updates. Add self-verification gate. Note: agent writes spec.md and investigation report — needs Write tool for this. Revised tools: add Write.
- **Acceptance criteria**:
  - Valid frontmatter, tools include Write but NOT Edit
  - Full investigation process adapted (Steps 1-6)
  - Four new improvements integrated
  - Self-verification gate
  - Produces: spec.md, investigation report, pipeline-state.json update

### Task 5: Create planning agent
- **Files**: `plugins/yk-dev-pipeline/agents/planning.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the planning agent definition. Adapt from `skills/planning/SKILL.md` (363 lines). Remove pipeline context. Frontmatter: model sonnet, tools Read/Grep/Glob/Write/Edit, effort high, maxTurns 30, color green, memory project. Add reference discovery for js-ts-best-practices.md. Add four new improvements: (1) task DAG with dependency analysis — generate proper DAG visualization with critical path, (2) solvability verification — verify each task can be accomplished with available tools, (3) error propagation analysis — what happens if each task fails, (4) non-redundancy check — verify no overlapping tasks. Add self-verification gate.
- **Acceptance criteria**:
  - Valid frontmatter
  - Full planning process adapted (Steps 1-7)
  - Four new improvements integrated into task breakdown step
  - Self-verification gate
  - Produces: plan.md, archived plan, pipeline-state.json update

### Task 6: Create implementation agent
- **Files**: `plugins/yk-dev-pipeline/agents/implementation.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the implementation agent definition. Adapt from `skills/implementation/SKILL.md` (497 lines — largest skill). Remove pipeline context. Frontmatter: model opus, tools Read/Grep/Glob/Bash/Edit/Write, effort high, maxTurns 100, color orange, memory project. Add reference discovery for all implementation references (3 mandatory + 6 conditional). Add five new improvements: (1) strict incremental execution — one task at a time instead of 3-task batches, (2) scaffolding-first — generate skeleton then fill, (3) self-verification gate after each task — re-read diff, check acceptance criteria, (4) test-first enforcement — strict Red-Green-Refactor for TDD tasks, (5) context budget awareness — suggest session split after 6+ tasks. Keep extended thinking markers. Keep all existing quality gate, security gate, and self-review content.
- **Acceptance criteria**:
  - Valid frontmatter with maxTurns: 100 (highest — most work)
  - Full implementation process adapted (Steps 1-9)
  - Five new improvements integrated
  - Self-verification gate after each task AND at completion
  - Reference loading for all 9 implementation references
  - Fix mode and test-fix mode supported
  - Produces: working code, pipeline-state.json updates

### Task 7: Create code-review agent
- **Files**: `plugins/yk-dev-pipeline/agents/code-review.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the code-review agent definition. Adapt from `skills/code-review/SKILL.md` (500 lines). Remove pipeline context. Frontmatter: model opus, tools Read/Grep/Glob/Bash/Write (NO Edit — read-only review, Write for creating review.md), disallowedTools: Edit, effort high, maxTurns 50, color red, memory project. Add reference discovery for review-checklists.md + clean-code + js-ts-best-practices. Add four new improvements: (1) confidence scoring — 0.0-1.0 per finding, (2) trajectory analysis — analyze git log for change sequence quality, (3) counterfactual analysis — "is this complexity justified?", (4) static analysis first — run tsc/lint/test before AI review. The parallel review capability is orchestrator-level (not in this agent). Agent system prompt should accept a "focus areas" parameter to support being invoked with subset of review areas. Add self-verification gate.
- **Acceptance criteria**:
  - Valid frontmatter with disallowedTools: Edit
  - Full review process adapted (Steps 1-8)
  - All 18 review areas preserved
  - Four new improvements integrated
  - Supports focus-area parameter for parallel review
  - Self-verification gate
  - Produces: review.md, fix-plan.md, pipeline-state.json update

### Task 8: Create testing agent
- **Files**: `plugins/yk-dev-pipeline/agents/testing.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the testing agent definition. Adapt from `skills/testing/SKILL.md` (485 lines). Remove pipeline context. Frontmatter: model sonnet, tools Read/Grep/Glob/Bash/Edit/Write, effort high, maxTurns 80, color yellow, memory project. Add reference discovery for test-patterns.md. Add five new improvements: (1) mutation testing workflow — introduce mutations, verify tests catch them, (2) property-based testing — identify invariants, generate fast-check tests, (3) multi-metric quality assessment — coverage + mutation score + assertion density + behavior coverage, (4) regression-first generation — prioritize tests for review findings, (5) flaky test protocol — run 3x, quarantine, document. Add self-verification gate.
- **Acceptance criteria**:
  - Valid frontmatter
  - Full testing process adapted (Steps 1-8)
  - Five new improvements integrated
  - Self-verification gate
  - Produces: test files, test-report.md, fix-test-plan.md (if blocked), pipeline-state.json update

### Task 9: Create documentation agent
- **Files**: `plugins/yk-dev-pipeline/agents/documentation.md`
- **Depends on**: Task 1
- **Test mode**: no-test
- **Description**: Create the documentation agent definition. Adapt from `skills/documentation/SKILL.md` (341 lines — smallest skill). Remove pipeline context. Frontmatter: model sonnet, tools Read/Grep/Glob/Write/Edit, effort medium, maxTurns 40, color cyan, memory project. Add reference discovery for doc-templates.md. Add four new improvements: (1) ADR automation — detect architectural decisions, generate ADRs, (2) living documentation — compare spec vs actual, document deviations, (3) command verification — run/verify every command in README, (4) agent-contextual docs — generate .claude/CLAUDE.md for future AI sessions. Add self-verification gate.
- **Acceptance criteria**:
  - Valid frontmatter with effort: medium (less intensive)
  - Full documentation process adapted (Steps 1-8)
  - Four new improvements integrated
  - Self-verification gate
  - Produces: README.md, docs/*, doc-report.md, pipeline-state.json update

## Phase 3: Orchestrator Update

### Task 10: Update orchestrator SKILL.md for agent delegation
- **Files**: `plugins/yk-dev-pipeline/skills/SKILL.md`
- **Depends on**: Tasks 3-9 (all agents must exist)
- **Test mode**: no-test
- **Description**: Rewrite the orchestrator SKILL.md to delegate to agents instead of reading SKILL.md files. Keep ALL existing logic: intent detection rules (0-5), phase selection menu, continuation logic table, handoff resolution algorithm, prerequisite validation, pipeline health check, error recovery table, context management. Change the delegation mechanism: instead of "Use the Read tool to load `brainstorm/SKILL.md`", use "Invoke the brainstorm agent with the Agent tool". Add agent invocation templates with full context passing (project path, artifact locations, pipeline state). Add parallel code review section: spawn 3 code-review agents with different focus areas (security, quality, compliance), consolidate results. Add result verification: after each agent completes, verify artifacts were created and pipeline-state.json was updated. Keep backward compatibility note: if agents aren't available, fall back to reading SKILL.md files.
- **Acceptance criteria**:
  - All intent detection rules preserved (0-5)
  - All continuation logic preserved
  - All handoff resolution preserved
  - Phase selection menu preserved
  - Prerequisite validation preserved
  - Pipeline health check preserved
  - Error recovery table preserved
  - Delegation changed from Read SKILL.md → Agent tool invocation
  - Agent invocation templates with full context for all 7 agents
  - Parallel code review architecture (3 agents: security, quality, compliance)
  - Result verification after each agent completes
  - Backward compatibility fallback to skills

## Phase 4: Project Updates

### Task 11: Update marketplace.json
- **Files**: `.claude-plugin/marketplace.json`
- **Depends on**: Task 2
- **Test mode**: no-test
- **Description**: Update version to 3.0.0 and update description/tags to mention agents.
- **Acceptance criteria**:
  - Version 3.0.0
  - Description mentions agent-based pipeline
  - Tags include "agents"

### Task 12: Update root CLAUDE.md
- **Files**: `CLAUDE.md`
- **Depends on**: Tasks 1, 10 (new structure and orchestrator complete)
- **Test mode**: no-test
- **Description**: Update the project CLAUDE.md to document the new architecture. Add `agents/` directory to the Repository Structure section. Add explanation of dual-mode (skills for backward compatibility, agents for improved experience). Update Reference Library section. Add agent definition format section. Update Editing Skills section with agent editing guidance. Update Adding New Content section with agent creation guidance.
- **Acceptance criteria**:
  - Repository Structure includes `agents/` and `references/`
  - Dual-mode architecture explained
  - Agent definition format documented
  - Editing guidance covers both skills and agents

### Task 13: Update TESTING.md
- **Files**: `TESTING.md`
- **Depends on**: Tasks 3-10 (agents and orchestrator complete)
- **Test mode**: no-test
- **Description**: Add agent testing section to TESTING.md. Include: how to test agents standalone (`@brainstorm`, `claude --agent yk-dev-pipeline:brainstorm`), test prompts for each agent, quality criteria for agent output, how to test parallel code review, how to test orchestrator→agent delegation, regression markers for agent changes.
- **Acceptance criteria**:
  - Agent testing section added
  - Test prompts for each of 7 agents
  - Parallel review testing documented
  - Orchestrator delegation testing documented

### Task 14: Update example pipeline-state.json
- **Files**: `examples/pipeline-state.example.json`
- **Depends on**: Task 10
- **Test mode**: no-test
- **Description**: Update the example pipeline state to show v3.0 format with new fields: `version: "3.0"`, `orchestration_mode: "agents"`, `parallel_review: true`, `agent_results` section showing turns used per agent.
- **Acceptance criteria**:
  - New fields present in example
  - Backward compatible with v2.0 structure
  - Shows parallel review results

### Task 15: Update README.md
- **Files**: `README.md`
- **Depends on**: Tasks 1-14
- **Test mode**: no-test
- **Description**: Update README to document v3.0.0 features. Add agent architecture section. Document dual-mode (skills + agents). List improvements per agent. Update installation instructions if needed. Update version badge.
- **Acceptance criteria**:
  - Version 3.0.0 documented
  - Agent architecture explained
  - Per-agent improvements listed
  - Installation instructions correct

### Task 16: Update USAGE.md
- **Files**: `USAGE.md`
- **Depends on**: Tasks 1-14
- **Test mode**: no-test
- **Description**: Update USAGE.md to document agent invocation. Show both skill-based (existing) and agent-based (new) usage. Document standalone agent usage. Document parallel code review behavior.
- **Acceptance criteria**:
  - Both skill and agent modes documented
  - Standalone agent usage examples
  - Parallel code review explained

## Task Dependency Graph

```
Task 1 (shared references) ──┬──→ Task 3 (brainstorm agent)
                             ├──→ Task 4 (investigation agent)
                             ├──→ Task 5 (planning agent)
Task 2 (plugin.json)        ├──→ Task 6 (implementation agent)
  │                          ├──→ Task 7 (code-review agent)
  │                          ├──→ Task 8 (testing agent)
  │                          └──→ Task 9 (documentation agent)
  │                                    │
  │                          Tasks 3-9 ┤
  │                                    ▼
  │                          Task 10 (orchestrator update) ──┬──→ Task 12 (CLAUDE.md)
  │                                                         ├──→ Task 13 (TESTING.md)
  │                                                         ├──→ Task 14 (examples)
  └──────────────────────────────────→ Task 11 (marketplace)│
                                                            │
                                              Tasks 11-14 ──┤
                                                            ▼
                                              Task 15 (README.md)
                                              Task 16 (USAGE.md)
```

**Critical path**: Task 1 → Task 6 (implementation agent, largest) → Task 10 → Task 15

## Test Strategy
- **TDD tasks**: none (markdown project, no executable code)
- **Test-after tasks**: none
- **No-test tasks**: all 16 tasks — verified by manual testing per TESTING.md
- **Verification**: Each agent tested standalone via `@{agent-name}` or `claude --agent yk-dev-pipeline:{agent-name}`

## Risk Notes
- **Agent file size**: Each agent ~350-550 lines. If too large for context, can split into agent + injected skill via `skills` frontmatter field.
- **Reference discovery**: Agents need to find reference files. Using Glob pattern `**/references/{phase}/*.md` to discover paths dynamically.
- **Parallel review consolidation**: Orchestrator must handle merging 3 partial reviews. If consolidation is complex, can fall back to single comprehensive review.
- **Plugin agent limitations**: No hooks, mcpServers, or permissionMode in plugin agents. Design works within these constraints.

## Next Step
Implementation phase — work through tasks 1-16 in order.