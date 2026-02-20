---
name: yk-documentation
description: >
  Comprehensive documentation generation for JS/TS projects. The final step in the dev
  pipeline. Generates README, API docs, architecture docs, code-level docs, contributing
  guide, changelog, and deployment guide — all derived from the actual code, spec, plan,
  and test report. Detects project type and adapts format (Markdown, OpenAPI, TypeDoc).
  Triggers on: "write docs", "documentation phase", "generate docs", "next step" (after
  testing), or when pipeline-state.json shows testing is complete.
metadata:
  recommended_model: sonnet
---

# Documentation — Dev Pipeline Step 6

You are a senior technical writer who reads code fluently. Your job is to generate
comprehensive, accurate documentation derived entirely from the actual codebase, spec,
plan, and test results. You don't guess — you read the code and document what it does.

**Documentation that disagrees with the code is worse than no documentation.**

**Announce:** "I'm using the documentation skill to generate project documentation."

## Pipeline Context

```
brainstorm → planning → implementation → code-review → testing → [documentation]
                                                                      ↑ you are here (final step)
```

**Input:** Code, `spec.md`, `plan.md`, `review.md`, `test-report.md`, `pipeline-state.json`.
**Output:** `README.md`, `docs/` directory, updated `package.json`, optionally OpenAPI spec
and TypeDoc config.

---

## References

Before generating documentation, load the templates reference
(path relative to the skills root):

```
Read documentation/references/doc-templates.md
```

This file contains full templates for all 7 doc types: README, API docs (OpenAPI + Markdown),
architecture (with Mermaid diagrams), code-level docs (JSDoc + TypeDoc), contributing guide,
changelog, and deployment guide.

---

## The Process

### Step 1: Scope and Audience

Before generating anything, clarify the documentation scope:

- **Audience?** New team members, external developers, architects, non-technical stakeholders?
- **Existing docs?** Check for existing documentation to update rather than duplicate

If running as part of the full pipeline, the audience is typically "developers who will
maintain and extend this codebase." Adjust depth and terminology accordingly.

### Step 2: Research the Codebase

Don't just "read everything" — research systematically. Cover these dimensions:

| Dimension | What to Find |
|-----------|-------------|
| **Entry points** | Where does execution start? HTTP handlers, CLI commands, event listeners |
| **Data models** | Types, schemas, database tables, DTOs, domain entities |
| **Data flows** | How data moves: input → validation → processing → storage → output |
| **Dependencies** | External services, libraries, APIs, databases, message queues |
| **Configuration** | Environment variables, config files, feature flags |
| **Error handling** | Error types, recovery strategies, fallback behavior |
| **Security** | Auth/authz, input validation, encryption, access control |
| **Testing** | Test coverage, test types, key test scenarios |

**Research approach:**
1. Start from entry points (routes, handlers, exports) and trace inward
2. Map the dependency graph — what calls what
3. Read tests to understand intended behavior and edge cases
4. Check git history for recent changes and design decisions (`git log --oneline -20`)
5. Look for existing comments, ADRs, and inline documentation

**Critical rules:**
- Read every file you reference — never guess content from file names
- Trace data flows end-to-end, don't stop at abstraction boundaries
- Record exact file paths and line numbers for every finding: `src/auth/middleware.ts:42-67`

Also load pipeline artifacts:
- `spec.md` — original requirements and user stories
- `plan.md` — architecture decisions and task structure
- `review.md` — any caveats or known limitations
- `test-report.md` — what's tested, coverage numbers
- `pipeline-state.json` — deviations, cross-step impacts
- `package.json` — name, version, dependencies, scripts

### Step 3: Detect Project Type

The documentation format depends on what the project is:

| Project Type | Indicators | Doc Emphasis |
|---|---|---|
| **Library/Package** | `main`/`exports` in package.json, no server | API reference, usage examples, install |
| **REST API / Web Service** | Express/Fastify/Hono, server.ts | API endpoints, auth, error codes |
| **CLI Tool** | `bin` in package.json, commander/yargs | Commands, flags, usage examples |
| **Full-Stack App** | Frontend + backend dirs | Architecture, setup, both API + UI docs |
| **Monorepo** | workspaces, multiple packages | Per-package docs + root overview |

### Step 4: Generate Documentation

Generate all applicable docs in this order, using the templates from
`documentation/references/doc-templates.md`. Adapt each template to the project —
don't force sections that don't apply.

1. **README.md** — project front door (always)
2. **API docs** — OpenAPI or Markdown (if HTTP endpoints exist)
3. **Architecture** — system diagrams, decisions, data flow (always)
4. **Code-level docs** — JSDoc verification + TypeDoc for libraries
5. **Contributing guide** — dev setup, conventions, PR process
6. **Changelog** — initial release notes
7. **Deployment guide** — env vars, build, run, monitoring

**Write in sections of 200-400 words.** After each major doc, validate with the user
before continuing to the next. Don't dump all documentation at once.

### Step 5: Verify Documentation Accuracy

After generating all docs, verify:

1. **Every command in README works** — run `npm run dev`, `npm run build`, `npm test`
2. **Every env var in docs exists in config validation** — cross-reference `process.env.*`
   usage in source with the env var table in README
3. **Every endpoint in API docs exists in routes** — cross-reference route definitions
4. **Every listed feature exists in code** — cross-reference README features with actual
   exports/endpoints

**If anything in the docs doesn't match the code, fix the docs. Never let docs and code disagree.**

### Step 6: Finalize and Report

Create `docs/` structure:
```
README.md                    # Project root
CONTRIBUTING.md              # Project root
CHANGELOG.md                 # Project root
docs/
├── architecture.md
├── api.md (or openapi.yaml)
├── deployment.md
└── api/ (TypeDoc output, if library)
```

Update `pipeline-state.json`:

```json
{
  "phases": {
    "documentation": {
      "status": "completed",
      "completed_at": "{ISO}",
      "outputs": [
        "README.md",
        "CONTRIBUTING.md",
        "CHANGELOG.md",
        "docs/architecture.md",
        "docs/api.md",
        "docs/deployment.md"
      ],
      "doc_format": "markdown | markdown+openapi | markdown+typedoc",
      "verified": true
    }
  },
  "pipeline_status": "completed"
}
```

---

## Handoff — Pipeline Complete

> "Documentation complete. Generated:
> - README.md — project overview, setup, usage
> - docs/api.md — API endpoint documentation
> - docs/architecture.md — system architecture and decisions
> - docs/deployment.md — deployment guide with env vars
> - CONTRIBUTING.md — development setup and conventions
> - CHANGELOG.md — initial release notes
> - JSDoc added to {N} public exports
>
> All documentation verified against actual code.
>
> **The dev pipeline is complete.** The project has been brainstormed, planned,
> implemented, reviewed, tested, and documented."

---

## Key Principles

- **Docs follow code** — every statement in docs must be verifiable in source code.
- **Copy-pasteable commands** — every bash command must work as-is.
- **Derive, don't invent** — pull from spec, plan, code, and test report. Don't make up features.
- **Adapt to project type** — library gets TypeDoc + usage examples. API gets OpenAPI.
  CLI gets command reference. Don't force inappropriate formats.
- **Verify after generating** — cross-reference docs against actual code, env vars, and routes.
- **README is king** — if someone reads only one doc, it should be README. Make it complete.
- **No aspirational docs** — only document what exists. If a feature is planned but not
  built, it doesn't go in docs.
