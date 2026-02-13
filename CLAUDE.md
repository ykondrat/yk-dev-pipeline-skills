# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **not a traditional software project** — it contains no executable code, no dependencies, and no build system. It is a collection of Claude AI skills (structured markdown instruction files) that guide Claude through a 6-phase JS/TS development lifecycle:

```
Brainstorm → Planning → Implementation → Code Review → Testing → Documentation
```

Each phase has a dedicated SKILL.md with detailed processes, and produces artifacts (spec.md, plan.md, etc.) consumed by the next phase.

## Repository Structure

- `skills/SKILL.md` — **Router/entry point**. Defines the pipeline, handles phase navigation, manages `pipeline-state.json`.
- `skills/{phase}/SKILL.md` — Phase-specific instructions (brainstorm, planning, implementation, code-review, testing, documentation).
- `skills/implementation/references/` — Deep reference materials (~5,500 lines): JS/TS best practices, SQL/NoSQL/Redis database patterns.
- `skills/code-review/references/review-checklists.md` — 18-area review checklists with severity guidance.
- `examples/pipeline-state.example.json` — Example state tracking file.

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
- **Artifact chaining** — each phase produces files the next phase reads (spec.md → plan.md → code → review.md → test-report.md → docs). Design docs and plans are also archived in `docs/plans/` with date prefixes (e.g., `YYYY-MM-DD-{topic}-design.md`, `YYYY-MM-DD-{topic}-plan.md`)
- **Fix plan files** — code review produces `fix-plan.md`, testing produces `fix-test-plan.md` when blocked. The implementation skill reads all three: `plan.md`, `fix-plan.md`, `fix-test-plan.md`
- **Failure recovery loops** — code review can block and loop back to implementation; testing can block (status `"blocked"`) and loop back to implementation, or skip failing tests (marked `.todo`) — user decides via `AskUserQuestion`
- **Pipeline state** tracked in `pipeline-state.json` at the project root (statuses: `pending`, `in-progress`, `completed`, `blocked`)
- **Review severity levels**: 🔴 Critical, 🟡 Major, 🔵 Minor, ⚪ Nitpick, 🟣 Debatable

## Editing Skills

When modifying skills, preserve:
- The YAML frontmatter format
- The progressive loading pattern (skills reference other files, not inline everything)
- The artifact chaining between phases
- Severity emoji conventions in review-related files
- The "suggest next phase, wait for user confirmation" interaction pattern

## Adding New Content

- **New reference materials** go in the appropriate `references/` directory, following the structure of existing references (rationale + examples + severity tables)
- **New review checklist areas** go in `skills/code-review/references/review-checklists.md` with standard severity emoji levels
- **Parent SKILL.md must be updated** to reference any new files
