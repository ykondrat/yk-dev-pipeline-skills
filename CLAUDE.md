# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **not a traditional software project** — it contains no executable code, no dependencies, and no build system. It is a collection of Claude AI **agents and skills** (structured markdown instruction files) that guide Claude through a 6-phase JS/TS development lifecycle:

```
Brainstorm/Investigation → Planning → Implementation → Code Review → Testing → Documentation
```

**v3.0 Architecture:** Each phase is an independent **agent** running in its own context window with scoped tools and model selection. The orchestrator skill manages routing, state, and handoffs. Original skills are kept for backward compatibility.

## Repository Structure

This repo is structured as a **Claude Code plugin marketplace** with one plugin:

- `.claude-plugin/marketplace.json` — **Marketplace catalog** (`owner`, `plugins` with `source` paths). Validated by Claude Code on `/plugin marketplace add`.
- `plugins/yk-dev-pipeline/` — **Plugin directory** containing the dev pipeline.
  - `plugins/yk-dev-pipeline/.claude-plugin/plugin.json` — Plugin manifest (`name`, `version`, `description`, `author`). Agents and skills are auto-discovered from their directories.
  - **`plugins/yk-dev-pipeline/agents/`** — **Independent phase agents** (v3.0). Each agent runs in its own context window with scoped tools:
    - `brainstorm.md` — Opus, full tools + web research, blue
    - `investigation.md` — Opus, no Edit (investigate-only), purple
    - `planning.md` — Sonnet, Read/Write/Edit, green
    - `implementation.md` — Opus, full tools, maxTurns 100, orange
    - `code-review.md` — Opus, no Edit (read-only review), red
    - `testing.md` — Sonnet, full tools, maxTurns 80, yellow
    - `documentation.md` — Sonnet, Read/Write/Edit, cyan
  - **`plugins/yk-dev-pipeline/references/`** — **Shared reference library** (v3.0). Used by agents via Read tool:
    - `references/brainstorm/creativity-techniques.md`
    - `references/investigation/investigation-patterns.md`
    - `references/implementation/` — 9 files (clean-code, js-ts, standards, security, databases, frameworks)
    - `references/code-review/review-checklists.md`
    - `references/testing/test-patterns.md`
    - `references/documentation/doc-templates.md`
  - `plugins/yk-dev-pipeline/skills/SKILL.md` — **Orchestrator** (v3.0). Manages intent detection, phase selection, agent invocation, parallel code review, state management, handoff resolution. Falls back to skill-based approach if agents unavailable.
  - `plugins/yk-dev-pipeline/skills/{phase}/SKILL.md` — **Phase skills** (v2.0, kept for backward compatibility). Original skill-based instructions for each phase.
  - `plugins/yk-dev-pipeline/skills/{phase}/references/` — **Skill reference files** (kept for backward compatibility with skill-based approach).
- `examples/` — Example artifacts: `pipeline-state.example.json`, `pipeline-state-investigation.example.json`, `plan.example.md`, `review.example.md`, `fix-plan.example.md`, `fix-test-plan.example.md`, `test-report.example.md`.
- `TESTING.md` — **Testing guide**: test prompts per agent/skill, output quality criteria, regression markers.

## No Build/Test/Lint Commands

There are no package.json, build tools, or test runners. "Testing" an agent means running
it in a real Claude conversation and verifying Claude follows the instructions correctly.
See `TESTING.md` for the full testing guide.

## File Formats

**Agent files** (`agents/*.md`) use:
1. YAML frontmatter (`name`, `description`, `model`, `tools`, `effort`, `maxTurns`, `memory`, `color`)
2. Markdown body as the agent's system prompt
3. Process steps, reference loading instructions, self-verification gates

**Skill files** (`skills/*/SKILL.md`) use:
1. YAML frontmatter (`name`, `description`)
2. Markdown with clear hierarchical sections
3. Code examples in fenced blocks with language tags
4. Tables for severity guidance and pattern catalogs

## Key Conventions

- **Agents run in isolated context** — each phase agent has its own context window, scoped tools, and model
- **Parallel code review** — orchestrator invokes 3 code-review agents simultaneously (security, quality, compliance)
- **Shared reference library** — `references/` directory at plugin root, loaded on demand by agents via Read/Glob
- **Artifact chaining** — each phase produces files the next reads (spec.md → plan.md → code → review.md → test-report.md → docs)
- **Fix plan files** — code review produces `fix-plan.md`, testing produces `fix-test-plan.md`
- **Failure recovery loops** — code review/testing can block → implementation → re-review/re-test
- **Pipeline state** tracked in `pipeline-state.json` (v3.0 adds `version`, `orchestration_mode`, `agent_results`)
- **Review severity levels**: 🔴 Critical, 🟡 Major, 🔵 Minor, ⚪ Nitpick
- **Self-verification gates** — every agent verifies its own output before completing
- **Per-agent improvements** — brainstorm: constraint exploration, multi-perspective; investigation: git bisect, hypothesis matrix; planning: task DAG, solvability; implementation: incremental execution, scaffolding-first; code-review: confidence scoring, trajectory analysis; testing: mutation testing, property-based; documentation: ADR automation, living docs

## Editing Agents

When modifying agents, preserve:
- The YAML frontmatter with all required fields (name, description, model, tools, effort, maxTurns, memory, color)
- Tool scoping (code-review has no Edit, investigation has no Edit)
- The self-verification gate at the end
- Reference loading instructions (Glob pattern + Read)
- The artifact production and pipeline-state.json updates

## Editing Skills (Backward Compatibility)

When modifying skills, preserve:
- The YAML frontmatter format
- The progressive loading pattern
- The artifact chaining between phases
- Severity emoji conventions
- The "suggest next phase, wait for confirmation" interaction pattern

**After modifying an agent or skill, consult `TESTING.md`.**

## Adding New Content

- **New agents** go in `plugins/yk-dev-pipeline/agents/` with full frontmatter
- **New reference materials** go in `plugins/yk-dev-pipeline/references/{phase}/`
- **New review checklist areas** go in `references/code-review/review-checklists.md`
- **Orchestrator SKILL.md must be updated** to reference any new agents or phases
- **Keep skill references in sync** — if adding references to `references/`, also add to `skills/{phase}/references/` for backward compatibility