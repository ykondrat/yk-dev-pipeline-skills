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

You are the orchestrator for a 6-phase development pipeline. Each phase runs as an
**independent agent** in its own context window with scoped tools and model selection.
You manage intent detection, state, prerequisites, handoffs, and result verification.

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

## Architecture: Agent-Based Execution

Each phase is an independent agent (`agents/*.md`) with:
- **Own context window** — no cross-phase pollution
- **Scoped tools** — code-review is read-only, investigation can't edit
- **Per-phase model** — opus for complex phases, sonnet for structured phases
- **Persistent memory** — project-scoped learning across sessions

| Phase | Agent | Model | Key Constraint |
|-------|-------|-------|----------------|
| Brainstorm | `brainstorm` | opus | Full tools + web research |
| Investigation | `investigation` | opus | No Edit — investigate only |
| Planning | `planning` | sonnet | Read/Write only |
| Implementation | `implementation` | opus | Full tools, maxTurns: 100 |
| Code Review | `code-review` | opus | No Edit — read-only review |
| Testing | `testing` | sonnet | Full tools, maxTurns: 80 |
| Documentation | `documentation` | sonnet | Read/Write only |

**Fallback:** If agents are unavailable, fall back to reading the phase SKILL.md
files in `skills/{phase}/SKILL.md` and following instructions directly.

---

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

### Starting a Phase — Agent Invocation

To start any phase, invoke the corresponding agent using the **Agent tool**. Pass full
context in the prompt so the agent can work independently:

**Agent invocation template:**
```
Project: {project_name}
Working directory: {cwd}

Read pipeline-state.json at {cwd}/pipeline-state.json for current state.
{Phase-specific artifact instructions — see below}

Execute the full {phase} process. Update pipeline-state.json when complete.
```

**Phase-specific context to include in the agent prompt:**

| Phase | Include in prompt |
|-------|------------------|
| Brainstorm | "This is a new project. The user wants to build: {user's request}" |
| Investigation | "The user wants to fix/debug: {user's description of the problem}" |
| Planning | "Read spec.md at {cwd}/spec.md and the design doc listed in pipeline-state.json." |
| Implementation | "Read plan.md at {cwd}/plan.md. Mode: {fresh/resume/fix/test-fix}. {If fix: 'Read fix-plan.md'} {If test-fix: 'Read fix-test-plan.md'}" |
| Code Review | "Review all source code. Read spec.md, plan.md for compliance checking." |
| Code Review (parallel) | See Parallel Code Review section below |
| Testing | "Read spec.md, plan.md, review.md for test derivation. Build on existing TDD tests." |
| Documentation | "Generate docs from actual code. Read spec.md, plan.md, review.md, test-report.md for context. Mode: {full/update}." |

### Result Verification

**After each agent completes**, verify its work before proceeding:

1. Read `pipeline-state.json` — confirm the agent updated the phase status
2. Check that output artifacts exist on disk (files listed in `outputs` array)
3. If status is `completed` and artifacts exist → present results to user
4. If status is still `pending` or artifacts missing → warn user, suggest re-running

### Parallel Code Review

For code review, invoke **3 code-review agents simultaneously** with different focus areas:

```
Agent 1 — Security Focus:
"Focus your review on security areas ONLY: Areas 2, 12, 15, 16, 17.
Write findings to {cwd}/review-security.md.
Do NOT write fix-plan.md — the orchestrator will consolidate."

Agent 2 — Quality Focus:
"Focus your review on quality areas ONLY: Areas 1, 3, 4, 5, 6, 7, 8.
Write findings to {cwd}/review-quality.md.
Do NOT write fix-plan.md — the orchestrator will consolidate."

Agent 3 — Compliance Focus:
"Focus your review on compliance areas ONLY: Areas 9, 10, 11, 13, 14, 18.
Write findings to {cwd}/review-compliance.md.
Do NOT write fix-plan.md — the orchestrator will consolidate."
```

**After all three complete**, consolidate:
1. Read `review-security.md`, `review-quality.md`, `review-compliance.md`
2. Merge findings — deduplicate by file:line (if same issue found by multiple agents)
3. Resolve conflicting severities — take highest
4. Generate unified `review.md` combining all findings
5. Generate `fix-plan.md` from consolidated findings (Critical → Major → Minor)
6. Compute overall verdict: Block if ANY agent found criticals, Request Changes if majors
7. Update `pipeline-state.json` with consolidated review results
8. Clean up partial files (delete review-security.md, review-quality.md, review-compliance.md)

**Fallback:** If parallel review is problematic (e.g., agents unavailable), invoke a
single code-review agent for a comprehensive full review.

### Continuing the Pipeline

After each phase completes, suggest the next phase. The user confirms before proceeding.

**Phase 1 complete** → "Your spec is ready! Ready for planning?"
→ Invoke the `planning` agent

**Phase 2 complete** → "Plan ready — {N} tasks. Ready to start implementing?"
→ Invoke the `implementation` agent

**Phase 3 complete** → "Implementation complete. Ready for code review?"
→ Invoke `code-review` agent(s) — parallel by default

**Phase 4 complete (approved)** → "Approved. Ready for testing?"
→ Invoke the `testing` agent

**Phase 4 complete (blocked)** → "Issues found. I'll fix them and re-review."
→ Invoke `implementation` agent (fix mode with fix-plan.md)
→ Then invoke `code-review` agent(s) again (diff review)

**Phase 5 complete** → "Testing complete. Ready for documentation?"
→ Invoke the `documentation` agent

**Phase 6 complete** → "Pipeline complete! Project is built, reviewed, tested, and documented."

### Jumping to a Specific Phase

The user can skip phases or jump to any phase directly:

- "Just implement this" → Start at Phase 3 (note missing spec/plan)
- "Review my code" → Start at Phase 4
- "Write tests for this" → Start at Phase 5
- "Generate docs" → Start at Phase 6
- "Continue from where we left off" → Check pipeline-state.json

### Pipeline State

All phases track progress in `pipeline-state.json`:

```json
{
  "project_name": "{project name}",
  "created_at": "{ISO}",
  "version": "3.0",
  "orchestration_mode": "agents",
  "selected_phases": ["brainstorm", "planning", "implementation", "code-review", "testing", "documentation"],
  "current_phase": "implementation",
  "phases": {
    "brainstorm": { "status": "completed", "outputs": ["spec.md", "docs/plans/..."] },
    "planning": { "status": "completed", "outputs": ["plan.md"] },
    "implementation": { "status": "in-progress", "last_completed_task": 3 },
    "code-review": { "status": "pending" },
    "testing": { "status": "pending" },
    "documentation": { "status": "pending" }
  }
}
```

**`selected_phases`:** Array of phases the user chose to run. If absent, all phases are
selected (backward compatible). Phases not in `selected_phases` are skipped during
continuation and handoff resolution.

**Note:** The first phase key is either `brainstorm` OR `investigation` — they are
alternatives, never both.

### Prerequisite Validation

**Before invoking any phase agent**, validate prerequisites. If validation fails, present
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
- Testing without `review.md` → "No review available. Skipping review cross-reference."
- Documentation without `spec.md`/`plan.md` → "No spec/plan. Generating docs from code only."

### State Validation

When reading `pipeline-state.json`, validate before proceeding:

- **Missing artifacts:** Phase says "completed" but outputs don't exist → warn and suggest re-run
- **Blocked without fix plan:** code-review/testing "blocked" but fix-plan missing → flag inconsistency
- **Stale state:** `current_phase` doesn't match progress → ask user which phase to continue
- **Out-of-order completion:** Later phase done but earlier isn't → ask if intentional
- **Both brainstorm and investigation:** Unusual — ask which applies

### Pipeline Health Check

Triggered by "pipeline status", "check health", "what's the state?", "is everything okay?":

1. Read `pipeline-state.json` — report current phase and progress
2. Verify all artifacts exist on disk
3. Check for orphaned fix plans
4. Check for stale state
5. Check git status
6. Report with recovery options

### Error Recovery

| Failure | Try First | Then Try | Finally |
|---------|-----------|----------|---------|
| Build fails repeatedly | Read error, fix root cause | Check Node/TS version | Ask user |
| Dependency install fails | `npm cache clean --force`, retry | Check for typosquat | Ask user |
| Tests fail after fixes | Full test suite, check regression | Revert, try alternative | Generate fix-test-plan.md |
| Type errors cascade | Fix root type first | Check tsconfig | Ask user |
| Agent fails to complete | Check pipeline-state.json for partial progress | Re-invoke agent | Fall back to skill-based approach |

### Continuation Logic

When the user says "next step", "continue", "go ahead", or similar:

1. Read `pipeline-state.json`
2. Find `current_phase` and phase statuses
3. **Phase resolution:** Check `selected_phases`. Skip phases not in list. **Fix loop
   exception:** code-review blocked → implementation and testing blocked → implementation
   always activate regardless of `selected_phases`.
4. Route to next action and invoke the corresponding agent:

| Current state | Next action |
|---|---|
| No pipeline-state.json | Ask: new project (brainstorm) or investigate existing code? |
| brainstorm/investigation: completed, planning: pending | Invoke `planning` agent |
| planning: completed, implementation: pending | Invoke `implementation` agent |
| implementation: completed, code-review: pending | Invoke `code-review` agent(s) |
| code-review: completed (approved), testing: pending | Invoke `testing` agent |
| code-review: blocked | Invoke `implementation` agent (fix mode) |
| testing: completed, documentation: pending | Invoke `documentation` agent |
| testing: blocked | Invoke `implementation` agent (test-fix mode) |
| documentation: completed | Pipeline complete |
| Any phase: in-progress | Re-invoke that phase's agent with resume context |

### Handoff Resolution

**Algorithm:** Read `selected_phases`. Starting from the phase after the current one in
the ordered sequence [planning, implementation, code-review, testing, documentation],
find the first phase that (a) exists in `selected_phases` and (b) has status `pending`.
If `selected_phases` is absent, use all phases. If no next phase → pipeline complete.

**Fix loop exception:** code-review blocked and testing blocked always route to
implementation, regardless of `selected_phases`.

**Phase completion message templates:**

| Completing phase | If next phase exists | If no next phase |
|---|---|---|
| brainstorm/investigation | "Your spec is ready! The next step is **{next}**." | "Spec ready — pipeline complete for selected scope." |
| planning | "Plan ready — {N} tasks. Next step: **{next}**." | "Plan ready — pipeline complete." |
| implementation | "Implementation complete. Next step: **{next}**." | "Implementation complete — pipeline complete." |
| code-review (approved) | "Approved. → **{next}** phase." | "Approved — pipeline complete." |
| code-review (blocked) | "Blocked. → Implementation → Re-review." (always) | (N/A) |
| testing (passing) | "Testing complete. Next step: **{next}**." | "Testing complete — pipeline complete." |
| testing (blocked) | "Blocked. → Implementation → Re-test." (always) | (N/A) |
| documentation | "Pipeline complete." | "Pipeline complete." |

### Intent Detection Rules

**Rule 0 — Phase selection from natural language.** Extract `selected_phases` from user's request.

**Rule 1 — Pipeline state takes priority.** Check `pipeline-state.json` before starting new.

**Rule 2 — Build-something intent => Brainstorm.** Keywords: "build", "create", "develop", "new feature", "new project", "add a feature", "I want to make", "I have an idea", "design", "architect", "set up", "scaffold", "bootstrap", "spin up", "prototype", "MVP", "I need a", "let's make", "we need a", "from scratch", "start fresh".

**Rule 3 — Fix-something intent => Investigation.** Keywords: "fix", "bug", "broken", "debug", "refactor", "slow", "performance", "tech debt", "optimize", "root cause", "not working", "error", "crash", "failing", "regression", "flaky", "memory leak", "timeout", "bottleneck", "clean up", "modernize", "legacy", "N+1".

**Rule 4 — Explicit phase name => that phase directly.**

**Rule 5 — Has artifacts => skip to appropriate phase.**

## Phase Summary

### Phase 1a: Brainstorm
**Agent:** `brainstorm` (opus, blue)
**Purpose:** Deep-dive requirements via conversational questioning.
**Output:** `spec.md` + detailed design doc
**Improvements:** Constraint-based exploration, multi-perspective rewriting, completeness scoring

### Phase 1b: Investigation
**Agent:** `investigation` (opus, purple)
**Purpose:** Systematic investigation of bugs, performance, refactoring.
**Output:** `spec.md` + investigation report
**Improvements:** Git bisect integration, hypothesis-evidence matrix, evidence thresholds

### Phase 2: Planning
**Agent:** `planning` (sonnet, green)
**Purpose:** Break spec into ordered, dependency-aware task list.
**Output:** `plan.md` with task DAG
**Improvements:** Task DAG, solvability verification, error propagation analysis

### Phase 3: Implementation
**Agent:** `implementation` (opus, orange)
**Purpose:** Execute plan — write code, install deps, verify.
**Output:** Working code, passing builds
**Improvements:** Strict incremental execution, scaffolding-first, per-task self-verification

### Phase 4: Code Review
**Agent:** `code-review` (opus, red) — **runs as 3 parallel agents**
**Purpose:** Strict review across 18 areas. Read-only.
**Output:** `review.md` + `fix-plan.md`
**Improvements:** Parallel review, confidence scoring, trajectory analysis, counterfactual analysis

### Phase 5: Testing
**Agent:** `testing` (sonnet, yellow)
**Purpose:** Expand test coverage — integration, e2e, security, edge cases.
**Output:** Test files + `test-report.md`
**Improvements:** Mutation testing, property-based testing, multi-metric quality assessment

### Phase 6: Documentation
**Agent:** `documentation` (sonnet, cyan)
**Purpose:** Generate all project docs from actual code.
**Output:** README.md, API docs, architecture, contributing, changelog, deployment
**Improvements:** ADR automation, living documentation, command verification

## Handling User Requests

| User says | Action |
|---|---|
| "Build me a REST API for..." | Invoke `brainstorm` agent |
| "I want to build..." / "New feature" | Invoke `brainstorm` agent |
| "Fix this bug" / "Debug this" | Invoke `investigation` agent |
| "Refactor this" / "Clean up this code" | Invoke `investigation` agent |
| "Performance issue" / "This is slow" | Invoke `investigation` agent |
| "I have a spec, let's plan" | Invoke `planning` agent |
| "Here's my plan, start coding" | Invoke `implementation` agent |
| "Review this code" | Invoke `code-review` agent(s) — parallel |
| "Write tests" | Invoke `testing` agent |
| "Generate docs" | Invoke `documentation` agent |
| "Fix the review issues" | Invoke `implementation` agent (fix mode) |
| "Fix the failing tests" | Invoke `implementation` agent (test-fix mode) |
| "Continue" / "Next step" | Check pipeline-state.json, invoke next agent |
| "What's the status?" | Read pipeline-state.json, summarize |
| "Check health" | Run pipeline health check |
| "Start over" | Reset pipeline-state.json, start Phase 1 |

## Context Management

The pipeline is designed for **session splitting**. Each agent reads inputs from disk
and writes outputs to disk. `pipeline-state.json` tracks state. Because agents run in
their own context windows, the main conversation stays clean — no reference file bloat.

### When to Suggest a New Session

- After heavy phases (brainstorm with research, implementation with many tasks, code review)
- After 6+ implementation tasks
- After parallel code review consolidation
- When conversation feels slow

> "Phase complete — artifacts on disk. Start a new session and say 'continue' to pick up cleanly."

### New Session Startup Protocol

1. Read `pipeline-state.json`
2. Check `selected_phases`
3. Verify phase outputs exist on disk
4. Invoke the next phase's agent

## Important Rules

1. **Invoke the agent for each phase.** Agents have detailed instructions, references, and self-verification gates.
2. **Pass full context in agent prompts.** Agents don't see the main conversation — include project path, artifact locations, and pipeline state.
3. **Verify results after each agent.** Check pipeline-state.json and output artifacts.
4. **Update pipeline-state.json** after each phase and agent consolidation.
5. **Suggest next phase** but wait for user confirmation before invoking.
6. **If user provides their own artifacts**, adapt — don't force earlier phases.
7. **Use parallel code review by default.** Fall back to single agent if needed.