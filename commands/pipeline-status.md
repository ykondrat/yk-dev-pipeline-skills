---
description: Show current ai-tools pipeline status — phase progress, findings counts, blockers, and suggested next step.
argument-hint: ""
allowed-tools: Read, Glob, Grep, Bash
---

You are producing a **status snapshot** of the current ai-tools pipeline run.

## What to do

1. **Read `pipeline-state.json`** in the working directory. If missing, tell the user
   no pipeline is in progress and suggest `/pipeline-start`.

2. **Validate** the state against
   [`pipeline-state.schema.json`](../pipeline-state.schema.json). Call out any schema
   violations (stale fields, invalid enums, `brainstorm`+`investigation` both present).

3. **Render a compact report** (no code changes, no phase work):

   ```
   Project: {project_name}
   Version: {version} ({orchestration_mode})
   Entry:   {brainstorm | investigation}
   Phases:  {selected_phases}
   Current: {current_phase}

   Phase          Status        Outputs                              Notes
   -------------  ------------  -----------------------------------  ---------------------------------
   brainstorm     ✅ completed  spec.md
   planning       ✅ completed  plan.md                              {task_count} tasks / {phase_count} phases
   implementation 🟡 in-progress src/**, tests/**                    task {current_task}/{task_count}
   code-review    ⛔ blocked    review.md, fix-plan.md               🔴 {critical} 🟡 {major} 🔵 {minor}
   testing        ⏸ pending
   documentation  ⏸ pending
   ```

4. **Suggest the next step**:
   - If a `blocked` verdict exists → `/pipeline-continue` (routes to implementation fix mode).
   - If `in-progress` phase → `/pipeline-continue` (resume same agent).
   - If all done → celebrate, link to final outputs.
   - If schema invalid → tell the user what to fix.

5. **List artifact paths** produced so far (relative to project root) so the user can
   open them directly.

Do NOT invoke any phase agent. This command is read-only.
