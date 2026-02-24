# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **not a traditional software project** — it contains no executable code, no dependencies, and no build system. It is a collection of Claude AI skills (structured markdown instruction files) that guide Claude through a 6-phase JS/TS development lifecycle:

```
Brainstorm/Investigation → Planning → Implementation → Code Review → Testing → Documentation
```

Each phase has a dedicated SKILL.md with detailed processes, and produces artifacts (spec.md, plan.md, etc.) consumed by the next phase.

## Repository Structure

This repo is structured as a **Claude Code plugin marketplace** with one plugin:

- `.claude-plugin/marketplace.json` — **Marketplace catalog** (`owner`, `plugins` with `source` paths). Validated by Claude Code on `/plugin marketplace add`.
- `plugins/yk-dev-pipeline/` — **Plugin directory** containing the dev pipeline.
  - `plugins/yk-dev-pipeline/.claude-plugin/plugin.json` — Minimal plugin manifest (`name`, `version`, `description`, `author`). Skills are auto-discovered from the `skills/` directory.
  - `plugins/yk-dev-pipeline/skills/SKILL.md` — **Router/entry point**. Single entry point for all general requests. Contains Intent Detection Rules (5 rules for classifying user requests) and Continuation Logic (state-to-next-action routing table). Manages `pipeline-state.json`.
  - `plugins/yk-dev-pipeline/skills/{phase}/SKILL.md` — Phase-specific instructions (brainstorm, investigation, planning, implementation, code-review, testing, documentation).
  - `plugins/yk-dev-pipeline/skills/investigation/references/` — Investigation patterns: debugging strategies, root cause categories, refactoring analysis, performance investigation, evidence documentation.
  - `plugins/yk-dev-pipeline/skills/implementation/references/` — Deep reference materials: JS/TS best practices, SQL/NoSQL/Redis database patterns, web framework patterns (Express/Fastify/Hono/Next.js), frontend framework patterns (React/Next.js/Vue 3).
  - `plugins/yk-dev-pipeline/skills/code-review/references/review-checklists.md` — All 18 review area checklists with severity guidance.
  - `plugins/yk-dev-pipeline/skills/testing/references/test-patterns.md` — Vitest reference, test writing patterns, factories, mocking, assertions, frontend testing, anti-patterns.
  - `plugins/yk-dev-pipeline/skills/documentation/references/doc-templates.md` — Templates for all 7 doc types (README, API, architecture, etc.).
- `examples/` — Example artifacts: `pipeline-state.example.json`, `pipeline-state-investigation.example.json`, `plan.example.md`, `review.example.md`, `fix-plan.example.md`, `fix-test-plan.example.md`, `test-report.example.md`.

## No Build/Test/Lint Commands

There are no package.json, build tools, or test runners. "Testing" a skill means uploading it to a Claude conversation, giving a realistic task, and verifying Claude follows the instructions correctly.

## Skill File Format

Every SKILL.md uses:
1. YAML frontmatter (`name`, `description`)
2. Markdown with clear hierarchical sections
3. Code examples in fenced blocks with language tags
4. Tables for severity guidance and pattern catalogs

## Key Conventions

- **SKILL.md files stay under ~500 lines** — deep content goes in `references/` subdirectories
- **Reference files are loaded on demand** — skills tell Claude when to read them
- **Artifact chaining** — each phase produces files the next phase reads (spec.md → plan.md → code → review.md → test-report.md → docs). Design docs, investigation reports, and plans are also archived in `docs/plans/` with date prefixes (e.g., `YYYY-MM-DD-{topic}-design.md`, `YYYY-MM-DD-{topic}-investigation.md`, `YYYY-MM-DD-{topic}-plan.md`)
- **Fix plan files** — code review produces `fix-plan.md`, testing produces `fix-test-plan.md` when blocked. The implementation skill reads all three: `plan.md`, `fix-plan.md`, `fix-test-plan.md`
- **Failure recovery loops** — code review can block and loop back to implementation; testing can block (status `"blocked"`) and loop back to implementation, or skip failing tests (marked `.todo`) — user decides via `AskUserQuestion`
- **Pipeline state** tracked in `pipeline-state.json` at the project root (statuses: `pending`, `in-progress`, `completed`, `blocked`)
- **Review severity levels**: 🔴 Critical, 🟡 Major, 🔵 Minor, ⚪ Nitpick. Debatable findings are tagged `[DEBATABLE]` (not a separate severity level)
- **Router-first routing** — The Router SKILL.md is the single entry point for all general requests. Phases 1a-3 have narrowed triggers (explicit phase-name mentions only); Phases 4-6 retain broad standalone triggers. "next step" / "continue" are exclusively Router-handled via the Continuation Logic table.

## Editing Skills

When modifying skills, preserve:
- The YAML frontmatter format
- The progressive loading pattern (skills reference other files, not inline everything)
- The artifact chaining between phases
- Severity emoji conventions in review-related files
- The "suggest next phase, wait for user confirmation" interaction pattern

## Adding New Content

- **New reference materials** go in the appropriate `references/` directory, following the structure of existing references (rationale + examples + severity tables)
- **New review checklist areas** go in `plugins/yk-dev-pipeline/skills/code-review/references/review-checklists.md` with standard severity emoji levels
- **Parent SKILL.md must be updated** to reference any new files
