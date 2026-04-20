<p align="center">
  <img src="https://avatars.githubusercontent.com/u/25372917?s=400&u=cb65675c0d91605c69be4e24dd2f5043b01125cc&v=4" width="120" height="120" style="border-radius: 50%;" alt="Yevhen Kondratyev" />
</p>

# 🏗️ Dev Pipeline for Claude — v3.1 (Agent-First + Slash Commands + Schema)

A comprehensive 6-phase development pipeline that turns Claude into a senior engineering team. From idea to production-ready, tested, documented code — using **independent agents** with isolated context, scoped tools, and per-phase model selection.

```
Brainstorm/Investigation → Planning → Implementation → Code Review → Testing → Documentation
```

## What Is This?

A collection of **Claude Code agents, skills, and slash commands** (structured markdown files with YAML frontmatter) that guide Claude through a complete software development lifecycle for JavaScript/TypeScript projects. Each agent runs in its **own context window** with scoped tools, dedicated model, self-verification gates, and operational guardrails (git safety + resume idempotency).

**Think of it as a senior engineering team, where each team member has their own expertise and tools.**

## What's New in v3.1

- **Two orchestrator skills** — `yk-dev-pipeline` (main router, auto-detects agents) + `yk-agent-orchestrator` (agent-first, explicit v3.0 flow)
- **Five slash commands** — `/pipeline-start`, `/pipeline-continue`, `/pipeline-status`, `/pipeline-review`, `/pipeline-reset`
- **Canonical JSON Schema** — `pipeline-state.schema.json` validates every state update. Explicit verdict → status mapping.
- **Operational guardrails** — Every agent has a git-safety + resume-safety section (no unauthorized commits, no clobbering artifacts, idempotent re-runs)
- **Deduplicated references** — Reference library lives in one place (`references/` at repo root); phase skills and agents both load from the same tree
- **Frontmatter documentation** — `agents/FRONTMATTER.md` documents which YAML fields are authoritative vs advisory

## What v3.0 Established

- **Independent agents** — Each phase runs in its own context window (no cross-phase pollution)
- **Scoped tools** — Code review is read-only, investigation can't edit source
- **Per-phase models** — Opus for complex phases (brainstorm, implementation, review), Sonnet for structured phases
- **Parallel code review** — 3 agents review simultaneously (security, quality, compliance)
- **Self-verification gates** — Every agent validates its output before completing
- **Research-backed improvements** — Mutation testing, property-based tests, hypothesis-evidence matrix, ADR automation
- **Backward compatible** — Phase skills kept for Claude.ai Projects uploads and skill-only installs

## Features

- 🎛️ **Phase Selection** — Full pipeline, quick build, review & polish, or custom selection
- 🧠 **Brainstorm** (opus, blue) — Requirements via questioning + constraint exploration, multi-perspective analysis, completeness scoring
- 🔎 **Investigation** (opus, purple) — Root cause analysis + git bisect, hypothesis-evidence matrix, evidence thresholds
- 📋 **Planning** (sonnet, green) — Task breakdown + DAG dependency analysis, solvability verification, error propagation
- ⚡ **Implementation** (opus, orange) — Strict incremental execution + scaffolding-first, per-task self-verification
- 🔍 **Code Review** (opus, red) — 18-area review + parallel agents, confidence scoring, trajectory analysis, counterfactual analysis
- 🧪 **Testing** (sonnet, yellow) — Test generation + mutation testing, property-based tests, multi-metric quality assessment
- 📝 **Documentation** (sonnet, cyan) — Doc generation + ADR automation, living documentation, command verification

### Bonus: Reference Materials

- **JS/TS Best Practices** — TypeScript patterns, Node.js, modern JS, security, performance, design patterns, algorithms
- **Web Frameworks** — Express, Fastify, Hono, Next.js API routes — routing, middleware, error handling, project setup
- **Frontend Frameworks** — React, Next.js App Router, Vue 3 Composition API — components, hooks, state management, testing
- **SQL Databases** — PostgreSQL + MySQL with Prisma, Drizzle, and raw driver patterns
- **NoSQL Databases** — MongoDB + DynamoDB with Mongoose and AWS SDK patterns
- **Redis** — Caching, sessions, rate limiting, queues, pub/sub
- **Review Checklists** — Deep checklists for all 18 code review areas
- **Test Patterns** — Vitest reference, component testing, MSW mocking, factories, anti-patterns

## Installation

### Claude Code (CLI)

#### Plugin Marketplace

```bash
/plugin marketplace add git@github.com:ykondrat/ai-tools.git
/plugin install ai-tools@ai-tools
```

After installation, restart Claude Code to apply changes.

**Verify installation:**
```bash
/plugin list
```

**Uninstall:**
```bash
/plugin uninstall ai-tools
```

### Claude.ai Projects

1. Go to [claude.ai](https://claude.ai) → **Projects** → **Create Project**
2. Click **Add Knowledge** → upload the `skills/` folder contents
3. Add this to **Project Instructions**:

```
You have access to the YK Dev Pipeline skills in your project knowledge.

Available skills:
- yk-dev-pipeline (main entry — start here for all requests)
- yk-agent-orchestrator (agent-first orchestrator — only usable if Task tool is available)
- yk-brainstorm (Phase 1a: Requirements for new features)
- yk-investigation (Phase 1b: Bug fixes, refactoring, performance)
- yk-planning (Phase 2: Task breakdown)
- yk-implementation (Phase 3: Code writing)
- yk-code-review (Phase 4: Review)
- yk-testing (Phase 5: Tests)
- yk-documentation (Phase 6: Docs)

When I request development work, start at skills/SKILL.md. It auto-detects
whether agents are available and routes accordingly. Always read reference
files when the skill instructs you to (look under references/ — the shared
library at the repo root).
```

### Claude.ai Chat (One-time Use)

1. Download this repository as ZIP
2. Start a new conversation at [claude.ai](https://claude.ai)
3. Drag the ZIP into the chat
4. Say: **"Read skills/SKILL.md and help me build [your project]"**

## Quick Start

📖 **[Read the complete USAGE.md guide](USAGE.md)** for detailed instructions, workflows, and examples.

After installation, just say: **"Build me a REST API for a task management app"** — Claude will start the pipeline automatically. You can also specify which phases to run: **"Build me a REST API, skip review and docs"**.

## Pipeline Flow

```
┌──────────────┐
│ 1a Brainstorm│──┐
│  spec.md     │  │  ┌──────────┐     ┌────────────────┐
│  design doc  │  ├─▶│ Planning │────▶│ Implementation │
└──────────────┘  │  │          │     │                │◀────────┐
┌──────────────┐  │  │ plan.md  │     │  working code  │         |
│1b Investigate│──┘  │ archived │     │                │         |
│  spec.md     │     └──────────┘     └────────┬───────┘         |
│  inv. report │                               │                 |
└──────────────┘                               ▼                 |
┌──────────────┐     ┌────────────────┐     ┌────────────────┐   |
│Documentation │     │    Testing     │     │  Code Review   │   |
│              │     │                │     │                │   |
│ README, docs │◀─┐  │ test-report.md │◀─ ┐ │ review.md      │   |
│ API, deploy  │  │  │ fix-test-plan  │   │ │ fix-plan.md    │   |
└──────────────┘  │  └───────┬────────┘   | └────────┬───────┘   |
                  │          │            |          ▼           |
                  │          │            |  ┌────────────────┐  |
                  │          │            └◀─│- if (blocked)+ │─▶┘
                  │          │               └────────────────┘  │
                  │          ▼                                   │
                  │  ┌────────────────┐                          │
                  └◀─│- if (failing)+ │─────────────────────────▶┘
                     └────────────────┘
```

Each phase:
- Reads the previous phase's output
- Follows its SKILL.md instructions
- Produces artifacts for the next phase
- Updates `pipeline-state.json` for tracking

## Skills Reference

| Phase | Skill | What It Does | Output |
|-------|-------|-------------|--------|
| 1a | [Brainstorm](skills/brainstorm/SKILL.md) | Requirements gathering, approach exploration | `spec.md`, design doc |
| 1b | [Investigation](skills/investigation/SKILL.md) | Debugging, root cause analysis, refactoring assessment | `spec.md`, investigation report |
| 2 | [Planning](skills/planning/SKILL.md) | Task breakdown, dependency graph | `plan.md`, `docs/plans/*-plan.md` |
| 3 | [Implementation](skills/implementation/SKILL.md) | Code writing with quality gates | Working code |
| 4 | [Code Review](skills/code-review/SKILL.md) | 18-area strict review | `review.md`, `fix-plan.md` |
| 5 | [Testing](skills/testing/SKILL.md) | Comprehensive test generation | Tests, `test-report.md`, `fix-test-plan.md` (if blocked) |
| 6 | [Documentation](skills/documentation/SKILL.md) | Full project documentation | README, API docs, etc. |

> **Routing note:** The Router handles all general requests and routes to the correct phase. Phases 1a-3 only trigger on explicit phase-name mentions; Phases 4-6 also work standalone.

## Slash Commands

Installing the plugin registers five `/pipeline-*` commands for explicit pipeline control:

| Command | What it does |
|---------|--------------|
| `/pipeline-start <description>` | Kick off a fresh pipeline run with phase selection |
| `/pipeline-continue [note]` | Resume from `pipeline-state.json` — auto-picks next phase or fix loop |
| `/pipeline-status` | Read-only snapshot of phase progress, findings, blockers |
| `/pipeline-review [full \| security \| quality \| compliance \| parallel \| diff]` | Trigger code review standalone |
| `/pipeline-reset [confirm]` | Archive current artifacts so a fresh run can start |

## State Schema

`pipeline-state.schema.json` (at the repo root) is the canonical JSON Schema (draft-07) for `pipeline-state.json`. Every agent and orchestrator validates its state updates against this schema. Key invariants:

- `brainstorm` and `investigation` are mutually exclusive — never both.
- Code-review verdicts map 1:1 to status: 🛑 Block / ⚠️ Request Changes → `blocked`; ✅ Approve / ✅ Approve+ → `completed`.
- Testing may not be `completed` while `test_counts.failing > 0`.

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
ai-tools/
├── .claude-plugin/
│   ├── marketplace.json                 ← Marketplace catalog (source: ".")
│   └── plugin.json                      ← Plugin manifest
├── agents/                              ← v3.0 phase agents (isolated context)
│   ├── brainstorm.md
│   ├── investigation.md
│   ├── planning.md
│   ├── implementation.md
│   ├── code-review.md
│   ├── testing.md
│   ├── documentation.md
│   └── FRONTMATTER.md
├── skills/                              ← Skill-based fallbacks (v2.0) + orchestrators
│   ├── SKILL.md                         ← yk-dev-pipeline (main router)
│   ├── agent-orchestrator/
│   │   └── SKILL.md                     ← yk-agent-orchestrator (v3.0)
│   ├── brainstorm/SKILL.md              ← Phase 1a
│   ├── investigation/SKILL.md           ← Phase 1b
│   ├── planning/SKILL.md                ← Phase 2
│   ├── implementation/SKILL.md          ← Phase 3
│   ├── code-review/SKILL.md             ← Phase 4
│   ├── testing/SKILL.md                 ← Phase 5
│   ├── documentation/SKILL.md           ← Phase 6
│   └── package.json
├── commands/                            ← /pipeline-* slash commands
│   ├── pipeline-start.md
│   ├── pipeline-continue.md
│   ├── pipeline-status.md
│   ├── pipeline-review.md
│   └── pipeline-reset.md
├── references/                          ← Shared reference library
│   ├── brainstorm/creativity-techniques.md
│   ├── investigation/investigation-patterns.md
│   ├── implementation/                  ← 9 files (clean-code, js-ts, security, databases, frameworks)
│   ├── code-review/review-checklists.md
│   ├── testing/test-patterns.md
│   └── documentation/doc-templates.md
├── examples/
│   ├── pipeline-state.example.json
│   ├── pipeline-state-investigation.example.json
│   ├── plan.example.md
│   ├── review.example.md
│   ├── fix-plan.example.md
│   ├── fix-test-plan.example.md
│   └── test-report.example.md
├── pipeline-state.schema.json           ← JSON Schema draft-07 for pipeline-state.json
├── README.md                            ← You are here
├── USAGE.md
├── CONTRIBUTING.md
├── CLAUDE.md
├── TESTING.md
├── LICENSE
└── .skillrc.json
```

## Customization

These skills are designed to be forked and customized:

- **Add your own conventions** — edit the style/naming sections to match your team
- **Add/remove review areas** — edit `code-review/SKILL.md` and `references/review-checklists.md`
- **Change coverage targets** — edit `testing/SKILL.md`
- **Add frameworks** — extend with Angular, Svelte, NestJS, etc. (React, Vue, Express, Fastify, Hono, Next.js already covered)
- **Change defaults** — swap Vitest for Jest, Prisma for Drizzle, etc.

## How It Works Under the Hood

Claude AI skills are markdown files with structured instructions. When Claude reads a SKILL.md file, it follows the instructions as a detailed playbook. The key principles:

1. **Progressive loading** — Claude reads only the skills it needs for the current phase, not everything at once
2. **Reference on demand** — Deep checklists and best practices are in separate files, loaded when needed
3. **Artifact chaining** — Each phase produces files (spec.md, plan.md, etc.) that the next phase reads. Plans and design docs are also archived in `docs/plans/` with date prefixes
4. **State tracking** — `pipeline-state.json` tracks progress across phases, including which phases are selected
5. **Phase selection** — Users choose which phases to run (full, quick build, review & polish, or custom). Handoff resolution dynamically determines the next phase based on the selection
6. **Human in the loop** — Claude suggests the next phase, but waits for user confirmation
7. **Router-first routing** — The pipeline router handles all general requests ("build me...", "fix this bug", "next step") and routes to the correct phase using 6 intent detection rules (0-5) and pipeline state. Individual phases only trigger on explicit phase-name mentions.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Areas where contributions are welcome:
- Additional reference materials (Angular, Svelte, NestJS, tRPC patterns)
- Language-specific adaptations (Python, Go, Rust)
- Infrastructure patterns (Docker, Kubernetes, Terraform, CI/CD)
- Bug fixes and improvements to existing skills

## Publishing to Marketplaces

This skill collection is structured as a **Claude Code plugin marketplace** repo. It can be installed via:

```bash
/plugin marketplace add git@github.com:ykondrat/ai-tools.git
/plugin install ai-tools@ai-tools
```

The `.claude-plugin/marketplace.json` at the repo root defines the marketplace catalog, and `.claude-plugin/plugin.json` (also at the root) provides the plugin manifest. The repo uses a flat single-plugin layout — `source: "."` in the marketplace points at the repo root itself. Agents, skills, and commands are auto-discovered from `agents/`, `skills/`, and `commands/` at the root.

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
