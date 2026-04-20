---
description: Kick off a fresh ai-tools pipeline run — brainstorm/investigation → planning → implementation → code-review → testing → documentation.
argument-hint: "[feature description | bug to fix | refactor target]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task
---

You are invoking the **ai-tools pipeline orchestrator** for a new pipeline run.

## User request

$ARGUMENTS

## What to do

1. **Run the orchestrator.** Load
   [`skills/SKILL.md`](../skills/SKILL.md) (or the
   `yk-dev-pipeline` skill) and follow its routing rules.

2. **Detect intent** from the request above:
   - Ambiguous / new-feature / greenfield → start at **brainstorm** (Phase 1a).
   - Bug / refactor / debugging / performance → start at **investigation** (Phase 1b).
   - Pure review / tests / docs on existing code → route directly to that phase.

3. **Ask clarifying questions first** via AskUserQuestion:
   - Confirm the detected entry phase.
   - Offer phase-selection (which of the 7 phases to include).
   - Confirm the working directory for artifacts (`./` vs a sub-folder).

4. **Initialize `pipeline-state.json`** against
   [`pipeline-state.schema.json`](../pipeline-state.schema.json) with:
   - `project_name` derived from the request or cwd
   - `version: "3.0"`
   - `orchestration_mode: "agents"` (falls back to `"skills"` only if agents unavailable)
   - `selected_phases` from the user's answer
   - `current_phase` = the chosen entry phase, `status: "pending"`

5. **Invoke the first phase agent** via the Task tool with the scoped subagent (e.g.
   `brainstorm`, `investigation`). Do NOT execute phase work yourself — delegate.

6. **After the agent completes**, surface its outputs (file paths), update
   `pipeline-state.json`, and ask whether to continue to the next selected phase.

Resume-safety: if `pipeline-state.json` already exists in the working directory, refuse
to overwrite — tell the user to use `/pipeline-continue` or `/pipeline-reset` instead.
