# 🏗️ Dev Pipeline Skills for Claude

A comprehensive 6-phase development pipeline that turns Claude into a senior engineering team. From idea to production-ready, tested, documented code — using structured AI skills.

```
Brainstorm → Planning → Implementation → Code Review → Testing → Documentation
```

## What Is This?

A collection of **Claude AI skills** (structured instruction files) that guide Claude through a complete software development lifecycle for JavaScript/TypeScript projects. Each skill contains detailed processes, checklists, best practices, and reference materials that Claude reads and follows.

**Think of it as a senior engineering team's playbook, optimized for AI.**

## Features

- 🧠 **Brainstorm** — Deep-dive requirements gathering with conversational questioning
- 📋 **Planning** — Task breakdown with dependencies, acceptance criteria, file structure, and archived plans in `docs/plans/`
- ⚡ **Implementation** — Batch execution with quality gates, tech stack detection, and self-review
- 🔍 **Code Review** — 18-area strict review with 4-level severity and actionable fix plans
- 🧪 **Testing** — 5 test types, >80% coverage target, auto-detect runner, generate-run-fix loop, failure recovery (skip or loop back to implementation)
- 📝 **Documentation** — 7 doc types generated from actual code (README, API, architecture, deployment, etc.)

### Bonus: Reference Materials

- **JS/TS Best Practices** (52KB) — TypeScript patterns, Node.js, modern JS, security, performance, design patterns, algorithms
- **SQL Databases** (33KB) — PostgreSQL + MySQL with Prisma, Drizzle, and raw driver patterns
- **NoSQL Databases** (28KB) — MongoDB + DynamoDB with Mongoose and AWS SDK patterns
- **Redis** (40KB) — Caching, sessions, rate limiting, queues, pub/sub
- **Review Checklists** (16KB) — Deep checklists for all 18 code review areas

## Installation

### Claude Code (CLI)

#### Plugin Marketplace

```bash
/plugin marketplace add git@github.com:ykondrat/yk-dev-pipeline-skills.git
/plugin install yk-dev-pipeline@yk-dev-pipeline-skills
```

After installation, restart Claude Code to apply changes.

**Verify installation:**
```bash
/plugin list
```

**Uninstall:**
```bash
/plugin uninstall yk-dev-pipeline
```

### Claude.ai Projects

1. Go to [claude.ai](https://claude.ai) → **Projects** → **Create Project**
2. Click **Add Knowledge** → upload the `plugins/yk-dev-pipeline/skills/` folder contents
3. Add this to **Project Instructions**:

```
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

### Claude.ai Chat (One-time Use)

1. Download this repository as ZIP
2. Start a new conversation at [claude.ai](https://claude.ai)
3. Drag the ZIP into the chat
4. Say: **"Read plugins/yk-dev-pipeline/skills/SKILL.md and help me build [your project]"**

## Quick Start

📖 **[Read the complete USAGE.md guide](USAGE.md)** for detailed instructions, workflows, and examples.

After installation, just say: **"Build me a REST API for a task management app"** — Claude will start the pipeline automatically.

## Pipeline Flow

```
┌─────────────┐     ┌──────────┐     ┌────────────────┐
│  Brainstorm  │────▶│ Planning │────▶│ Implementation │
│              │     │          │     │                │
│  spec.md     │     │ plan.md  │     │  working code  │
│  design doc  │     │ archived │     │                │
└─────────────┘     └──────────┘     └───────┬────────┘
                                              │
                                              ▼
┌─────────────┐     ┌────────────────┐     ┌────────────────┐
│Documentation │◀────│    Testing     │◀────│  Code Review   │
│              │     │                │     │                │
│ README, docs │     │ test-report.md │     │ review.md      │
│ API, deploy  │     │ fix-test-plan  │     │ fix-plan.md    │
└─────────────┘     └───────┬────────┘     └────────┬───────┘
                          │                    │
                          │           (if blocked, loops back
                          │            to Implementation)
                          │
                 (if tests keep failing,
                  user chooses: skip or
                  loop back to Implementation)
```

Each phase:
- Reads the previous phase's output
- Follows its SKILL.md instructions
- Produces artifacts for the next phase
- Updates `pipeline-state.json` for tracking

## Skills Reference

| Phase | Skill | What It Does | Output |
|-------|-------|-------------|--------|
| 1 | [Brainstorm](plugins/yk-dev-pipeline/skills/brainstorm/SKILL.md) | Requirements gathering, approach exploration | `spec.md`, design doc |
| 2 | [Planning](plugins/yk-dev-pipeline/skills/planning/SKILL.md) | Task breakdown, dependency graph | `plan.md`, `docs/plans/*-plan.md` |
| 3 | [Implementation](plugins/yk-dev-pipeline/skills/implementation/SKILL.md) | Code writing with quality gates | Working code |
| 4 | [Code Review](plugins/yk-dev-pipeline/skills/code-review/SKILL.md) | 18-area strict review | `review.md`, `fix-plan.md` |
| 5 | [Testing](plugins/yk-dev-pipeline/skills/testing/SKILL.md) | Comprehensive test generation | Tests, `test-report.md`, `fix-test-plan.md` (if blocked) |
| 6 | [Documentation](plugins/yk-dev-pipeline/skills/documentation/SKILL.md) | Full project documentation | README, API docs, etc. |

## Example Usage

### Full Pipeline
```
You: "Build me a URL shortener with analytics"
Claude: [Brainstorm] Asks about requirements, proposes approaches → spec.md
Claude: [Planning] Breaks into 15 tasks with dependencies → plan.md + docs/plans/
Claude: [Implementation] Implements in batches, quality gates after each
Claude: [Code Review] Reviews across 18 areas, finds 2 major issues → fix-plan.md
Claude: [Implementation] Fixes the 2 issues
Claude: [Code Review] Re-reviews (diff mode) → approved
Claude: [Testing] Writes 47 tests, 2 keep failing → asks: skip or fix?
You: "Go back and fix"
Claude: [Testing] Generates fix-test-plan.md with actionable fixes
Claude: [Implementation] Reads fix-test-plan.md, fixes the 2 issues
Claude: [Testing] Re-runs → 47 tests passing, 89% coverage → test-report.md
Claude: [Documentation] Generates README, API docs, deployment guide
```

### Jump to Any Phase
```
You: "Review this TypeScript project" [uploads code]
Claude: [Code Review] Full review across 18 areas

You: "Write tests for my API" [uploads code]
Claude: [Testing] Detects Vitest, writes tests, runs them, fixes failures

You: "Generate docs for this project" [uploads code]
Claude: [Documentation] README, API docs, architecture, deployment guide
```

## Repository Structure

```
yk-dev-pipeline-skills/
├── .claude-plugin/
│   └── marketplace.json                 ← Marketplace catalog
├── plugins/
│   └── yk-dev-pipeline/
│       ├── .claude-plugin/
│       │   └── plugin.json              ← Plugin manifest
│       └── skills/
│           ├── SKILL.md                 ← Pipeline router
│           ├── brainstorm/
│           │   └── SKILL.md             ← Phase 1
│           ├── planning/
│           │   └── SKILL.md             ← Phase 2
│           ├── implementation/
│           │   ├── SKILL.md             ← Phase 3
│           │   └── references/
│           │       ├── js-ts-best-practices.md
│           │       ├── databases-sql.md
│           │       ├── databases-nosql.md
│           │       └── databases-redis.md
│           ├── code-review/
│           │   ├── SKILL.md             ← Phase 4
│           │   └── references/
│           │       └── review-checklists.md
│           ├── testing/
│           │   ├── SKILL.md             ← Phase 5
│           │   └── references/
│           │       └── test-patterns.md
│           └── documentation/
│               ├── SKILL.md             ← Phase 6
│               └── references/
│                   └── doc-templates.md
├── examples/
│   └── pipeline-state.example.json
├── README.md                            ← You are here
├── USAGE.md
├── CONTRIBUTING.md
├── LICENSE
└── .skillrc.json
```

## Customization

These skills are designed to be forked and customized:

- **Add your own conventions** — edit the style/naming sections to match your team
- **Add/remove review areas** — edit `code-review/SKILL.md` and `references/review-checklists.md`
- **Change coverage targets** — edit `testing/SKILL.md`
- **Add frameworks** — extend the best practices reference with React, Next.js, etc.
- **Change defaults** — swap Vitest for Jest, Prisma for Drizzle, etc.

## How It Works Under the Hood

Claude AI skills are markdown files with structured instructions. When Claude reads a SKILL.md file, it follows the instructions as a detailed playbook. The key principles:

1. **Progressive loading** — Claude reads only the skills it needs for the current phase, not everything at once
2. **Reference on demand** — Deep checklists and best practices are in separate files, loaded when needed
3. **Artifact chaining** — Each phase produces files (spec.md, plan.md, etc.) that the next phase reads. Plans and design docs are also archived in `docs/plans/` with date prefixes
4. **State tracking** — `pipeline-state.json` tracks progress across phases
5. **Human in the loop** — Claude suggests the next phase, but waits for user confirmation

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Areas where contributions are welcome:
- Additional reference materials (React, Vue, Angular, Next.js patterns)
- New review checklist areas
- Additional test patterns
- Language-specific adaptations (Python, Go, Rust)
- Bug fixes and improvements to existing skills

## Publishing to Marketplaces

This skill collection is structured as a **Claude Code plugin marketplace** repo. It can be installed via:

```bash
/plugin marketplace add git@github.com:ykondrat/yk-dev-pipeline-skills.git
/plugin install yk-dev-pipeline@yk-dev-pipeline-skills
```

The `.claude-plugin/marketplace.json` at the repo root defines the marketplace catalog, and `plugins/yk-dev-pipeline/.claude-plugin/plugin.json` provides the plugin manifest. Skills are auto-discovered from the `skills/` directory within the plugin.

## Security

⚠️ **Important**: Always review skills before installation, especially from untrusted sources.

This skill collection:
- Contains only markdown instruction files (no executable code)
- Has no external dependencies
- Does not make network requests
- Does not access credentials or sensitive data

## License

MIT — see [LICENSE](LICENSE) for details.

---

**Built with ❤️ for developers who want AI-assisted development done right.**
