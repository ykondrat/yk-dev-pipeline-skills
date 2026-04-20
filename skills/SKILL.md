---
name: yk-dev-pipeline
description: >
  Main entry point for the 6-phase JS/TS development pipeline: brainstorm (or
  investigation), planning, implementation, code-review, testing, documentation.
  Auto-detects whether agents are available — prefers agent-based orchestration
  (v3.0) when running inside the installed plugin, falls back to skill-based
  orchestration (v2.0) for Claude.ai Projects uploads and skill-only installs.
  Semi-automatic — suggests the next step, the user confirms. Triggers on:
  "start a new project", "build me a...", "let's develop...", "new feature",
  "start pipeline", "fix this bug", "debug this", "investigate", "refactor",
  "performance issue", "tech debt", "I want to build", "I need a...",
  "help me create", "let's make", "new app", "new service", "new API",
  "add a feature", "something is broken", "this doesn't work", "not working",
  "it's broken", "this is slow", "this keeps failing", "there's an error",
  "help me fix", "figure out why", "what's wrong with", "clean up this code",
  "improve this", "modernize", "start developing", "let's get started",
  "I have an idea for", "set up a project", "scaffold", "prototype",
  "memory leak", "crashing", "regression", "flaky", or any request to build,
  fix, or improve a JS/TS application.
metadata:
  recommended_model: sonnet
  orchestration_mode: auto
---

# JS/TS Development Pipeline — Router + Skill-Orchestrator

You are the **main entry point** for the 6-phase development pipeline. You have two
execution paths and you pick the right one up front:

- **Agent path (preferred)** — if the Task tool is available and
  `agents/*.md` (at the plugin root) exists, hand off to the agent-orchestrator
  at [`agent-orchestrator/SKILL.md`](./agent-orchestrator/SKILL.md) (skill name
  `yk-agent-orchestrator`). Agents run in isolated context windows, support
  parallel code review, and give higher-quality output.
- **Skill path (fallback)** — if agents are not available (typical of Claude.ai
  Projects uploads and skill-only installs), run the pipeline **in-place** using
  the phase SKILL.md files directly. That's what the rest of this file is about.

Before doing anything else, decide which path applies.

## Path Selection (first 30 seconds)

1. Check if the Task / Agent tool is available to you.
2. If yes, Glob for `**/agents/*.md` (agents live at the plugin root). If the seven phase
   agents are present (brainstorm, investigation, planning, implementation,
   code-review, testing, documentation), **hand off**:

   > "I can run this pipeline with the agent-orchestrator for better isolation
   > and parallel code review. Switching to `yk-agent-orchestrator`."

   Then follow the agent-orchestrator skill and stop reading this file.

3. If agents are unavailable OR the Task tool is unavailable, continue below —
   you are the **skill-orchestrator**.

Record the chosen path in `pipeline-state.json` as `orchestration_mode`:
`"agents"` or `"skills"`.

---

# Skill-Orchestrator (v2.0 Flow)

Everything below runs when agents are unavailable. Instead of delegating to
subagents, you execute each phase yourself by **reading the phase's SKILL.md**
into this conversation and following its instructions.

## Pipeline Overview

```
[1a. Brainstorm] ─┐
                  ├→ [2. Planning] → [3. Implementation] → [4. Code Review] → [5. Testing] → [6. Documentation]
[1b. Investigation]┘       ↓                 ↓                    ↓                  ↓                ↓
                        plan.md         working code        review.md + fix-plan   test-report.md    README + docs/
       ↓                                                         ↓ (if blocked)
    spec.md                                                 back to Implementation
```

Phase 1 has two alternatives — **brainstorm** (new work) or **investigation**
(bug/refactor). Mutually exclusive in `pipeline-state.json`.

## Phase SKILL.md Locations

| Phase | Skill file |
|-------|-----------|
| 1a. Brainstorm | [`brainstorm/SKILL.md`](./brainstorm/SKILL.md) |
| 1b. Investigation | [`investigation/SKILL.md`](./investigation/SKILL.md) |
| 2. Planning | [`planning/SKILL.md`](./planning/SKILL.md) |
| 3. Implementation | [`implementation/SKILL.md`](./implementation/SKILL.md) |
| 4. Code Review | [`code-review/SKILL.md`](./code-review/SKILL.md) |
| 5. Testing | [`testing/SKILL.md`](./testing/SKILL.md) |
| 6. Documentation | [`documentation/SKILL.md`](./documentation/SKILL.md) |

**Reference library** (shared, at plugin root): `references/`.
Each phase skill specifies which references to load. Use Glob + Read to load them
on demand — do not preload the whole library.

## How to Use

### Phase Selection

When starting fresh (no `pipeline-state.json`, or the user says "start over"),
present the phase menu via `AskUserQuestion`:

> Which phases do you want to include?
>
> Presets:
>   [A] Full pipeline — recommended
>   [B] Quick build (brainstorm/investigation → planning → implementation)
>   [C] Review & polish (code review → testing → documentation)
>   [D] Custom

Normalize free-form answers to canonical phase names: `brainstorm`, `investigation`,
`planning`, `implementation`, `code-review`, `testing`, `documentation`.

Store in `pipeline-state.json` as `selected_phases`. Omitted → all phases selected.

### Initialize State

```json
{
  "project_name": "{derived}",
  "created_at": "{ISO}",
  "version": "3.0",
  "orchestration_mode": "skills",
  "selected_phases": [/* user's answer */],
  "current_phase": "{first selected phase}",
  "phases": { "{first selected phase}": { "status": "pending" } }
}
```

Validate against [`../pipeline-state.schema.json`](../pipeline-state.schema.json)
before writing.

### Running a Phase (Skill-Only)

1. Read the phase SKILL.md into the conversation (e.g.,
   `brainstorm/SKILL.md`).
2. Read the references it declares you should load (via Glob + Read).
3. Execute the instructions top-to-bottom.
4. Produce the required artifacts on disk (`spec.md`, `plan.md`, etc.).
5. Update `pipeline-state.json` with `status`, `outputs`, and any phase-specific
   fields (`task_count`, `verdict`, `findings_counts`, `test_counts`, etc.).
   Validate against the schema before writing.
6. Announce completion + suggest the next phase.

Because you do not have isolated contexts in skill mode, context will grow as
you progress. After heavy phases (brainstorm with lots of Q&A, implementation
with many tasks, code review), **suggest a new session** so the user can say
`continue` in a fresh conversation — the pipeline state will pick up cleanly.

### Code Review in Skill Mode

Skill mode runs code review as **a single pass** across all 18 areas, not in
parallel. The parallel-review optimization only works with agent invocation.
Follow the steps in [`code-review/SKILL.md`](./code-review/SKILL.md) and write
`review.md` + `fix-plan.md` directly.

### Handoff & Continuation

Follow the ordered sequence `[brainstorm | investigation, planning,
implementation, code-review, testing, documentation]`, walking forward to the
next phase in `selected_phases` whose status is `pending`.

**Fix-loop exception (always overrides `selected_phases`):**

- `code-review.status === "blocked"` → run `implementation` in **fix mode**
  (read `fix-plan.md`, apply fixes), then re-run code review on the diff.
- `testing.status === "blocked"` → run `implementation` in **test-fix mode**
  (read `fix-test-plan.md`), then re-run testing.

### Prerequisite Validation

| Phase | Required files | Required prior status | If missing |
|-------|---------------|----------------------|------------|
| Planning | `spec.md` | brainstorm OR investigation: completed | Offer to run brainstorm/investigation |
| Implementation | `plan.md` | planning: completed | Offer to run planning |
| Implementation (fix) | `fix-plan.md` | code-review: blocked | Offer to re-review |
| Implementation (test-fix) | `fix-test-plan.md` | testing: blocked | Offer to re-run testing |
| Code Review | source files | implementation: completed OR standalone | Run standalone if code exists |
| Testing | source files | code-review: completed OR standalone | Run standalone if code exists |
| Documentation | source files | testing: completed OR standalone | Run standalone if code exists |

When an optional input is missing, announce reduced scope (e.g., "No `spec.md` —
code review will skip Area 10 — spec compliance").

### Intent Detection

- **Rule 0** — phase selection from natural language.
- **Rule 1** — existing `pipeline-state.json` wins; offer to resume.
- **Rule 2** — "build / create / new feature" → `brainstorm`.
- **Rule 3** — "fix / bug / debug / refactor / slow / tech debt" → `investigation`.
- **Rule 4** — explicit phase name → that phase.
- **Rule 5** — artifacts already exist → skip to the appropriate phase.

### Pipeline Health Check (read-only)

On "pipeline status" / "check health":

1. Read `pipeline-state.json`.
2. Validate against the schema.
3. Verify each `outputs` file exists and is non-empty.
4. Cross-check: blocked phases must have a fix plan on disk.
5. Report status + suggest the next step.

---

## Handling User Requests

| User says | Skill-mode action |
|---|---|
| "Build me a REST API for..." | Read `brainstorm/SKILL.md`, follow it |
| "Fix this bug" / "Debug this" | Read `investigation/SKILL.md`, follow it |
| "Refactor this" / "Performance issue" | Read `investigation/SKILL.md`, follow it |
| "I have a spec, let's plan" | Read `planning/SKILL.md`, follow it |
| "Start coding" (with plan.md) | Read `implementation/SKILL.md`, follow it |
| "Review this code" | Read `code-review/SKILL.md`, follow it |
| "Write tests" | Read `testing/SKILL.md`, follow it |
| "Generate docs" | Read `documentation/SKILL.md`, follow it |
| "Continue" / "Next step" | Read `pipeline-state.json`, go to next skill |
| "What's the status?" | Pipeline health check |
| "Start over" | Reset `pipeline-state.json`, Phase 1 |

## Important Rules

1. **Try the agent path first.** Hand off to `yk-agent-orchestrator` whenever
   agents are available — you get parallel review and isolation for free.
2. **In skill mode, read the phase SKILL.md every time.** Do not rely on memory.
3. **Validate `pipeline-state.json` against the schema before every write.**
4. **Suggest next phase, then wait for confirmation.** Never autopilot.
5. **Respect fix loops.** `blocked` verdicts always route back to implementation.
6. **Respect `selected_phases`.** Skip phases not in the list — except for fix
   loops, which always activate.
7. **Suggest new sessions after heavy phases** to keep context usable.

## Upgrade Path

When agents later become available (user installs the plugin, or enables the
Agent tool), say:

> "Agents are now available. I can switch to the agent-orchestrator for
> isolation + parallel review. Switch?"

On confirmation, update `orchestration_mode` to `"agents"` and hand off to
`yk-agent-orchestrator`. All existing artifacts (`spec.md`, `plan.md`, etc.)
carry forward unchanged.
