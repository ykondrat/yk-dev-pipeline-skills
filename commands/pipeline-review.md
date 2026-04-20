---
description: Run the comprehensive 18-area code review — standalone or in parallel (security/quality/compliance).
argument-hint: "[full | security | quality | compliance | diff]"
allowed-tools: Read, Write, Glob, Grep, Bash, Task
---

You are triggering **Phase 4: Code Review** — standalone, without requiring a prior
pipeline run.

## Mode

$ARGUMENTS

## What to do

1. **Parse the mode** from `$ARGUMENTS` (default: `full`):
   - `full` → one `code-review` agent, all 18 areas → writes `review.md` + `fix-plan.md`.
   - `security` | `quality` | `compliance` → one focused agent → writes the focus-specific
     review file (`review-security.md`, `review-quality.md`, `review-compliance.md`).
   - `parallel` → launch **three `code-review` agents simultaneously** (security +
     quality + compliance), then merge the three partial reports into a single
     `review.md` and `fix-plan.md`.
   - `diff` → review only files changed on the current branch (run `git diff
     --name-only origin/main...HEAD` first).

2. **Ask clarifying questions** via AskUserQuestion only if the mode is ambiguous or
   the working directory contains multiple projects.

3. **Invoke the `code-review` agent(s) via the Task tool.** For `parallel`, send
   three Task calls in a single message so they run concurrently. The agent is
   read-only (no Edit tool) — it will NOT change source code.

4. **After the agent(s) return**:
   - If `parallel`: merge the three `review-{focus}.md` files into a single
     `review.md` (dedupe overlapping findings by file:line + confidence).
   - Update or create `pipeline-state.json` so `phases["code-review"]` satisfies
     [`pipeline-state.schema.json`](../pipeline-state.schema.json).
   - Respect the **verdict → status mapping**:
     - 🛑 Block / ⚠️ Request Changes → `status: "blocked"` → suggest `/pipeline-continue`
     - ✅ Approve / ✅ Approve+ → `status: "completed"` → suggest testing phase next.

5. **Surface outputs** — link to `review.md`, `fix-plan.md`, and any focus-specific
   reports — then ask whether to proceed with fixes or move to testing/documentation.

This command works standalone: a prior `spec.md`/`plan.md` is helpful but not required.
