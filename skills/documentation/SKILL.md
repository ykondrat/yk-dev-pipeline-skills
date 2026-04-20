---
name: yk-documentation
description: >
  Comprehensive documentation generation for JS/TS projects. The final step in the dev
  pipeline. Generates README, API docs, architecture docs, code-level docs, contributing
  guide, changelog, and deployment guide — all derived from actual code.
  Triggers on: "write docs", "documentation phase", "generate docs",
  "write documentation", "create docs", "document this", "doc this", "doc it",
  "add documentation", "update the docs", "generate documentation",
  "write a README", "update the README", "create API docs", "document the API",
  "add JSDoc", "document the code", "start documentation",
  "we need docs", "this needs documentation", "architecture docs",
  or when pipeline-state.json shows testing is complete.
  Works standalone on any codebase — does not require earlier pipeline phases.
  NOTE: "next step" after testing is handled by the pipeline router, not this skill directly.
metadata:
  recommended_model: sonnet
---

> **⚠️ Legacy skill.** The authoritative Phase 6 implementation is the **`documentation` agent** at [`../../agents/documentation.md`](../../agents/documentation.md). This skill is kept for backward compatibility (Claude.ai Projects uploads and skill-only Claude Code installs). Edits here may drift from the agent version — see [`docs/review-2026-04-17.md`](../../../../docs/review-2026-04-17.md) for the consolidation plan.

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

**STOP — you MUST load references before generating documentation.** Use the Read tool to
read the file below. Do not skip this step.

**Always load (mandatory):**
- `../../references/documentation/doc-templates.md` — full templates for all 7 doc types: README, API docs (OpenAPI + Markdown), architecture (with Mermaid diagrams), code-level docs (JSDoc + TypeDoc), contributing guide, changelog, and deployment guide

---

## Documentation Modes

**Full Generation** (default, first pass): Generate all docs from scratch.
**Update Mode** (incremental): Update only docs affected by code changes since last
documentation. Triggered by "update docs", "refresh docs", "docs are stale", or when
re-running the pipeline on an existing documented project.

### Detecting Update Mode

Check `pipeline-state.json` for:
- `documentation.status === "completed"` — docs were previously generated
- `documentation.last_documented_commit` — the git commit when docs were last generated

If both exist, compare `last_documented_commit` to HEAD:
```bash
git diff {last_documented_commit}..HEAD --name-only
```

If there are changes, announce update mode:
> "Previous docs exist (generated at commit {hash}). {N} files changed since then.
> I'll update only affected documentation sections."

Map changed files to affected doc sections:
| Changed file type | Affected docs |
|---|---|
| Route/handler files | API docs, README endpoints section |
| Config / env handling | Deployment guide env table, README config section |
| Public exports / types | API reference, architecture data models |
| package.json (scripts, deps) | README setup/install, contributing guide |
| New source modules | Architecture docs, README feature list |
| Test files | No doc update needed (unless coverage changes significantly) |

### Drift Detection (Standalone)

Can run anytime — triggered by "check docs", "are docs up to date?", "doc health".
Without generating or changing anything, compare existing docs against current code:

1. **Env var check** — compare env vars listed in docs vs `process.env.*` in source
2. **Endpoint check** — compare endpoints in API docs vs route definitions
3. **Feature check** — compare features in README vs actual exports/endpoints
4. **Command check** — compare CLI commands in docs vs registered commands in code
5. **Dependency check** — compare deps listed in docs vs package.json

Report drift as: current (matches code), stale (exists but outdated), missing (in code
but not in docs).

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
`../../references/documentation/doc-templates.md`. Adapt each template to the project —
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

### Step 6: Doc Coverage Analysis

Inventory the project's public API surface and check what's documented:

**Inventory these categories:**
1. **Public exports** — every `export` from package entry points (for libraries)
2. **HTTP endpoints** — every registered route with method, path, and handler
3. **Environment variables** — every `process.env.*` or config reference
4. **Error types** — every custom error class or error code
5. **CLI commands** — every registered command and flag (for CLI tools)

**For each item, classify:**
- **Documented** — appears in docs with description and example
- **Mentioned** — appears in docs but lacks description or example
- **Undocumented** — exists in code but not in any doc

**Target: 100% of public API surface documented.** If an export/endpoint/env var is
intentionally internal, it should be marked `@internal` in JSDoc.

**Read `doc_impacts` from code review** (if available in `pipeline-state.json`). These
are changes the reviewer flagged as requiring doc updates — make sure each one is covered.

### Step 7: Generate Doc Report

Create `{project-dir}/doc-report.md`:

```markdown
# Documentation Report

## Summary
- **Mode**: Full / Update
- **Docs generated**: {list}
- **Verified against code**: ✓

## Doc Coverage
| Category | Total | Documented | Mentioned | Undocumented | Coverage |
|----------|-------|-----------|-----------|-------------|----------|
| Public exports | {N} | {N} | {N} | {N} | {N}% |
| HTTP endpoints | {N} | {N} | {N} | {N} | {N}% |
| Environment variables | {N} | {N} | {N} | {N} | {N}% |
| Error types | {N} | {N} | {N} | {N} | {N}% |
| CLI commands | {N} | {N} | {N} | {N} | {N}% |
| **Total** | **{N}** | **{N}** | **{N}** | **{N}** | **{N}%** |

## Undocumented Items
{List each undocumented item with its file:line location.
Mark items that are intentionally internal (@internal).}

## Cross-Reference Validation
| Doc Claim | Source Location | Status |
|-----------|----------------|--------|
| "POST /api/users creates a user" | src/routes/users.ts:23 | ✓ Verified |
| "Requires DATABASE_URL env var" | src/config.ts:5 | ✓ Verified |

## Drift Analysis (update mode only)
| Section | Status | Details |
|---------|--------|---------|
| API endpoints | Current | No endpoint changes |
| Env vars | Stale | Added REDIS_URL, not in docs → fixed |
| README features | Current | No new features |

## Review Doc Impacts (if code review flagged changes)
| Impact | Source | Doc Updated |
|--------|--------|-------------|
| New endpoint POST /api/tokens | [CR-012] | ✓ Added to api.md |
| Changed auth middleware signature | [CR-015] | ✓ Updated architecture.md |
```

### Step 8: Finalize and Report

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
      "mode": "full | update",
      "last_documented_commit": "{git commit hash}",
      "verified": true,
      "doc_coverage": {
        "public_exports": "{N}/{N}",
        "endpoints": "{N}/{N}",
        "env_vars": "{N}/{N}",
        "error_types": "{N}/{N}",
        "overall": "{N}%"
      }
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
> - doc-report.md — coverage analysis and cross-reference validation
> - JSDoc added to {N} public exports
>
> Doc coverage: {N}% of public API surface documented. All docs verified against code.
>
> **The dev pipeline is complete for the selected scope.** Completed phases:
> {list the phases from `selected_phases` that have status "completed"}."

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
- **Update, don't regenerate** — when docs exist and code changed, update only affected
  sections. Drift detection prevents stale docs.
- **100% public API coverage** — every export, endpoint, env var, and error type that's
  part of the public surface must be documented. Track coverage in doc-report.md.
