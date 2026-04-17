# Agent-per-Phase Refactoring — Specification

## Overview
- **One-liner**: Refactor each pipeline phase into an independent Claude Code agent while keeping existing skills for backward compatibility
- **Problem**: Current skills run in the main conversation context, causing context pollution between phases, no tool scoping, no model optimization per phase, and no context isolation
- **Type**: New feature (agents) + architectural refactoring (shared references, orchestrator update)
- **Status**: New — extending existing plugin

## Target Users
- Claude Code users who install the yk-dev-pipeline plugin for JS/TS development
- Users who want to use individual agents standalone (e.g., just the code-review agent)
- Users who prefer the existing skill-based approach (backward compatible)

## User Stories
- As a developer, I want each pipeline phase to run in its own context window so that brainstorm research doesn't pollute my implementation context
- As a developer, I want the code review agent to be read-only so it cannot accidentally modify my code
- As a developer, I want parallel code review (security + quality + performance) so reviews are faster and more thorough
- As a developer, I want to use individual agents standalone (e.g., `@code-review` on any project) without running the full pipeline
- As a developer, I want existing skills to keep working so my current workflow isn't broken

## Core Features (v3.0)

### 1. Seven Independent Agents
Each pipeline phase becomes a standalone agent in `plugins/yk-dev-pipeline/agents/`:

| Agent | Model | Key Tools | Effort | MaxTurns | Color |
|-------|-------|-----------|--------|----------|-------|
| `brainstorm` | opus | Read, Grep, Glob, WebSearch, WebFetch, Bash, Edit, Write | high | 50 | blue |
| `investigation` | opus | Read, Grep, Glob, WebSearch, WebFetch, Bash | high | 50 | purple |
| `planning` | sonnet | Read, Grep, Glob, Write, Edit | high | 30 | green |
| `implementation` | opus | Read, Grep, Glob, Bash, Edit, Write | high | 100 | orange |
| `code-review` | opus | Read, Grep, Glob, Bash | high | 50 | red |
| `testing` | sonnet | Read, Grep, Glob, Bash, Edit, Write | high | 80 | yellow |
| `documentation` | sonnet | Read, Grep, Glob, Write, Edit | medium | 40 | cyan |

- Each agent has a markdown file with YAML frontmatter + system prompt
- Each agent loads its own references on demand via Read
- Each agent reads/writes artifacts to disk (spec.md, plan.md, etc.)
- Each agent updates pipeline-state.json

### 2. Orchestrator Skill Update
The router SKILL.md becomes a lightweight orchestrator that:
- Keeps all existing routing, intent detection, state management logic
- Delegates to agents via the Agent tool instead of reading SKILL.md files
- Supports parallel agent invocation for code review
- Passes full context (project path, artifact locations, pipeline state) in agent prompts

### 3. Shared Reference Library
References move to `plugins/yk-dev-pipeline/references/` (shared root):
- Agents load references by absolute path via Read tool
- Multiple agents can share the same references (e.g., clean-code-principles.md used by both implementation and code-review)
- Original skill reference paths kept via symlinks or copies for backward compatibility

### 4. Parallel Code Review
The orchestrator can spawn multiple code-review agents simultaneously with different focus areas:
- Security-focused review agent
- Quality & performance review agent
- Spec/plan compliance review agent
- Results consolidated by orchestrator into unified review.md

### 5. Per-Agent Improvements
Each agent gets research-backed improvements (see Design Reference for details):
- **Brainstorm**: Constraint-based exploration, multi-perspective rewriting, completeness scoring
- **Investigation**: Git bisect integration, hypothesis-evidence matrix, evidence thresholds
- **Planning**: Task DAG with dependency analysis, solvability verification, error propagation analysis
- **Implementation**: Strict incremental execution, scaffolding-first, self-verification gates
- **Code Review**: Parallel specialized review, trajectory analysis, counterfactual analysis, confidence scoring
- **Testing**: Mutation testing workflow, property-based testing, multi-metric quality assessment
- **Documentation**: ADR automation, living documentation, command verification

### 6. Cross-Cutting Improvements (All Agents)
- Self-verification gates before completing
- Just-in-time reference loading
- Persistent project-scoped memory
- Session split awareness
- Structured error recovery with categorized errors

## Out of Scope (Future)
- MCP server integration (not supported by plugin agents — security restriction)
- Agent hooks (not supported by plugin agents)
- Agent teams (experimental feature, not stable enough)
- Step-level state tracking from Super-YK design (can be added later within each agent)
- Non-JS/TS language support

## Tech Stack
- **Format**: Markdown files with YAML frontmatter (Claude Code agent definition format)
- **Runtime**: Claude Code CLI / IDE extensions
- **Plugin system**: Claude Code plugin marketplace
- **State**: JSON (pipeline-state.json)

## Architecture

### Directory Structure
```
plugins/yk-dev-pipeline/
├── .claude-plugin/plugin.json          # Updated manifest (version 3.0.0)
├── skills/                             # KEPT: existing skills (backward compatible)
│   ├── SKILL.md                        # UPDATED: orchestrator delegates to agents
│   ├── brainstorm/SKILL.md             # Kept as-is
│   ├── investigation/SKILL.md          # Kept as-is
│   ├── planning/SKILL.md               # Kept as-is
│   ├── implementation/SKILL.md         # Kept as-is
│   ├── code-review/SKILL.md            # Kept as-is
│   ├── testing/SKILL.md                # Kept as-is
│   └── documentation/SKILL.md          # Kept as-is
├── agents/                             # NEW: independent agents
│   ├── brainstorm.md
│   ├── investigation.md
│   ├── planning.md
│   ├── implementation.md
│   ├── code-review.md
│   ├── testing.md
│   └── documentation.md
└── references/                         # NEW: shared reference library
    ├── brainstorm/
    │   └── creativity-techniques.md
    ├── investigation/
    │   └── investigation-patterns.md
    ├── implementation/
    │   ├── clean-code-principles.md
    │   ├── js-ts-best-practices.md
    │   ├── implementation-standards.md
    │   ├── security-patterns.md
    │   ├── databases-sql.md
    │   ├── databases-nosql.md
    │   ├── databases-redis.md
    │   ├── frameworks-web.md
    │   └── frameworks-frontend.md
    ├── code-review/
    │   └── review-checklists.md
    ├── testing/
    │   └── test-patterns.md
    └── documentation/
        └── doc-templates.md
```

### Data Flow
```
User Request
     │
     ▼
Orchestrator Skill (SKILL.md)
     │  intent detection, phase selection, state management
     │
     ├──→ Agent: brainstorm ──→ spec.md + design doc
     │         or
     ├──→ Agent: investigation ──→ spec.md + investigation report
     │
     ├──→ Agent: planning ──→ plan.md
     │
     ├──→ Agent: implementation ──→ working code
     │
     ├──→ Parallel agents: code-review ──→ review.md + fix-plan.md
     │    (security + quality + compliance)
     │
     ├──→ Agent: testing ──→ test files + test-report.md
     │
     └──→ Agent: documentation ──→ README + docs/
```

## Error Handling Strategy
- Each agent handles its own errors within its context
- If an agent fails, orchestrator receives error in agent result
- Orchestrator presents recovery options to user
- Fix loops (code-review blocked → implementation → re-review) handled by orchestrator
- Each agent writes progress to pipeline-state.json for session recovery

## Security Considerations
### Constraints
- Plugin agents cannot define hooks, mcpServers, or permissionMode (silently ignored)
- Code review agent is read-only (Read, Grep, Glob, Bash only) — cannot modify code
- Investigation agent has no Write/Edit — investigates only, doesn't modify

## Deployment & Environment
- Plugin installed via: `/plugin marketplace add` + `/plugin install`
- Agents auto-discovered from `agents/` directory
- Compatible with Claude Code CLI, desktop app, IDE extensions
- Backward compatible: existing skill-based workflow still works

## Constraints
- Subagents cannot spawn other subagents (flat delegation only)
- Plugin agents don't support hooks/mcpServers/permissionMode
- Agent prompts must be self-contained (no conversation history from parent)
- Agent results return as single message to orchestrator

## Open Questions
- Should each agent's memory scope be `project` or `local`?
- Should we add a `verification` agent that runs after each phase to validate outputs?
- Optimal batch size for parallel code review (2 vs 3 specialized agents)?

## Design Reference
See [full design document](docs/plans/2026-04-16-agent-refactoring-design.md)