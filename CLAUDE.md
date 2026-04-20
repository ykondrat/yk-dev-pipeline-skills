# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **not a traditional software project** — it contains no executable code, no dependencies, and no build system. It is a collection of Claude AI **agents and skills** (structured markdown instruction files) that guide Claude through a 6-phase JS/TS development lifecycle:

```
Brainstorm/Investigation → Planning → Implementation → Code Review → Testing → Documentation
```

**v3.0 Architecture:** Each phase is an independent **agent** running in its own context window with scoped tools and model selection. The orchestrator skill manages routing, state, and handoffs. Original skills are kept for backward compatibility.

## Repository Structure

This repo is a **flat single-plugin layout**: the repo root *is* the plugin. It's still installable via `/plugin marketplace add` because both `.claude-plugin/marketplace.json` and `.claude-plugin/plugin.json` sit at the root, and the marketplace entry points at `"."` as its source.

- `.claude-plugin/marketplace.json` — **Marketplace catalog** (`owner`, `plugins` with `source: "."`). Validated by Claude Code on `/plugin marketplace add`.
- `.claude-plugin/plugin.json` — **Plugin manifest** (`name`, `version`, `description`, `author`). Agents, skills, and commands are auto-discovered from their sibling directories.
- **`agents/`** — **Independent phase agents** (v3.0). Each agent runs in its own context window with scoped tools:
  - `brainstorm.md` — Opus, full tools + web research, blue
  - `investigation.md` — Opus, no Edit (investigate-only), purple
  - `planning.md` — Sonnet, Read/Write/Edit, green
  - `implementation.md` — Opus, full tools, maxTurns 100, orange
  - `code-review.md` — Opus, no Edit (read-only review), red
  - `testing.md` — Sonnet, full tools, maxTurns 80, yellow
  - `documentation.md` — Sonnet, Read/Write/Edit, cyan
  - `FRONTMATTER.md` — single source of truth for which frontmatter fields are authoritative vs advisory
- **`references/`** — **Shared reference library** (v3.0). Used by both agents and skills via Read tool. All reference content lives here, deduplicated:
  - `references/brainstorm/creativity-techniques.md`
  - `references/investigation/investigation-patterns.md`
  - `references/implementation/` — 9 files (clean-code, js-ts, standards, security, databases, frameworks)
  - `references/code-review/review-checklists.md`
  - `references/testing/test-patterns.md`
  - `references/documentation/doc-templates.md`
- **`commands/`** — **Slash commands** — `/pipeline-start`, `/pipeline-continue`, `/pipeline-status`, `/pipeline-review`, `/pipeline-reset`. Each is a markdown file with YAML frontmatter defining the command.
- **`pipeline-state.schema.json`** — **Canonical JSON Schema (draft-07)** for `pipeline-state.json`. Every phase (agent or skill) validates its state updates against this schema. Documents the verdict → status mapping.
- **`skills/`** — Skill-based fallbacks (v2.0) plus the two orchestrators:
  - `skills/SKILL.md` — **Main router + skill-orchestrator** (`yk-dev-pipeline`). Auto-detects agent availability: hands off to the agent-orchestrator when agents are present, runs the pipeline in-place (v2.0 skill mode) when they are not.
  - `skills/agent-orchestrator/SKILL.md` — **Agent-orchestrator** (`yk-agent-orchestrator`, v3.0). Invokes phase agents via the Task tool; coordinates parallel code review; falls back to the skill-orchestrator if agents are unavailable.
  - `skills/{phase}/SKILL.md` — **Phase skills** (v2.0). Carry a "legacy — see `../../agents/`" banner. Kept for Claude.ai Projects uploads and skill-only installs where agents are unavailable. All reference paths point at the shared `../references/` tree.
- `examples/` — Example artifacts: `pipeline-state.example.json`, `pipeline-state-investigation.example.json`, `plan.example.md`, `review.example.md`, `fix-plan.example.md`, `fix-test-plan.example.md`, `test-report.example.md`.
- `docs/` — Design and review docs (e.g., `docs/review-2026-04-17.md`).
- `TESTING.md` — **Testing guide**: test prompts per agent/skill, output quality criteria, regression markers.

## No Build/Test/Lint Commands

There are no package.json, build tools, or test runners. "Testing" an agent means running
it in a real Claude conversation and verifying Claude follows the instructions correctly.
See `TESTING.md` for the full testing guide.

## File Formats

**Agent files** (`agents/*.md`) use:
1. YAML frontmatter (`name`, `description`, `model`, `tools`, `disallowedTools`, `color`, plus advisory `effort`, `maxTurns`, `memory`). See `agents/FRONTMATTER.md` for which are authoritative vs advisory.
2. Markdown body as the agent's system prompt
3. Process steps, reference loading instructions, operational guardrails (git safety + idempotency), self-verification gates

**Skill files** (`skills/*/SKILL.md`) use:
1. YAML frontmatter (`name`, `description`, optional `metadata`)
2. Markdown with clear hierarchical sections
3. Code examples in fenced blocks with language tags
4. Tables for severity guidance and pattern catalogs

**Slash command files** (`commands/*.md`) use:
1. YAML frontmatter (`description`, `argument-hint`, `allowed-tools`)
2. Markdown body used as the prompt body when the command is invoked
3. `$ARGUMENTS` placeholder for user-supplied input

**State schema** (`pipeline-state.schema.json`) is JSON Schema draft-07 — every agent and skill must produce state that validates against it.

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
- The YAML frontmatter — authoritative fields (`name`, `description`, `model`, `tools`, `disallowedTools`, `color`) must be present and correct
- Tool scoping (code-review has no Edit, investigation has no Edit)
- Operational guardrails section (git safety + resume/idempotency)
- The self-verification gate at the end
- Reference loading instructions (Glob pattern + Read)
- Artifact production and `pipeline-state.json` updates that validate against `pipeline-state.schema.json`

## Editing Skills

When modifying skills, preserve:
- The YAML frontmatter format
- The progressive loading pattern
- The artifact chaining between phases
- Severity emoji conventions
- The "suggest next phase, wait for confirmation" interaction pattern
- Reference paths pointing at the shared `../references/{phase}/` tree
- The "legacy — see `../../agents/`" banner on phase skills (`brainstorm`, `investigation`, `planning`, `implementation`, `code-review`, `testing`, `documentation`)

**After modifying an agent or skill, consult `TESTING.md`.**

## Editing the Two Orchestrators

- `skills/SKILL.md` (`yk-dev-pipeline`) is the main router — it picks the agent path or the skill path based on tool/agent availability. Keep the path-selection preamble at the top intact.
- `skills/agent-orchestrator/SKILL.md` (`yk-agent-orchestrator`) is the agent-first orchestrator. Keep it focused on Task-tool invocation, parallel review, and schema-validated state updates.

## Editing Slash Commands

- Slash commands live in `commands/*.md`. Filename (minus `.md`) is the command name.
- Keep the `allowed-tools` list conservative — do not grant tools the command doesn't actually need.
- `$ARGUMENTS` interpolates user input; treat it as untrusted.
- Destructive commands (e.g., `/pipeline-reset`) must confirm via `AskUserQuestion` unless the user supplies the literal `confirm` token.

## Adding New Content

- **New agents** go in `agents/` with full frontmatter (see `agents/FRONTMATTER.md`)
- **New reference materials** go in `references/{phase}/`
- **New review checklist areas** go in `references/code-review/review-checklists.md`
- **Both orchestrators must be updated** when you add/remove phases or change agent names
- **The JSON schema must be updated** if you add new per-phase state fields — every agent's state update is validated against it