# How to Use YK Dev Pipeline Skills

This guide shows you how to use the skills in different environments and scenarios.

## Table of Contents
1. [Using in Claude Code (CLI)](#using-in-claude-code-cli)
2. [Using in Claude.ai Projects](#using-in-claudeai-projects)
3. [Using in Claude.ai Chat](#using-in-claudeai-chat)
4. [Skill Invocation Methods](#skill-invocation-methods)
5. [Common Workflows](#common-workflows)

---

## Using in Claude Code (CLI)

### Installation

#### Method 1: Using `/plugin` Command (Recommended)

```bash
# In Claude Code CLI
/plugin install ykondrat/yk-dev-pipeline-skills

# Or install from a specific branch
/plugin install ykondrat/yk-dev-pipeline-skills@main

# Or install from local directory
/plugin install /path/to/dev-pipeline-skills
```

After installation, restart Claude Code to apply changes.

To verify installation:
```bash
/plugin list
```

To uninstall:
```bash
/plugin uninstall yk-dev-pipeline-skills
```

#### Method 2: Manual Installation

```bash
# Option 1: Install globally (for all projects)
cp -r skills ~/.claude/skills/yk-dev-pipeline

# Option 2: Install per-project
cp -r skills /path/to/your/project/.claude/skills/yk-dev-pipeline
```

### Invocation in Claude Code

Claude Code automatically detects skills in `.claude/skills/`. You can invoke them:

#### Method 1: Direct Invocation via Slash Command
```bash
# In your terminal or Claude Code chat
/yk-dev-pipeline

# Or invoke specific phases
/yk-brainstorm
/yk-planning
/yk-implementation
/yk-code-review
/yk-testing
/yk-documentation
```

#### Method 2: Natural Language Trigger
Just describe what you want and Claude will automatically detect and use the appropriate skill:

```
"Let's brainstorm a new feature for user authentication"
→ Claude automatically invokes yk-brainstorm

"Review this code I just wrote"
→ Claude automatically invokes yk-code-review

"Write tests for my API"
→ Claude automatically invokes yk-testing
```

#### Method 3: Explicit Request
```
"Use the yk-dev-pipeline skill to build a REST API"
"Run the yk-brainstorm skill for this new feature"
"Execute yk-code-review on the current codebase"
```

---

## Using in Claude.ai Projects

### Setup

1. Go to [claude.ai](https://claude.ai) → **Projects** → **Create Project**
2. Click **Add Knowledge** → Upload the entire `skills/` folder
3. Add this to **Project Instructions**:

```markdown
You have access to the YK Dev Pipeline skills in your project knowledge.

Available skills:
- yk-dev-pipeline (router - start here)
- yk-brainstorm (Phase 1: Requirements)
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

### Automatic Triggers (Natural Language)

Each skill has trigger phrases in its description. Claude automatically detects:

| Skill | Trigger Phrases |
|-------|----------------|
| **yk-brainstorm** | "let's brainstorm", "new project", "I want to build", "help me plan" |
| **yk-planning** | "plan this", "create a plan", "break this down", "next step" (after brainstorm) |
| **yk-implementation** | "implement this", "start coding", "build it", "execute the plan" |
| **yk-code-review** | "review this code", "code review", "check my code" |
| **yk-testing** | "write tests", "testing phase", "add tests" |
| **yk-documentation** | "write docs", "generate docs", "documentation phase" |

### Explicit Skill References

```
"Use the yk-brainstorm skill to explore this idea"
"Run yk-code-review on src/api/users.ts"
"Execute yk-testing with >90% coverage target"
```

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

---

## Common Workflows

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
  → Generates plan.md

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

### Workflow 4: Pipeline State Management

```bash
# Check current status
You: "Where are we in the pipeline?"

Claude: "Currently in implementation phase. Completed:
  ✓ Brainstorm (spec.md, design doc)
  ✓ Planning (plan.md, 15 tasks)
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
→ Reads skills/implementation/references/js-ts-best-practices.md

"What are the code review checklists?"
→ Reads skills/code-review/references/review-checklists.md

"Show database patterns for PostgreSQL"
→ Reads skills/implementation/references/databases-sql.md
```

### Parallel Phase Execution (Advanced)

Normally phases run sequentially, but for existing projects you can run them in parallel:

```bash
"Review the code AND write tests in parallel"
→ Claude spawns two agents:
   - Agent 1: yk-code-review
   - Agent 2: yk-testing
```

---

## Pipeline State File

The pipeline creates `pipeline-state.json` at your project root:

```json
{
  "project_name": "blog-api",
  "created_at": "2026-02-13T10:30:00Z",
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
      "outputs": ["plan.md"],
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
- Skip completed phases
- Track outputs and decisions

---

## Tips for Best Results

1. **Trust the Process**: Let each phase complete before moving to the next
2. **Review Incrementally**: During brainstorm and planning, validate in small chunks
3. **Don't Skip Phases**: Each phase builds on the previous one's outputs
4. **Use Quality Gates**: Let implementation run all checks after each batch
5. **Fix Review Issues**: Don't proceed to testing until code review approves
6. **Be Specific**: The more context you provide, the better the results
7. **Read the Outputs**: Check spec.md, plan.md, review.md - they guide the next phase

---

## Troubleshooting

### "Skill not found"
- **Claude Code**:
  - Run `/plugin list` to verify installation
  - If not listed, reinstall with `/plugin install ykondrat/yk-dev-pipeline-skills`
  - Restart Claude Code after installation
  - Alternatively, manually check `~/.claude/skills/yk-dev-pipeline/` exists
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
├─ Planning: 23 tasks across 5 phases
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

## Next Steps

- Read [CLAUDE.md](CLAUDE.md) for repository-specific guidance
- Check [examples/pipeline-state.example.json](examples/pipeline-state.example.json) for state file structure
- Review individual SKILL.md files to understand each phase
- Read reference materials in `skills/*/references/` for deep dives

---

**Ready to start?** Just say: `"Let's build something with yk-dev-pipeline"`
