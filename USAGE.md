# How to Use YK Dev Pipeline

This guide shows you how to use the pipeline agents and skills in different environments.

## Table of Contents
1. [Using in Claude Code (CLI) — Agent Mode (v3.0+)](#using-in-claude-code-cli)
2. [Slash Commands](#slash-commands)
3. [Using Agents Standalone](#using-agents-standalone)
4. [Using in Claude.ai Projects](#using-in-claudeai-projects)
5. [Using in Claude.ai Chat](#using-in-claudeai-chat)
6. [Skill Invocation Methods](#skill-invocation-methods)
7. [Common Workflows](#common-workflows)
8. [Pipeline State Schema](#pipeline-state-schema)

---

## Using in Claude Code (CLI)

### Installation

```bash
# Add the marketplace, then install the plugin
/plugin marketplace add git@github.com:ykondrat/ai-tools.git
/plugin install ai-tools@ai-tools
```

After installation, restart Claude Code to apply changes.

To verify installation:
```bash
/plugin list
```

To uninstall:
```bash
/plugin uninstall ai-tools
```

### Invocation in Claude Code

After installation, Claude Code automatically detects the plugin skills. You can invoke them:

#### Skill invocation (main entry)
```
@yk-dev-pipeline
```

This invokes the main router, which auto-detects whether agents are available and hands off to `yk-agent-orchestrator` (v3.0 flow) or runs the pipeline in-place using the phase SKILL.md files (v2.0 flow). Individual phases run as agents in their own context windows when agents are available.

If you want to force the agent flow explicitly (or bypass the router), invoke `@yk-agent-orchestrator` directly — it will stop and hand back if agents aren't available.

#### Pipeline slash commands

The plugin registers five `/pipeline-*` commands for explicit control:

```
/pipeline-start Build me a URL shortener
/pipeline-continue
/pipeline-status
/pipeline-review parallel
/pipeline-reset
```

See the [Slash Commands](#slash-commands) section below for details.

#### Natural Language Trigger
Just describe what you want — the Router detects your intent and routes to the correct phase:

```
"Build me a REST API for a blog"
→ Router detects build-something intent → Phase 1a (Brainstorm)

"Build me an API, skip review and docs"
→ Router extracts selected_phases via Rule 0 → Brainstorm (phases: brainstorm, planning, implementation, testing)

"Fix this bug in the auth module"
→ Router detects fix-something intent → Phase 1b (Investigation)

"Review this code I just wrote"
→ Phase 4 (Code Review) — works standalone

"Write tests for my API"
→ Phase 5 (Testing) — works standalone

"Next step" / "Continue"
→ Router reads pipeline-state.json → routes to next incomplete phase (respects selected_phases)
```

---

## Slash Commands

Five commands ship with the plugin for explicit pipeline control. Each is a markdown file under `commands/` at the repo root.

### `/pipeline-start <description>`

Kick off a fresh pipeline run. The command uses `AskUserQuestion` to let you select entry phase (brainstorm / investigation) and which of the 7 phases to include, then invokes the correct agent via the Task tool.

```
/pipeline-start Build me a REST API for a task manager
/pipeline-start Investigate the flaky payments webhook
```

Resume-safety: refuses to overwrite an existing `pipeline-state.json`. Use `/pipeline-continue` or `/pipeline-reset` instead.

### `/pipeline-continue [note]`

Resume from the current `pipeline-state.json`. The command reads the state, determines the next action (next pending phase, resume in-progress phase, or enter fix loop if a phase is `blocked`), and invokes the corresponding agent. An optional `note` is passed as extra context to the next agent.

```
/pipeline-continue
/pipeline-continue prioritize performance fixes
```

### `/pipeline-status`

Read-only snapshot. Reads `pipeline-state.json`, validates it against the schema, and prints per-phase progress, findings counts, blockers, and a suggested next step. Never invokes agents.

```
/pipeline-status
```

### `/pipeline-review [full | security | quality | compliance | parallel | diff]`

Trigger code review standalone — works on any codebase, no prior pipeline required.

| Mode | Behavior |
|------|----------|
| `full` (default) | One `code-review` agent, all 18 areas → `review.md` + `fix-plan.md` |
| `security` / `quality` / `compliance` | One focused agent → `review-{focus}.md` |
| `parallel` | Three `code-review` agents in parallel (security + quality + compliance), merged into a unified `review.md` |
| `diff` | Review only files changed on the current branch vs `origin/main` |

```
/pipeline-review parallel
/pipeline-review diff
```

### `/pipeline-reset [confirm]`

Archive the current pipeline artifacts (`pipeline-state.json`, `spec.md`, `plan.md`, `review.md`, `fix-plan.md`, `test-report.md`, etc.) to `.pipeline-archive/<timestamp>/` so a fresh run can start. Source code and tests are never touched. Requires explicit confirmation (either the literal `confirm` argument or an `AskUserQuestion` answer).

```
/pipeline-reset
/pipeline-reset confirm
```

---

## Using Agents Standalone

Each pipeline agent can be used independently without the full pipeline:

### Via @-mention
Type `@` in Claude Code and select from the agent list:
```
@brainstorm — Run a brainstorming session for a new feature
@investigation — Investigate a bug or performance issue
@planning — Break a spec into tasks
@implementation — Execute a plan
@code-review — Review code (read-only, 18 areas)
@testing — Write comprehensive tests
@documentation — Generate project docs
```

### Via CLI flag
```bash
claude --agent ai-tools:brainstorm
claude --agent ai-tools:code-review
claude --agent ai-tools:testing
```

### Parallel Code Review
The orchestrator automatically runs 3 code-review agents in parallel:
- **Security focus** — Areas 2, 12, 15, 16, 17
- **Quality focus** — Areas 1, 3, 4, 5, 6, 7, 8
- **Compliance focus** — Areas 9, 10, 11, 13, 14, 18

Results are consolidated into a unified `review.md` with deduplicated findings.

### Agent vs Skill Mode
- **Agents (v3.0)**: Each phase runs in isolated context with scoped tools. Default in Claude Code.
- **Skills (v2.0)**: Phases run in main conversation context. Used as fallback or in Claude.ai.

---

## Using in Claude.ai Projects

### Setup

1. Go to [claude.ai](https://claude.ai) → **Projects** → **Create Project**
2. Click **Add Knowledge** → Upload the `skills/` folder contents
3. Add this to **Project Instructions**:

```markdown
You have access to the YK Dev Pipeline skills in your project knowledge.

Available skills:
- yk-dev-pipeline (router - start here for all requests)
- yk-brainstorm (Phase 1a: Requirements for new features)
- yk-investigation (Phase 1b: Bug fixes, refactoring, performance)
- yk-planning (Phase 2: Task breakdown)
- yk-implementation (Phase 3: Code writing)
- yk-code-review (Phase 4: Review)
- yk-testing (Phase 5: Tests)
- yk-documentation (Phase 6: Docs)

When I request development work, read the appropriate SKILL.md file and follow
its instructions. Start with skills/SKILL.md to understand the pipeline.

Always read reference files when the skill instructs you to.
```

### Invocation in Claude.ai Projects

#### Full Pipeline
```
"Build me a URL shortener with analytics"
```
Claude will:
1. Read `skills/SKILL.md` (router)
2. Start with `skills/brainstorm/SKILL.md`
3. Progress through all 6 phases

#### Jump to Specific Phase
```
"Review my code using the yk-code-review skill"
"Write tests using yk-testing"
"Generate documentation with yk-documentation"
```

---

## Using in Claude.ai Chat (Without Projects)

### Setup

1. Download this repository as ZIP
2. Start a new conversation at [claude.ai](https://claude.ai)
3. Drag the ZIP file into the chat

### Invocation

```
"Read skills/SKILL.md and help me build a task management API"
"Use skills/brainstorm/SKILL.md to explore requirements for user auth"
"Apply skills/code-review/SKILL.md to review my TypeScript code"
```

---

## Skill Invocation Methods

### Trigger Phrases

The pipeline uses a **router-first** architecture. Most requests go through the Router, which detects intent and routes to the correct phase. Phases 4-6 also work as standalone triggers.

#### Router-Handled Triggers

| Intent | Example Phrases | Routes To |
|--------|----------------|-----------|
| **Build something** | "build me a...", "I want to build", "new feature", "new project" | Brainstorm (1a) |
| **Fix/debug/refactor** | "fix this bug", "debug this", "refactor", "performance issue", "tech debt" | Investigation (1b) |
| **Continue pipeline** | "next step", "continue", "go ahead" | Next incomplete phase |
| **Check status** | "what's the status?", "where are we?" | Pipeline state summary |
| **Fix (review)** | "fix the issues", "address the feedback" | Implementation (fix mode) |
| **Fix (tests)** | "fix failing tests", "fix test failures" | Implementation (test-fix mode) |

#### Direct Phase Triggers

| Phase | Direct Triggers | Notes |
|-------|----------------|-------|
| **1a. Brainstorm** | "brainstorm", "brainstorm phase", "let's brainstorm" | Explicit name only |
| **1b. Investigation** | "investigation phase", "investigate this" | Explicit name only |
| **2. Planning** | "planning phase", "create a plan" | Requires spec.md |
| **3. Implementation** | "implementation phase", "execute the plan" | Requires plan.md |
| **4. Code Review** | "review this code", "code review", "check my code" | Works standalone |
| **5. Testing** | "write tests", "add tests", "testing phase" | Works standalone |
| **6. Documentation** | "write docs", "generate docs", "documentation phase" | Works standalone |

### How the Router Decides

The Router applies these rules in order:

0. **Phase selection from natural language.** If the request includes phase exclusion/inclusion language ("skip review and docs", "just brainstorm and plan"), extract `selected_phases` without showing the menu. Phase names are normalized (e.g., "impl" → implementation, "cr" → code-review, "docs" → documentation).
1. **Pipeline state takes priority.** If `pipeline-state.json` exists with an active pipeline, check if the request relates to the current pipeline before starting a new one.
2. **Build-something intent → Brainstorm.** Keywords: "build", "create", "develop", "new feature", "new project", "add a feature", "I want to make".
3. **Fix-something intent → Investigation.** Keywords: "fix", "bug", "broken", "debug", "refactor", "slow", "performance", "tech debt", "optimize", "root cause".
4. **Explicit phase name → that phase directly.** "brainstorm" → Brainstorm, "investigate" → Investigation, "plan" → Planning, etc.
5. **Has artifacts → skip to appropriate phase.** User provides spec → Planning. User provides plan → Implementation. Code exists but no spec/plan → Code-Review, Testing, or Documentation as requested.

### Pipeline State Awareness

Skills automatically check `pipeline-state.json`:

```
"Continue from where we left off"
→ Claude reads pipeline-state.json and resumes

"What's the status?"
→ Claude shows current phase and completed work

"Skip to testing"
→ Claude jumps to Phase 5 but notes skipped phases
```

### Continuation Logic

When you say "next step", "continue", or "go ahead", the Router reads `pipeline-state.json` and routes to the next action. If `selected_phases` is present, phases not in the list are skipped when resolving the next pending phase.

| Current State | Next Action |
|---------------|-------------|
| No pipeline-state.json | Show Phase Selection Menu, then ask: new project (brainstorm) or investigate existing code? |
| brainstorm/investigation: completed, planning: pending | Start Planning |
| planning: completed, implementation: pending | Start Implementation |
| implementation: completed, code-review: pending | Start Code-Review |
| code-review: completed (approved), testing: pending | Start Testing |
| code-review: blocked | Start Implementation (fix mode — reads fix-plan.md) — always, regardless of `selected_phases` |
| testing: completed, documentation: pending | Start Documentation |
| testing: blocked | Start Implementation (test-fix mode — reads fix-test-plan.md) — always, regardless of `selected_phases` |
| documentation: completed | Pipeline complete — summarize and suggest new work |
| Any phase: in-progress | Resume that phase |

**Note:** Fix loops (code-review blocked → implementation, testing blocked → implementation) always activate even if implementation wasn't in the selected phases — these are mandatory recovery loops.

---

## Common Workflows

### Phase Selection

When starting a new pipeline, Claude presents a phase selection menu:

```
Claude: "Which phases do you want to include?"
  [A] Full pipeline (all phases) — recommended
  [B] Quick build (brainstorm → planning → implementation)
  [C] Review & polish (code review → testing → documentation)
  [D] Custom — pick individual phases

You: "B"  →  Runs brainstorm, planning, implementation only
You: "D"  →  Claude asks which phases, e.g., "brainstorm, planning, testing"
```

You can also skip the menu entirely using natural language:

```
"Build me an API, skip review and docs"
→ selected_phases: [brainstorm, planning, implementation, testing]

"Just brainstorm and plan this"
→ selected_phases: [brainstorm, planning]

"Full pipeline"
→ selected_phases: all phases (menu skipped)
```

Phase names are flexible — "impl", "cr", "docs", "bs", "inv", "3", "4" all work.

### Workflow 1: Full Pipeline (New Project)

```bash
# In Claude Code or Claude.ai
You: "Build me a REST API for a blog with posts and comments"

Claude: [Invokes yk-brainstorm]
  → Asks questions about requirements
  → Proposes approaches
  → Generates spec.md + design doc
  → Creates pipeline-state.json

You: "Yes, proceed to planning"

Claude: [Invokes yk-planning]
  → Reads spec.md and design doc
  → Breaks into 12 tasks with dependencies
  → Generates plan.md + archives to docs/plans/

You: "Start implementing"

Claude: [Invokes yk-implementation]
  → Reviews plan critically
  → Detects tech stack
  → Executes tasks in batches (3 at a time)
  → Runs quality gates (tsc, build, lint, tests)
  → Verifies acceptance criteria

You: "Review the code"

Claude: [Invokes yk-code-review]
  → Reviews across 18 areas
  → Generates review.md + fix-plan.md
  → Either approves or blocks with fixes

You: "Write tests"

Claude: [Invokes yk-testing]
  → Writes unit, integration, e2e tests
  → Runs tests, fixes failures
  → If tests keep failing after 2 attempts, asks:
    "Skip failing tests or go back to implementation?"
  → Achieves >80% coverage
  → Generates test-report.md

You: "Generate docs"

Claude: [Invokes yk-documentation]
  → Generates README, API docs, architecture
  → Creates deployment guide
  → Adds JSDoc to exports
```

### Workflow 2: Jump to Specific Phase

```bash
# Review existing code
You: "Review my TypeScript API code"

Claude: [Invokes yk-code-review directly]
  → Reads all code
  → Reviews across 18 areas
  → Generates review.md

# Write tests for existing code
You: "Write comprehensive tests for src/api/"

Claude: [Invokes yk-testing directly]
  → Detects test runner
  → Writes tests
  → Runs and fixes them

# Generate docs for existing project
You: "Generate documentation for this codebase"

Claude: [Invokes yk-documentation directly]
  → Detects project type
  → Generates all docs from actual code
```

### Workflow 3: Iterate on a Phase

```bash
# After code review finds issues
Claude: "Code review blocked. Found 2 critical and 5 major issues."

You: "Fix them"

Claude: [Re-invokes yk-implementation]
  → Reads fix-plan.md
  → Fixes issues in order
  → Re-runs quality gates

You: "Re-review"

Claude: [Re-invokes yk-code-review in diff mode]
  → Reviews only changed code
  → Verifies fixes
  → Updates review.md
```

### Workflow 3b: Test Failure Recovery

```bash
# Tests keep failing after fix attempts
Claude: "3 tests are still failing after fix attempts.
  What would you like to do?"
  1. Go back to implementation
  2. Skip failing tests
  3. Keep trying

# Option 1: Go back to implementation
You: "Go back to implementation"

Claude: [Generates fix-test-plan.md]
  → Documents each failing test, root cause, and suggested fix
  → Sets testing status to "blocked" in pipeline-state.json
  → Hands off to implementation phase

Claude: [Re-invokes yk-implementation]
  → Reads fix-test-plan.md
  → Fixes the underlying code issues
  → Re-runs quality gates

You: "Re-run tests"

Claude: [Re-invokes yk-testing]
  → Re-runs all tests including previously failing ones
  → Generates updated test-report.md

# Option 2: Skip failing tests
You: "Skip them"

Claude: [Marks failing tests as .todo('reason')]
  → Documents skipped tests in test-report.md
  → Continues with coverage report
  → Proceeds to documentation
```

### Workflow 4: Pipeline State Management

```bash
# Check current status
You: "Where are we in the pipeline?"

Claude: "Currently in implementation phase. Completed:
  ✓ Brainstorm (spec.md, design doc)
  ✓ Planning (plan.md + docs/plans/, 15 tasks)
  ⏳ Implementation (9/15 tasks complete)
  ⏸ Code Review (pending)
  ⏸ Testing (pending)
  ⏸ Documentation (pending)"

# Resume after interruption
You: "Continue"

Claude: [Reads pipeline-state.json]
  → Resumes at Task 10 in implementation
  → Picks up where it left off

# Reset and start over
You: "Start over from scratch"

Claude: [Resets pipeline-state.json]
  → Begins at Phase 1 (brainstorm)
```

---

## Advanced Usage

### Customize Phase Behavior

You can override defaults by being explicit:

```bash
# Change coverage target
"Write tests with >90% coverage target"

# Use specific test framework
"Write tests using Jest instead of Vitest"

# Skip quality gates (not recommended)
"Implement without running linters"

# Change batch size
"Implement 5 tasks per batch instead of 3"
```

### Access Reference Materials

Skills automatically load references when needed, but you can request them explicitly:

```bash
"Show me the JS/TS best practices reference"
→ Reads implementation/references/js-ts-best-practices.md

"What are the code review checklists?"
→ Reads code-review/references/review-checklists.md

"Show database patterns for PostgreSQL"
→ Reads implementation/references/databases-sql.md

"Show me Express/Fastify patterns"
→ Reads implementation/references/frameworks-web.md

"Show me React patterns"
→ Reads implementation/references/frameworks-frontend.md

"Show me Vitest and test patterns"
→ Reads testing/references/test-patterns.md
```

### Parallel Phase Execution (Advanced)

Normally phases run sequentially because later phases depend on earlier ones (testing
uses code-review findings, documentation uses test reports). However, for existing
projects where phases are independent, you can request parallel execution:

```bash
"Review the code AND write tests in parallel"
→ Claude spawns two agents:
   - Agent 1: yk-code-review
   - Agent 2: yk-testing
```

> **Caveat**: Only parallelize phases that don't depend on each other's output.
> For example, testing and documentation can run in parallel on existing code,
> but code-review should complete before testing in a fresh pipeline (since
> test failures may stem from review findings).

---

## Pipeline State File

The pipeline creates `pipeline-state.json` at your project root:

```json
{
  "project_name": "blog-api",
  "created_at": "2026-02-13T10:30:00Z",
  "selected_phases": ["brainstorm", "planning", "implementation", "code-review", "testing", "documentation"],
  "current_phase": "implementation",
  "phases": {
    "brainstorm": {
      "status": "completed",
      "completed_at": "2026-02-13T10:45:00Z",
      "outputs": ["spec.md", "docs/plans/2026-02-13-blog-api-design.md"]
    },
    "planning": {
      "status": "completed",
      "completed_at": "2026-02-13T11:00:00Z",
      "outputs": ["plan.md", "docs/plans/2026-02-13-blog-api-plan.md"],
      "task_count": 15,
      "phase_count": 4
    },
    "implementation": {
      "status": "in-progress",
      "started_at": "2026-02-13T11:05:00Z",
      "completed_tasks": [1, 2, 3, 4, 5, 6, 7, 8, 9],
      "current_task": 10,
      "tech_stack": {
        "runtime": "Node.js 20",
        "language": "TypeScript 5.3",
        "framework": "Express",
        "database": "PostgreSQL",
        "orm": "Prisma"
      }
    },
    "code-review": { "status": "pending" },
    "testing": { "status": "pending" },
    "documentation": { "status": "pending" }
  }
}
```

This file helps Claude:
- Remember progress across sessions
- Resume from interruptions
- Skip completed and deselected phases
- Track outputs, decisions, and which phases were selected

---

## Tips for Best Results

1. **Trust the Process**: Let each phase complete before moving to the next
2. **Review Incrementally**: During brainstorm and planning, validate in small chunks
3. **Use Phase Selection**: Choose "Quick build" for prototypes or "Full pipeline" for production code — don't manually skip phases
4. **Use Quality Gates**: Let implementation run all checks after each batch
5. **Fix Review Issues**: Don't proceed to testing until code review approves
6. **Handle Test Failures**: If tests keep failing, choose "go back to implementation" to fix root causes rather than skipping
7. **Be Specific**: The more context you provide, the better the results
8. **Read the Outputs**: Check spec.md, plan.md, review.md - they guide the next phase

---

## Troubleshooting

### "Skill not found"
- **Claude Code**:
  - Run `/plugin list` to verify installation
  - If not listed, reinstall with `/plugin marketplace add git@github.com:ykondrat/ai-tools.git` then `/plugin install ai-tools@ai-tools`
  - Restart Claude Code after installation
- **Claude.ai**: Re-upload the skills folder to your project knowledge

### "Missing spec.md"
- You jumped to planning without brainstorming
- Either provide your own spec.md or run brainstorm first

### "Quality gates failing"
- This is intentional - fix the issues before proceeding
- Don't ask Claude to skip quality gates

### "Pipeline state out of sync"
- Delete `pipeline-state.json` and start fresh
- Or manually edit it to set the correct phase

---

## Examples

### Example 1: E-commerce API
```
You: "Build an e-commerce API with products, cart, and checkout"

[Full pipeline runs]
├─ Brainstorm: Explores payment methods, inventory, user auth
├─ Planning: 23 tasks across 5 phases → plan.md + docs/plans/
├─ Implementation: Builds API with Stripe integration
├─ Code Review: Finds 1 critical security issue, 3 major issues
├─ Implementation (fixes): Resolves all issues
├─ Code Review: Approves
├─ Testing: 127 tests, 94% coverage
└─ Documentation: README, API docs, deployment guide

Output: Production-ready e-commerce API in ~2 hours
```

### Example 2: React Component Library
```
You: "I want to build a React component library with design tokens"

[Brainstorm only]
Claude: Asks about:
  - Target frameworks (React, Vue, Svelte?)
  - Build system (Rollup, Vite, tsup?)
  - Documentation (Storybook?)
  - Theming approach

→ Generates spec.md with decisions
```

### Example 3: Code Review for PR
```
You: "Review this PR using yk-code-review"

[Code review only]
Claude:
  - Reviews all changed files
  - Checks against 18 areas
  - Generates review.md with findings
  - Categorizes by severity
  - Provides fix suggestions

Output: Comprehensive review report in ~5 minutes
```

---

## Pipeline State Schema

`pipeline-state.json` is the single source of truth for pipeline progress. Every agent and orchestrator updates it, and every update must validate against [`pipeline-state.schema.json`](pipeline-state.schema.json) (JSON Schema draft-07).

Key invariants enforced by the schema:

- `brainstorm` and `investigation` are mutually exclusive — never both keys in `phases`.
- `version`, `created_at`, `project_name`, `current_phase`, `phases` are required.
- Code-review verdict → status mapping: `🛑 Block` / `⚠️ Request Changes` → `blocked`; `✅ Approve` / `✅ Approve+` → `completed`.
- Testing may not be `completed` while `test_counts.failing > 0` (agent rule — not schema-enforced, but checked by the agent and orchestrator).

You can validate a state file locally with any JSON Schema draft-07 validator, e.g.:

```bash
npx ajv validate -s pipeline-state.schema.json -d pipeline-state.json
```

---

## Next Steps

- Read [CLAUDE.md](CLAUDE.md) for repository-specific guidance
- Check [examples/](examples/) for example artifacts: pipeline state, plan, review, fix plans, test report
- Review individual SKILL.md files under `skills/` and the agent files under `agents/`
- Read reference materials in `references/` (shared library, deduplicated)
- See [`agents/FRONTMATTER.md`](agents/FRONTMATTER.md) for which YAML fields are authoritative vs advisory

---

**Ready to start?** Just say: `"Let's build something with yk-dev-pipeline"` — or run `/pipeline-start` if you prefer the explicit command flow.
