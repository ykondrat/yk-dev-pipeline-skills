---
name: yk-investigation
description: >
  Systematic investigation skill for bugs, refactoring, performance issues, and tech debt
  in JavaScript/TypeScript projects. Reproduces issues, traces root causes, documents
  evidence, and proposes fix strategies.
  DIRECT TRIGGER: "investigation phase", "investigate this", "run an investigation",
  "start an investigation", "investigate the issue", "investigate the problem",
  "do an investigation", "investigation mode", "deep investigation".
  ROUTED FROM: Pipeline router when user intent is to fix, debug, refactor, or optimize.
  DO NOT USE FOR: General "fix this bug" or "debug this" requests — those go through
  the pipeline router which handles intent detection and pipeline state.
  This is an alternative first step in the dev-pipeline skill chain.
metadata:
  recommended_model: opus
---

> **⚠️ Legacy skill.** The authoritative Phase 1b implementation is the **`investigation` agent** at [`../../agents/investigation.md`](../../agents/investigation.md). This skill is kept for backward compatibility (Claude.ai Projects uploads and skill-only Claude Code installs). Edits here may drift from the agent version — see [`docs/review-2026-04-17.md`](../../../../docs/review-2026-04-17.md) for the consolidation plan.

# Investigation — Dev Pipeline Step 1b

Systematic investigation. Deeply understand the problem, trace root causes with evidence,
and produce a validated fix strategy — presented in small pieces, confirmed as you go.

## Pipeline Context

This skill is part of a 6-step development pipeline. Investigation is an alternative
to Brainstorm as the first phase — use it when the user needs to fix, refactor, or
optimize existing code rather than build something new.

```
[1a. brainstorm] ─┐
                  ├→ planning → implementation → code-review → testing → documentation
[1b. investigation]┘
       ↑ you are here
```

Each step produces artifacts the next step consumes. After investigation completes, use
the Handoff Resolution algorithm from the Router to determine the next phase.

Pipeline state is tracked in `pipeline-state.json` inside the project directory.

---

## References

When performing root cause analysis (Step 4), you MUST use the Read tool to load the
investigation patterns reference:

- `../../references/investigation/investigation-patterns.md` — debugging strategies, common root cause categories, refactoring analysis patterns, performance investigation techniques, and evidence documentation formats

---

## The Process

### Step 1: Understand the Codebase & Context

Before asking a single question, do your homework.

- Look at the current project directory (files, structure, README, package.json, tsconfig)
- Read recent git commits and history for the affected area
- Examine existing tests, CI configuration, and error logs if available
- Read the code in the area the user is pointing to
- Check for existing docs, specs, or design files
- Look at dependencies and their versions
- Check `pipeline-state.json` if it exists

**Show the user you did your homework.** Don't ask "what framework are you using?" if
you can see a `next.config.js`. Don't ask "where is the bug?" if the user already pointed
to a file. Summarize what you found:

> "I've looked at the codebase. Here's what I see: [summary of project structure,
> tech stack, affected area, recent changes]. Let me ask some questions to understand
> the problem better."

### Step 2: Web Research

After understanding the local context, offer the user a web research step. Use the
`AskUserQuestion` tool:

- **"Yes — research before we continue" (Recommended)** — Claude searches the web for
  known issues, solutions, and relevant context before proceeding.
- **"Skip — let's investigate locally"** — Jump straight to reproduction and analysis.

If the user opts in, conduct targeted web research using `WebSearch` and `WebFetch`.
Formulate queries based on what you learned in Step 1 — use actual library names, error
messages, and version numbers from the project.

**Search categories** (pick what's relevant):

- **Known issues with dependencies** — "[library] [version] known issues", "[library]
  breaking changes [version]". Check if this is a documented problem.
- **Error-specific searches** — Search for exact error messages, stack trace patterns,
  or error codes. Quote the error message for exact matches.
- **Community solutions** — "[error/symptom] [framework] solution", "[library] [problem
  description]". Look for Stack Overflow answers, GitHub issues, blog posts.
- **Changelogs & migration guides** — "[library] changelog [version]", "[library]
  migration guide [old version] to [new version]". Check for breaking changes.
- **Security advisories** — "[library] CVE", "[library] security vulnerability". Only
  if the issue might be security-related.

**How many searches?** Aim for 3-6 targeted `WebSearch` queries. Use `WebFetch` to read
2-3 of the most relevant results in depth. Don't spray dozens of queries — quality over
quantity.

**Context efficiency:** Consider launching a sub-agent for web research. Have it return
a concise summary (<200 words) of findings rather than keeping raw search results in the
main conversation context.

**Present findings to the user:**
- Concise bullet-point summary of key findings
- Highlight any known issues or solutions that match the observed problem
- Note relevant changelogs or breaking changes in project dependencies
- Flag potential red herrings vs likely matches
- Ask: "Does any of this look related to your issue? Want me to dig deeper on anything?"

Incorporate research findings into all subsequent steps — they can narrow the
investigation, suggest hypotheses, or confirm/rule out causes.

---

### Step 3: Reproduce & Characterize the Problem

**One question per message. No exceptions.** If a topic needs more exploration, break it
into multiple questions across multiple messages.

Prefer multiple-choice questions when possible (use the `AskUserQuestion` tool for discrete
choices). Open-ended questions are fine when the answer space is too broad for options.

Adapt the question flow based on what you already know from Step 1 and from previous
answers. Skip questions that are already answered by the project context.

**Question categories to cover** (in roughly this order, but adapt as needed):

**Problem Description**
- What's the symptom? What do you see vs what do you expect?
- When did this start? Was there a recent change (deploy, dependency update, config change)?
- How frequently does it occur? (always, intermittent, under load, specific conditions)

**Reproduction**
- Can you reproduce it? What are the exact steps?
- Does it happen in all environments (local, staging, production)?
- Any specific input or data that triggers it?

**Impact & Urgency**
- What's the blast radius? (single user, all users, data integrity, security)
- Is there a workaround in place?
- What's the priority? (blocking release, degraded experience, cosmetic)

**Prior Investigation**
- What has already been tried?
- Any error messages, stack traces, or logs available?
- Has anyone else looked at this?

**For Refactoring/Tech Debt (instead of bug-specific questions):**
- What's the pain point with the current code?
- What triggered the decision to refactor now?
- What does "better" look like? (readability, performance, testability, extensibility)
- Are there constraints? (backwards compatibility, API contracts, data migration)

**For Performance Issues:**
- What's slow? (page load, API response, build time, test suite)
- What are the numbers? (current vs expected latency/throughput)
- When did it become noticeable?
- Is it CPU-bound, memory-bound, I/O-bound, or network-bound?

**Not every investigation needs every category.** A clear "this function throws an error"
needs fewer questions than "the app feels slow sometimes." Read the room.

**Attempt reproduction** if possible. If the user described steps to reproduce:
- Try to reproduce the issue in the codebase
- Run the failing test or command
- Document what you observe

### Step 4: Investigate Root Cause

**STOP — use the Read tool now** to load `../../references/investigation/investigation-patterns.md`
before proceeding. This contains the debugging strategies and root cause patterns you need.

Conduct a systematic investigation:

- **Trace code paths** — Follow the execution path from entry point to failure point.
  Document each step with `file:line` references. For large codebases, consider
  launching a sub-agent for deep exploration — have it return only relevant `file:line`
  references rather than full file contents.
- **Check data flow** — Track how data transforms through the system. Look for where
  values become unexpected.
- **Analyze code smells** — For refactoring: identify coupling, complexity, duplication,
  naming issues, missing abstractions.
- **Profile performance** — For perf issues: identify hot paths, N+1 queries, memory
  leaks, unnecessary re-renders, blocking operations.
- **Review git history** — Check `git log` and `git blame` for the affected area.
  When did the problem code get introduced? What was the intent?
- **Check dependencies** — Are there known issues with dependency versions? Check
  changelogs for breaking changes.

**Document evidence as you go.** Every finding should have:
- File and line reference (`src/api/routes.ts:42`)
- What you found
- Why it matters

**Generate multiple hypotheses.** Don't anchor on the first theory. Consider at least
2-3 possible root causes before settling on one. For each hypothesis:
- Evidence supporting it
- Evidence against it
- How to confirm or rule it out

### Step 5: Present Findings & Propose Fix Strategy

Present findings incrementally — **200-300 word sections**, confirmed as you go.

**Section 1: Root Cause**
- State the root cause clearly with evidence
- Show the code path that leads to the problem
- Explain the mechanism (why does this cause the symptom?)

**Section 2: Affected Areas**
- List all files/components affected
- Note ripple effects and dependencies
- Identify risk areas (what could break if we change this?)

**Section 3: Fix Approaches**
Propose 2-3 fix approaches with trade-offs:

- **Lead with your recommendation** and explain why
- For each approach:
  - What changes are needed
  - Pros and cons
  - Effort estimate (small/medium/large)
  - Risk level (low/medium/high)
- Let the user pick, then note the decision AND the reasoning

**Section 4: Risk Assessment**
- What could go wrong with the fix?
- What regression tests are needed?
- Are there data migration concerns?
- Does this need a feature flag or phased rollout?

After each section, ask: "Does this analysis look right so far?"

### Step 6: Generate Output

Once the analysis is validated section by section, generate three artifacts:

#### Investigation Report

Write the full investigation report to:
```
{project-dir}/docs/plans/YYYY-MM-DD-{topic}-investigation.md
```

Use the investigation report template (see `investigation/docs/plans/YYYY-MM-DD-{topic}-investigation.md`).
This is the comprehensive document containing all evidence, analysis, and recommendations.

#### Spec Summary

Also generate a concise summary at:
```
{project-dir}/spec.md
```

This is a streamlined reference for the planning skill to consume. Use this adapted
template (omit irrelevant sections):

```markdown
# {Project/Issue Name} — Specification

## Overview
- **One-liner**: {what needs to be fixed/refactored/optimized}
- **Problem**: {root cause summary}
- **Type**: Bug fix | Refactoring | Performance optimization | Tech debt | Security fix
- **Status**: Extending existing codebase

## Root Cause
{Concise root cause with key evidence — file:line references}

## Affected Areas
| File | Impact | Change Needed |
|------|--------|---------------|
| {file} | {what's wrong} | {what to change} |

## Proposed Changes
### {Change Area 1}
- **Description**: {what to change}
- **Behavior**: {expected before/after}

### {Change Area 2}
...

## Fix Scenarios
- As a {user/developer}, I want {the fix} so that {the symptom is resolved}
- As a {user/developer}, I want {the fix} so that {regression is prevented}

## Out of Scope (Future)
- {related improvements deferred to later}

## Tech Stack
- **Runtime**: {detected from project}
- **Language**: {detected}
- **Framework**: {detected}
- **Key Dependencies**: {relevant to the fix}

## Risk Assessment
{What could go wrong, migration concerns, rollout strategy}

## Testing Strategy
- **Verification tests**: {tests that prove the fix works}
- **Regression tests**: {tests that ensure nothing else breaks}

## Constraints
{Backwards compatibility, API contracts, data integrity}

## Open Questions
{Anything unresolved for planning to address}

## Investigation Reference
See [full investigation report](docs/plans/YYYY-MM-DD-{topic}-investigation.md) for
detailed evidence, code traces, and alternative approaches considered.
```

#### Pipeline State

Create or update `{project-dir}/pipeline-state.json`. If `selected_phases` was already
set by the Router, preserve it. Otherwise, default to all phases:

```json
{
  "project_name": "{project-name}",
  "created_at": "{ISO timestamp}",
  "selected_phases": ["investigation", "planning", "implementation", "code-review", "testing", "documentation"],
  "current_phase": "investigation",
  "phases": {
    "investigation": {
      "status": "completed",
      "completed_at": "{ISO timestamp}",
      "issue_type": "{bug | refactoring | performance | tech-debt | security}",
      "root_cause": "{one-line root cause summary}",
      "outputs": [
        "spec.md",
        "docs/plans/YYYY-MM-DD-{topic}-investigation.md"
      ]
    },
    "planning": { "status": "pending" },
    "implementation": { "status": "pending" },
    "code-review": { "status": "pending" },
    "testing": { "status": "pending" },
    "documentation": { "status": "pending" }
  }
}
```

**Note:** When investigation is Phase 1, there is no `brainstorm` key in the phases —
they are alternatives, not sequential steps.

### Step 7: Git & Handoff

**Git (optional):** After generating files, offer to commit the investigation report:
> "Want me to commit the investigation report and spec to git?"

If yes, commit with a message like: `docs: add {topic} investigation report`

If no repo exists or user declines, move on — don't push it.

**Handoff:** Resolve the next phase using the Handoff Resolution algorithm from the
Router skill: read `pipeline-state.json`, find the next phase in `selected_phases`
after "investigation" (default: planning). Present:
> "The investigation is complete! The next step is **{resolved_next_phase}**."
If no more selected phases remain:
> "Investigation complete! That completes the selected pipeline scope."

If the investigation involved extensive research or deep codebase exploration, suggest a session split:
> "If this conversation is getting long, you can start a new session and say 'continue'
> — the spec and investigation report are on disk, so the next phase will pick up cleanly."

Otherwise, ask if the user wants to proceed to the resolved next phase.

---

## Key Principles

- **Evidence over assumptions** — Every conclusion must have supporting evidence with
  file:line references. "I think" is not evidence.
- **One question at a time** — Don't overwhelm with multiple questions per message.
- **Multiple hypotheses** — Consider at least 2-3 possible causes before committing to
  one. Actively try to disprove your leading theory.
- **Reproduce first** — If the issue can be reproduced, do it before theorizing. Seeing
  the actual failure is worth more than reading code.
- **Trace, don't guess** — Follow the actual execution path. Read the actual code. Don't
  assume what a function does based on its name.
- **Document everything** — Every finding, every dead end, every hypothesis. The
  investigation report should let someone else pick up where you left off.
- **Check context first** — Never ask questions you could answer by reading existing
  files, git history, or error logs.
- **Scope discipline** — Investigate the reported problem. If you find other issues along
  the way, note them but don't chase them. Stay focused.
- **Incremental validation** — Present findings in 200-300 word sections, validate each
  before moving on.
- **Lead with recommendations** — When proposing fixes, give a clear recommendation with
  reasoning, not "it depends."
