---
description: Resume the ai-tools pipeline run from pipeline-state.json — picks up the next pending phase or fix-loop.
argument-hint: "[optional note to pass to the next phase]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task
---

You are resuming an in-flight **ai-tools pipeline** run.

## Optional user note

$ARGUMENTS

## What to do

1. **Read `pipeline-state.json`** in the working directory. If it does not exist or
   fails schema validation against
   [`pipeline-state.schema.json`](../pipeline-state.schema.json), stop and tell the
   user to run `/pipeline-start` first.

2. **Determine the next action** by inspecting `phases` and `current_phase`:

   | State observed | Next action |
   |----------------|-------------|
   | Any phase `status: "in-progress"` with partial outputs | **Resume that phase** with the same agent (pass `last_completed_task` / `test_batches_completed`) |
   | `phases["code-review"].status: "blocked"` | Route to **implementation** in fix mode — read `fix-plan.md` |
   | `phases["testing"].status: "blocked"` | Route to **implementation** in fix-test mode — read `fix-test-plan.md` |
   | Current phase `status: "completed"` | Advance to next phase in `selected_phases` |
   | All selected phases `completed` | Set `current_phase: "complete"` and summarize outputs |

3. **Invoke the correct agent via Task** — do not execute phase work yourself.

4. **Never skip failure loops.** A `blocked` verdict always routes back to
   implementation before the pipeline can move forward.

5. **After each agent returns**, update `pipeline-state.json` (validate against the
   schema) and ask the user whether to continue.

If `$ARGUMENTS` is non-empty, pass it as additional context/instructions to the next
agent (e.g., "prioritize performance fixes", "skip a11y findings").
