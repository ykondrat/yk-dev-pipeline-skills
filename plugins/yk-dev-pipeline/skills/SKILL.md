---
name: yk-dev-pipeline
description: >
  Full JS/TS development pipeline with 6 phases: brainstorm, planning, implementation,
  code-review, testing, and documentation. Use this for any new JS/TS project or feature.
  Each phase produces artifacts consumed by the next. Semi-automatic — suggests the next
  step, user confirms. Triggers on: "start a new project", "build me a...", "let's develop...",
  "new feature", "start pipeline", or any request to build a JS/TS application or feature
  from scratch.
metadata:
  recommended_model: sonnet
---

# JS/TS Development Pipeline

You have access to a 6-phase development pipeline for building production-quality
JavaScript/TypeScript projects. Each phase has a dedicated skill with detailed instructions
and reference materials.

## Pipeline Overview

```
[1. Brainstorm] → [2. Planning] → [3. Implementation] → [4. Code Review] → [5. Testing] → [6. Documentation]
       ↓                ↓                 ↓                    ↓                  ↓                ↓
    spec.md          plan.md         working code        review.md + fix-plan   test-report.md    README + docs/
                                                              ↓ (if blocked)
                                                         back to Implementation
```

## How to Use

### Starting a New Project

When the user wants to build something, start at Phase 1 (Brainstorm). Read the
brainstorm skill and follow its process:

```
Read brainstorm/SKILL.md
```

### Continuing the Pipeline

After each phase completes, suggest the next phase. The user confirms before proceeding.
When moving to the next phase, read that phase's skill:

```
Phase 1 complete → "Ready for planning? I'll break this into tasks."
  Read planning/SKILL.md

Phase 2 complete → "Ready to start implementing? I'll work through the tasks."
  Read implementation/SKILL.md
  Read implementation/references/js-ts-best-practices.md
  Read implementation/references/databases-*.md (if applicable)

Phase 3 complete → "Ready for code review? I'll review everything."
  Read code-review/SKILL.md
  Read code-review/references/review-checklists.md

Phase 4 complete (approved) → "Ready for testing? I'll write comprehensive tests."
  Read testing/SKILL.md

Phase 4 complete (blocked) → "Issues found. I'll fix them and re-review."
  Read implementation/SKILL.md (to execute fix-plan.md)
  Then re-read code-review/SKILL.md (diff review)

Phase 5 complete → "Ready for documentation? I'll generate all project docs."
  Read documentation/SKILL.md

Phase 6 complete → "Pipeline complete! Project is built, reviewed, tested, and documented."
```

### Jumping to a Specific Phase

The user can skip phases or jump to any phase directly:

- "Just implement this" → Start at Phase 3 (but note missing spec/plan)
- "Review my code" → Start at Phase 4
- "Write tests for this" → Start at Phase 5
- "Generate docs" → Start at Phase 6
- "Continue from where we left off" → Check pipeline-state.json

### Pipeline State

All phases track progress in `pipeline-state.json`. Check this file to know where
the pipeline is and what's been completed:

```json
{
  "project_name": "{project name}",
  "created_at": "{ISO}",
  "current_phase": "implementation",
  "phases": {
    "brainstorm": { "status": "completed", "outputs": ["spec.md", "docs/plans/..."] },
    "planning": { "status": "completed", "outputs": ["plan.md"] },
    "implementation": { "status": "in-progress", "completed_tasks": [1, 2, 3] },
    "code-review": { "status": "pending" },
    "testing": { "status": "pending" },
    "documentation": { "status": "pending" }
  }
}
```

### State Validation

When reading `pipeline-state.json`, validate before proceeding:

- **Missing artifacts:** If a phase says "completed" but its outputs don't exist on disk,
  warn the user and suggest re-running that phase.
- **Blocked without fix plan:** If code-review or testing is "blocked" but `fix-plan.md` /
  `fix-test-plan.md` is missing, flag the inconsistency.
- **Stale state:** If `current_phase` doesn't match the most recently completed phase + 1,
  ask the user which phase they want to continue from.
- **Out-of-order completion:** If a later phase is completed but an earlier one isn't,
  note this and ask if it's intentional (e.g., user jumped to a phase directly).

## Phase Summary

### Phase 1: Brainstorm
**Skill:** `brainstorm/SKILL.md`
**Purpose:** Deep-dive into requirements through conversational questioning.
**Output:** `spec.md` (concise summary) + detailed design doc
**Key:** One question at a time, checks existing project context first, proposes approaches with trade-offs, YAGNI ruthlessly.

### Phase 2: Planning
**Skill:** `planning/SKILL.md`
**Purpose:** Break the spec into an ordered task list with dependencies and acceptance criteria.
**Output:** `plan.md` with task dependency graph
**Key:** Every task has files to create, dependencies, acceptance criteria. Tasks grouped into phases.

### Phase 3: Implementation
**Skill:** `implementation/SKILL.md`
**References:**
- `implementation/references/js-ts-best-practices.md`
- `implementation/references/databases-sql.md`
- `implementation/references/databases-nosql.md`
- `implementation/references/databases-redis.md`

**Purpose:** Execute the plan — write code, install deps, verify acceptance criteria.
**Output:** Working code, passing builds
**Key:** Critical plan review before coding, tech stack detection, batch execution (3 tasks per batch), quality gate after every batch (types + build + lint + tests), hard STOP on blockers.

### Phase 4: Code Review
**Skill:** `code-review/SKILL.md`
**Reference:** `code-review/references/review-checklists.md`
**Purpose:** Strict review across 18 areas — find every issue.
**Output:** `review.md` + `fix-plan.md`
**Key:** 4-level verdict (Block / Request Changes / Approve+suggestions / Approve). If blocked, loops back to implementation. Diff mode for re-reviews.

### Phase 5: Testing
**Skill:** `testing/SKILL.md`
**Purpose:** Write comprehensive tests — unit, integration, e2e, edge cases, benchmarks.
**Output:** Test files + `test-report.md`
**Key:** Auto-detect test runner (default Vitest), >80% coverage target, generate-run-fix loop, production bugs documented.

### Phase 6: Documentation
**Skill:** `documentation/SKILL.md`
**Purpose:** Generate all project documentation from actual code.
**Output:** README.md, API docs, architecture, contributing, changelog, deployment guide
**Key:** Every statement verified against code. Adapts format to project type. No aspirational docs.

## Handling User Requests

| User says | Action |
|---|---|
| "Build me a REST API for..." | Start Phase 1 (Brainstorm) |
| "I have a spec, let's plan" | Start Phase 2 (Planning), read their spec |
| "Here's my plan, start coding" | Start Phase 3 (Implementation), read their plan |
| "Review this code" | Start Phase 4 (Code Review) |
| "Write tests" | Start Phase 5 (Testing) |
| "Generate docs" | Start Phase 6 (Documentation) |
| "Fix the review issues" / "Address the feedback" | Start Phase 3 (Implementation) in fix mode — read `fix-plan.md` |
| "Fix the failing tests" / "Fix test failures" | Start Phase 3 (Implementation) in test-fix mode — read `fix-test-plan.md` |
| "Continue" / "Next step" | Check pipeline-state.json, proceed to next phase |
| "What's the status?" | Read pipeline-state.json, summarize progress |
| "Start over" | Reset pipeline-state.json, start Phase 1 |
| "Skip to testing" | Jump to Phase 5, note skipped phases |

## Important Rules

1. **Always read the skill file before starting a phase.** The skills contain detailed
   instructions, checklists, and examples that are essential for quality output.
2. **Read reference files when the skill tells you to.** Best practices, checklists, and
   database references exist for a reason.
3. **Follow the skill's process step by step.** Don't skip steps.
4. **Update pipeline-state.json** after each phase and significant milestone.
5. **Suggest the next phase** but wait for user confirmation before proceeding.
6. **If the user provides their own spec/plan/code**, adapt — don't force them through
   earlier phases they've already done.
