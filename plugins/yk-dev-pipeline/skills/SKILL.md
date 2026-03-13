---
name: yk-dev-pipeline
description: >
  Full JS/TS development pipeline with 6 phases: brainstorm (or investigation), planning,
  implementation, code-review, testing, and documentation. Use this for any JS/TS project,
  feature, bug fix, or refactoring. Each phase produces artifacts consumed by the next.
  Semi-automatic — suggests the next step, user confirms. Triggers on: "start a new project",
  "build me a...", "let's develop...", "new feature", "start pipeline", "fix this bug",
  "debug this", "investigate", "refactor", "performance issue", "tech debt",
  "I want to build", "I need a...", "help me create", "let's make",
  "new app", "new service", "new API", "add a feature",
  "something is broken", "this doesn't work", "not working", "it's broken",
  "this is slow", "this keeps failing", "there's an error",
  "help me fix", "figure out why", "what's wrong with",
  "clean up this code", "improve this", "modernize",
  "start developing", "let's get started", "I have an idea for",
  "set up a project", "scaffold", "prototype",
  "memory leak", "crashing", "regression", "flaky",
  or any request to build, fix, or improve a JS/TS application.
metadata:
  recommended_model: sonnet
---

# JS/TS Development Pipeline

You have access to a 6-phase development pipeline for building production-quality
JavaScript/TypeScript projects. Each phase has a dedicated skill with detailed instructions
and reference materials.

## Pipeline Overview

```
[1a. Brainstorm] ─┐
                  ├→ [2. Planning] → [3. Implementation] → [4. Code Review] → [5. Testing] → [6. Documentation]
[1b. Investigation]┘       ↓                 ↓                    ↓                  ↓                ↓
                        plan.md         working code        review.md + fix-plan   test-report.md    README + docs/
       ↓                                                         ↓ (if blocked)
    spec.md                                                 back to Implementation
```

Phase 1 has two alternatives:
- **Brainstorm** — for new features, new projects, adding functionality
- **Investigation** — for bug fixes, refactoring, performance issues, tech debt

## How to Use

### Phase Selection

When starting a new pipeline (no `pipeline-state.json` exists, or user says "start over"),
present the phase selection menu before routing to Phase 1. Use the `AskUserQuestion` tool:

> Which phases do you want to include in this pipeline run?
>
> Available phases:
>   1a. Brainstorm  (or 1b. Investigation)
>   2.  Planning
>   3.  Implementation
>   4.  Code Review
>   5.  Testing
>   6.  Documentation
>
> Presets:
>   [A] Full pipeline (all phases) — recommended
>   [B] Quick build (brainstorm/investigation → planning → implementation)
>   [C] Review & polish (code review → testing → documentation)
>   [D] Custom — pick individual phases

If the user picks [D], follow up with: "Which phases? (e.g., 'brainstorm, planning,
implementation, testing')". Accept phase names in any format and normalize to canonical names:

| Input | Canonical name |
|-------|---------------|
| 1a, brainstorm, bs | brainstorm |
| 1b, investigation, invest, inv | investigation |
| 2, planning, plan | planning |
| 3, implementation, impl, implement | implementation |
| 4, code-review, review, cr | code-review |
| 5, testing, test, tests | testing |
| 6, documentation, docs, doc | documentation |

Store the selection in `pipeline-state.json` as `selected_phases` (see Pipeline State below).
Then proceed to the first selected phase (brainstorm or investigation).

**Skip the menu when:**
- User's request already implies phase selection (see Rule 0 in Intent Detection)
- User explicitly names a single phase ("brainstorm this") — set `selected_phases` to all phases
- User says "full pipeline" or doesn't mention any phase selection — set `selected_phases` to all phases

### Starting Phase 1

Detect the user's intent and route to the correct Phase 1 skill.
Use the Read tool to load the appropriate SKILL.md:

**New feature / new project / adding functionality → Brainstorm:**
Use the Read tool to load `brainstorm/SKILL.md`

**Bug fix / refactoring / performance / tech debt / debugging → Investigation:**
Use the Read tool to load `investigation/SKILL.md`

Both skills produce `spec.md` as their output, so Phase 2 (Planning) works the same
regardless of which Phase 1 was used.

### Continuing the Pipeline

After each phase completes, suggest the next phase. The user confirms before proceeding.
When moving to the next phase, use the Read tool to load that phase's SKILL.md (the skill
will instruct you to load its own references — follow those instructions):

**Phase 1 complete** (brainstorm or investigation) → "Ready for planning? I'll break this into tasks."
→ Use the Read tool to load `planning/SKILL.md`

**Phase 2 complete** → "Ready to start implementing? I'll work through the tasks."
→ Use the Read tool to load `implementation/SKILL.md` (it will tell you to load its references)

**Phase 3 complete** → "Ready for code review? I'll review everything."
→ Use the Read tool to load `code-review/SKILL.md` (it will tell you to load its references)

**Phase 4 complete (approved)** → "Ready for testing? I'll write comprehensive tests."
→ Use the Read tool to load `testing/SKILL.md` (it will tell you to load its references)

**Phase 4 complete (blocked)** → "Issues found. I'll fix them and re-review."
→ Use the Read tool to load `implementation/SKILL.md` (to execute fix-plan.md)
→ Then re-load `code-review/SKILL.md` (diff review)

**Phase 5 complete** → "Ready for documentation? I'll generate all project docs."
→ Use the Read tool to load `documentation/SKILL.md` (it will tell you to load its references)

**Phase 6 complete** → "Pipeline complete! Project is built, reviewed, tested, and documented."

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
  "selected_phases": ["brainstorm", "planning", "implementation", "code-review", "testing", "documentation"],
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

**`selected_phases`:** Array of phases the user chose to run. If absent, all phases are
selected (backward compatible). The ordered phase sequence is always:
brainstorm/investigation → planning → implementation → code-review → testing → documentation.
Phases not in `selected_phases` are skipped during continuation and handoff resolution.

**Note:** The first phase key is either `brainstorm` OR `investigation` — they are
alternatives, never both. If investigation was Phase 1, the state will have an
`investigation` key instead of `brainstorm`.

### Prerequisite Validation

**Before routing to any phase**, validate prerequisites. If validation fails, present
clear recovery options — don't silently proceed with missing inputs.

| Phase | Required Files | Required Status | If Missing |
|-------|---------------|----------------|------------|
| Planning | `spec.md`, design doc or investigation report | brainstorm/investigation: completed | "Spec is missing. Re-run brainstorm/investigation, or provide a spec." |
| Implementation | `plan.md` | planning: completed | "Plan is missing. Run planning first, or provide a plan." |
| Implementation (fix) | `fix-plan.md` or `fix-test-plan.md` | code-review: blocked OR testing: blocked | "Fix plan is missing but phase is blocked. Re-run code review/testing." |
| Code Review | Source code files | implementation: completed | "No implementation found. Run implementation first." |
| Testing | Source code files | code-review: completed (approved) or standalone | "Code review is blocked — fix issues before testing." |
| Documentation | Source code files | testing: completed or standalone | "No code found to document." |

**Graceful degradation** — when optional inputs are missing, announce reduced scope:
- Code review without `spec.md` → "No spec available. Reviewing against best practices
  only — cannot verify spec compliance (Area 10)."
- Code review without `plan.md` → "No plan available. Cannot verify plan compliance
  (Area 11) or classify deviations."
- Testing without `review.md` → "No review available. Skipping review cross-reference
  and regression tests for review findings."
- Documentation without `spec.md`/`plan.md` → "No spec/plan available. Generating docs
  from code only — feature descriptions may be less detailed."

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
- **Both brainstorm and investigation:** If `pipeline-state.json` contains both a
  `brainstorm` and `investigation` key, flag this as unusual — they are alternatives.
  Ask the user which one applies.

### Pipeline Health Check

Triggered by "pipeline status", "check health", "what's the state of the pipeline?",
"is everything okay?". Runs a full validation without starting any phase:

1. **Read `pipeline-state.json`** — report current phase and progress
2. **Verify all artifacts** — check that every file listed in `outputs` arrays exists
3. **Check for orphaned fix plans** — `fix-plan.md` or `fix-test-plan.md` exists but
   no phase is blocked (leftover from a previous cycle)
4. **Check for stale state** — `current_phase` doesn't match actual progress
5. **Check git status** — uncommitted changes, diverged branches
6. **Report** with recovery options for any issues found

### Error Recovery

Common failure modes and structured recovery paths:

| Failure | Try First | Then Try | Finally |
|---------|-----------|----------|---------|
| Build fails repeatedly | Read error, fix root cause | Check Node/TS version match | Ask user — may need dependency update |
| Dependency install fails | `npm cache clean --force`, retry | Check package name for typosquat | Ask user — may need auth/registry config |
| Tests fail after fix-plan changes | Run full test suite, check regression | Revert last fix, try alternative approach | Generate `fix-test-plan.md`, block to implementation |
| Type errors cascade | Fix root type error first (usually in types/) | Check tsconfig strictness settings | Ask user — may need type strategy change |
| Out of disk/memory | Clear node_modules, reinstall | Reduce test parallelism | Ask user — environment issue |
| Lint errors on generated code | Auto-fix: `npm run lint -- --fix` | Fix manually if auto-fix fails | Disable specific rule with justification comment |

### Continuation Logic

When the user says "next step", "continue", "go ahead", or similar:

1. Read `pipeline-state.json`
2. Find `current_phase` and phase statuses
3. **Phase resolution:** Check `selected_phases`. If present, skip any phase not in the
   list when looking for the next pending phase. If `selected_phases` is absent, use all
   phases (backward compatible). **Fix loop exception:** `code-review: blocked →
   implementation` and `testing: blocked → implementation` always activate regardless of
   `selected_phases`. If implementation is not in `selected_phases`, ask the user first.
4. Route to the next action:

| Current state | Next action |
|---|---|
| No pipeline-state.json | Ask: new project (brainstorm) or investigate existing code? |
| brainstorm/investigation: completed, planning: pending | Start Planning |
| planning: completed, implementation: pending | Start Implementation |
| implementation: completed, code-review: pending | Start Code-Review |
| code-review: completed (approved), testing: pending | Start Testing |
| code-review: blocked | Start Implementation (fix mode — reads fix-plan.md) |
| testing: completed, documentation: pending | Start Documentation |
| testing: blocked | Start Implementation (test-fix mode — reads fix-test-plan.md) |
| documentation: completed | Pipeline complete — summarize and suggest new work |
| Any phase: in-progress | Resume that phase |

### Handoff Resolution

When a phase completes and needs to suggest the next step, it resolves the next phase
dynamically from `pipeline-state.json`:

**Algorithm:** Read `selected_phases`. Starting from the phase after the current one in
the ordered sequence [planning, implementation, code-review, testing, documentation],
find the first phase that (a) exists in `selected_phases` and (b) has status `pending`.
For brainstorm or investigation, the search starts from the beginning of the sequence.
If `selected_phases` is absent, use all phases. If no next phase is found, the pipeline
is complete for the selected scope.

**Fix loop exception:** code-review blocked and testing blocked always route to
implementation, regardless of `selected_phases`. These are mandatory recovery loops.
If implementation is not in `selected_phases`, ask the user: "Code review/testing found
issues that need fixing, but implementation was not in the selected phases. Should I add
it to fix these issues, or would you prefer to handle this manually?"

**Phase completion message templates:**

| Completing phase | If next phase exists | If no next phase |
|---|---|---|
| brainstorm/investigation | "Your spec is ready! The next step is **{next}**." | "Spec ready — that completes the selected pipeline." |
| planning | "Plan ready — {N} tasks. Next step: **{next}**." | "Plan ready — that completes the selected pipeline." |
| implementation | "Implementation complete. Next step: **{next}**." | "Implementation complete — that completes the selected pipeline." |
| code-review (approved) | "Approved. → **{next}** phase." | "Approved — pipeline complete for selected scope." |
| code-review (blocked) | "Blocked. → Implementation → Re-review." (always) | (N/A — always loops) |
| testing (passing) | "Testing complete. Next step: **{next}**." | "Testing complete — pipeline complete for selected scope." |
| testing (blocked) | "Blocked. → Implementation → Re-test." (always) | (N/A — always loops) |
| documentation | "Pipeline complete." (always — final phase) | "Pipeline complete." |

Each phase skill uses its own context-appropriate wording (task counts, coverage numbers,
etc.) but resolves `{next}` using this algorithm. Session split suggestions remain unchanged.

### Intent Detection Rules

When routing a new user request (not "next step" / "continue"):

**Rule 0 — Phase selection from natural language.** If the user's request includes
phase exclusion or inclusion language, extract `selected_phases` without showing the
menu. Store in `pipeline-state.json` and proceed directly to the first selected phase.
Examples:
- "Build me an API, skip review and docs" → `selected_phases`: ["brainstorm", "planning", "implementation", "testing"]
- "Just brainstorm and plan this" → `selected_phases`: ["brainstorm", "planning"]
- "Implement and test this" → `selected_phases`: ["implementation", "testing"]
- "Full pipeline" or no phase mentions → `selected_phases`: all phases (default)

**Rule 1 — Pipeline state takes priority.** If `pipeline-state.json` exists with an active pipeline, check if the request relates to the current pipeline before starting a new one.

**Rule 2 — Build-something intent => Brainstorm.** Keywords: "build", "create", "develop", "new feature", "new project", "add a feature", "I want to make", "I have an idea", "design", "architect", "set up", "scaffold", "bootstrap", "spin up", "prototype", "MVP", "add support for", "I need a", "let's make", "we need a", "how should we build", "from scratch", "start fresh".

**Rule 3 — Fix-something intent => Investigation.** Keywords: "fix", "bug", "broken", "debug", "refactor", "slow", "performance", "tech debt", "optimize", "root cause", "something is wrong", "not working", "error", "crash", "failing", "exception", "stack trace", "regression", "flaky", "intermittent", "memory leak", "timeout", "hanging", "bottleneck", "doesn't work", "stopped working", "keeps crashing", "what's causing", "why is this", "clean up", "modernize", "simplify", "decouple", "code smell", "hacky", "messy", "legacy", "latency", "N+1", "since upgrading".

**Rule 4 — Explicit phase name => that phase directly.** "brainstorm" => Brainstorm, "investigate" => Investigation, "plan" => Planning, "implement" => Implementation, "review" => Code-Review, "test" => Testing, "document"/"docs" => Documentation.

**Rule 5 — Has artifacts => skip to appropriate phase.** User provides spec => Planning. User provides plan => Implementation. Code exists but no spec/plan => Code-Review, Testing, or Documentation as requested.

## Phase Summary

### Phase 1a: Brainstorm
**Skill:** `brainstorm/SKILL.md`
**Reference:** `brainstorm/references/creativity-techniques.md`
**Purpose:** Deep-dive into requirements through conversational questioning.
**Output:** `spec.md` (concise summary) + detailed design doc
**Key:** One question at a time, checks existing project context first, offers web research step (similar projects, best practices, library comparisons), diverge-converge approach exploration with creativity techniques (pre-mortem, inversion, stakeholder role-play), assumption surfacing, YAGNI ruthlessly. Uses extended thinking at critical design decisions.
**When:** New features, new projects, adding functionality.

### Phase 1b: Investigation
**Skill:** `investigation/SKILL.md`
**Reference:** `investigation/references/investigation-patterns.md`
**Purpose:** Systematic investigation of bugs, performance issues, refactoring needs, or tech debt.
**Output:** `spec.md` (adapted for fixes) + investigation report
**Key:** Reproduce first, offers web research step (known issues, solutions, changelogs), trace root causes with evidence, multiple hypotheses, propose fix strategies with trade-offs.
**When:** Bug fixes, refactoring, performance optimization, tech debt, debugging.

### Phase 2: Planning
**Skill:** `planning/SKILL.md`
**Purpose:** Break the spec into an ordered task list with dependencies and acceptance criteria.
**Output:** `plan.md` with task dependency graph
**Key:** Every task has files to create, dependencies, acceptance criteria. Tasks grouped into phases.

### Phase 3: Implementation
**Skill:** `implementation/SKILL.md`
**References:**
- `implementation/references/clean-code-principles.md`
- `implementation/references/js-ts-best-practices.md`
- `implementation/references/databases-sql.md`
- `implementation/references/databases-nosql.md`
- `implementation/references/databases-redis.md`

**Purpose:** Execute the plan — write code, install deps, verify acceptance criteria.
**Output:** Working code, passing builds, TDD unit tests (for tasks marked `tdd`)
**Key:** Critical plan review before coding, tech stack detection, TDD for pure logic tasks (Red-Green-Refactor), batch execution (3 tasks per batch), quality gate after every batch (types + build + lint + tests), security gate, hard STOP on blockers. Follows Clean Code principles (meaningful names, small functions, clean error handling, DRY, SRP). Uses extended thinking for plan review, complex code, and self-review.

### Phase 4: Code Review
**Skill:** `code-review/SKILL.md`
**References:**
- `code-review/references/review-checklists.md`
- `implementation/references/clean-code-principles.md` (shared)
- `implementation/references/js-ts-best-practices.md` (shared)
- `implementation/references/databases-*.md` (shared, if applicable)
- `implementation/references/frameworks-*.md` (shared, if applicable)
**Purpose:** Strict review across 18 areas — find every issue.
**Output:** `review.md` + `fix-plan.md`
**Key:** 4-level verdict (Block / Request Changes / Approve+suggestions / Approve). If blocked, loops back to implementation. Diff mode for re-reviews. Uses same Clean Code and best-practice references as implementation for consistent standards.

### Phase 5: Testing
**Skill:** `testing/SKILL.md`
**Purpose:** Expand test coverage — integration, e2e, security, edge cases, benchmarks.
**Output:** Test files + `test-report.md`
**Key:** Builds on TDD unit tests from implementation (doesn't duplicate). Auto-detect test runner (default Vitest), >80% coverage target, generate-run-fix loop, production bugs documented.

### Phase 6: Documentation
**Skill:** `documentation/SKILL.md`
**Reference:** `documentation/references/doc-templates.md`
**Purpose:** Generate all project documentation from actual code.
**Output:** README.md, API docs, architecture, contributing, changelog, deployment guide, doc-report.md
**Key:** Full or update mode (incremental updates via git diff). Doc coverage analysis (100% public API target). Drift detection for stale docs. Every statement verified against code with cross-reference validation. Reads `doc_impacts` from code review. No aspirational docs.

## Handling User Requests

| User says | Action |
|---|---|
| "Build me a REST API for..." | Start Phase 1a (Brainstorm) |
| "I want to build..." / "New feature" | Start Phase 1a (Brainstorm) |
| "I need a service that..." / "Let's make a..." | Start Phase 1a (Brainstorm) |
| "Help me design a..." / "How should we build..." | Start Phase 1a (Brainstorm) |
| "Set up a new project" / "Scaffold a..." | Start Phase 1a (Brainstorm) |
| "Fix this bug" / "Debug this" | Start Phase 1b (Investigation) |
| "Refactor this" / "Clean up this code" | Start Phase 1b (Investigation) |
| "Performance issue" / "This is slow" | Start Phase 1b (Investigation) |
| "Investigate" / "Find the root cause" | Start Phase 1b (Investigation) |
| "Tech debt" / "Why is this broken?" | Start Phase 1b (Investigation) |
| "Something broke" / "This stopped working" | Start Phase 1b (Investigation) |
| "Why is this failing?" / "What's causing this?" | Start Phase 1b (Investigation) |
| "This is flaky" / "Intermittent failures" | Start Phase 1b (Investigation) |
| "Memory leak" / "CPU is spiking" / "Latency issues" | Start Phase 1b (Investigation) |
| "I have a spec, let's plan" | Start Phase 2 (Planning), read their spec |
| "Break this into tasks" / "Make a task list" | Start Phase 2 (Planning) |
| "Here's my plan, start coding" | Start Phase 3 (Implementation), read their plan |
| "Start building" / "Let's code this" | Start Phase 3 (Implementation) — check for plan.md |
| "Review this code" | Start Phase 4 (Code Review) |
| "Write tests" | Start Phase 5 (Testing) |
| "Generate docs" | Start Phase 6 (Documentation) |
| "Fix the review issues" / "Address the feedback" | Start Phase 3 (Implementation) in fix mode — read `fix-plan.md` |
| "Fix the failing tests" / "Fix test failures" | Start Phase 3 (Implementation) in test-fix mode — read `fix-test-plan.md` |
| "Continue" / "Next step" | Check pipeline-state.json, proceed to next phase |
| "What's the status?" / "How are we doing?" / "Where are we?" | Read pipeline-state.json, summarize progress |
| "Check health" / "Pipeline status" / "Is everything okay?" | Run pipeline health check — validate all artifacts, detect issues |
| "Start over" / "Start fresh" / "New pipeline" | Reset pipeline-state.json, start Phase 1 |
| "Skip to testing" | Jump to Phase 5, note skipped phases |

## Context Management

The pipeline is designed for **session splitting**. Each phase reads its inputs from disk
(spec.md, plan.md, review.md, etc.) and writes its outputs to disk. `pipeline-state.json`
tracks where you are. This means:

- **A new conversation can pick up from any phase cleanly.** The user just says "continue"
  and you read `pipeline-state.json` to know exactly where to resume.
- **Splitting sessions between phases is encouraged** — especially after the three heaviest
  phases: brainstorm/investigation, implementation, and code review.

### When to Suggest a New Session

Suggest starting a new session when:

- **A phase just completed** and the conversation is long (many tool calls, multiple
  batches completed, extensive research or discussion). Say:
  > "Phase complete — all artifacts are saved to disk. If this conversation is getting
  > long, you can start a new session and say 'continue' — I'll pick up right where
  > we left off from pipeline-state.json."
- **Implementation has completed 6+ tasks** — context is likely heavy with code, batch
  reports, and reference materials from earlier phases.
- **After brainstorm/investigation with web research** — research results consume
  significant context that's no longer needed once spec.md is written.
- **After code review** — review findings, file-by-file analysis, and fix-plan are all
  on disk. A fresh session for testing starts clean.

### Phase Timeout Heuristics

Monitor conversation weight and suggest session splits proactively:

| Signal | Threshold | Action |
|--------|-----------|--------|
| Implementation batches | 3+ batches completed | Suggest session split after current batch |
| Code review files | 15+ files reviewed | Suggest split before generating reports |
| Brainstorm with web research | 6+ web searches done | Suggest split after spec is written |
| Any phase | Conversation feels slow or repetitive | State is on disk — suggest fresh session |

### New Session Startup Protocol

When a user says "continue" in a new session:

1. Read `pipeline-state.json` — determine current phase and status
2. Check `selected_phases` — if absent, default to all phases (backward compatible)
3. Verify phase outputs exist on disk (the files listed in `outputs` arrays)
4. If outputs are missing, warn the user and suggest re-running that phase
5. Load the next phase's SKILL.md and proceed normally — all inputs come from disk

### Sub-Agent Guidance

Use sub-agents to keep the main conversation context lean:

- **Web research** (brainstorm/investigation) — launch a sub-agent for web searches,
  have it return a <200 word summary of findings rather than raw search results
- **Codebase exploration** (investigation) — launch a sub-agent for deep codebase
  exploration, have it return only relevant `file:line` references
- **Large code reviews** (code-review) — for codebases with >15 source files, consider
  using sub-agents for batched file-by-file review, consolidating findings in main context
- **Reference lookups** (implementation) — when you need to check a specific pattern from
  a reference file you've already read, use Grep on the reference file instead of
  re-reading the entire file

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
