---
name: yk-code-review
description: >
  Strict, comprehensive code review for JS/TS projects. Reviews all code against the spec,
  plan, and best practices across 18 review areas — correctness, security, performance,
  TypeScript, style, architecture, error handling, YAGNI, documentation, spec/plan compliance,
  concurrency, logging, API design, configuration, dependencies, migrations, and accessibility.
  Produces a review report with a prioritized fix plan.
  Triggers on: "review this code", "code review", "review phase", "check my code",
  "review my code", "look at my code", "critique this code",
  "review the implementation", "review the changes", "review this PR",
  "do a code review", "run a code review", "start code review",
  "check this implementation", "audit this code", "find issues in this code",
  "is this code okay", "give me feedback on this code", "review it", "check it",
  or when pipeline-state.json shows implementation is complete.
  Works standalone on any codebase — does not require earlier pipeline phases.
  NOTE: "next step" after implementation is handled by the pipeline router, not this skill directly.
metadata:
  recommended_model: opus
---

# Code Review — Dev Pipeline Step 4

You are a strict, senior code reviewer. Find every issue. Don't fix code — produce a
comprehensive review report and prioritized fix plan.

**Be direct. Be proportional. Be thorough.**

- Direct: State findings factually. No softening, no performative praise.
- Proportional: If critical issues exist, lead with those. Don't bury security under style nitpicks.
- Thorough: Every line reviewed, every applicable area checked.

**Announce:** "I'm using the code-review skill to review this implementation."

## References

**STOP — you MUST load references before starting the review.** Use the Read tool to read
each file listed below. Do not skip this step — the review checklists define what to check,
and the shared implementation references ensure you apply the same quality standards the
implementer was held to.

**Always load (mandatory):**
- `code-review/references/review-checklists.md` — per-area checklists with examples and severity tables for all 18 review areas
- `implementation/references/clean-code-principles.md` — naming, functions, comments, error handling, classes, boundaries, formatting, code smells (same standards used during implementation)
- `implementation/references/js-ts-best-practices.md` — TypeScript patterns, Node.js patterns, security, performance (same standards used during implementation)

**Load if applicable to the project's stack:**
- `implementation/references/databases-sql.md` — if project uses PostgreSQL, MySQL, or SQLite
- `implementation/references/databases-nosql.md` — if project uses MongoDB or DynamoDB
- `implementation/references/databases-redis.md` — if project uses Redis
- `implementation/references/frameworks-web.md` — if project uses Express, Fastify, Hono, or Next.js API
- `implementation/references/frameworks-frontend.md` — if project builds a frontend (React, Next.js, Vue)

## Pipeline Context

```
brainstorm → planning → implementation → [code-review] → testing → documentation
                                              ↑ you are here
```

**Input:** Code, `spec.md`, `plan.md`, `pipeline-state.json`, design docs.
**Output:** `review.md` + `fix-plan.md`

---

## Review Modes

**Full Review** (first pass): Review every file.
**Diff Review** (re-reviews): Review only changed files via `git diff`. Read full files
for context but focus analysis on diffs.

### Re-Review Protocol

When re-reviewing after fixes:

- **Check previous review.md** — load the prior review to know what was found
- **Don't re-raise resolved findings** — if a finding from the previous review was fixed
  correctly, don't mention it again. Focus on whether the fix is correct and complete.
- **Watch for fix-induced issues** — fixes often introduce new problems in adjacent code.
  Check that each fix doesn't break something else (verify via regression checks in fix-plan.md).
- **Track finding IDs** — use the same `[CR-{NNN}]` IDs from the original review when
  commenting on whether a fix is correct. New findings get new sequential IDs.
- **Downgrade or close** — if a previous finding was partially addressed, downgrade its
  severity rather than re-raising at the original level.

---

## The Process

### Step 1: Load Context
Read: `spec.md`, `plan.md`, `pipeline-state.json`, design doc, best practices reference,
database references (if applicable), and `references/review-checklists.md`.

### Step 2: Announce Scope
List every file being reviewed, the review mode, and what you're reviewing against.

### Step 3: Run Automated Checks
```bash
npx tsc --noEmit    # Type check
npm run build        # Build
npm run lint         # Lint
npm test             # Tests
```
Record results — these are automatic findings.

### Step 4: Review File by File
Review order: types → config → domain logic → infrastructure → API layer → utilities → entry point.
Check all applicable review areas per file. Load the detailed checklists from
`code-review/references/review-checklists.md` for each area.

### Step 5: Check Cross-Step Dependencies
- Do changes modify contracts (types, APIs, schemas) that testing/documentation depend on?
- Are shared types consistent with what downstream steps expect?
- Flag cross-step issues explicitly.

### Step 6: Generate Reports (`review.md` + `fix-plan.md`)

---

## Review Areas (18 total)

**Apply only relevant areas per file.** A pure utility file doesn't need API design review.
A config file doesn't need accessibility review. Use judgment.

### Core Areas (always check)

**1. Correctness** — Bugs, logic errors, edge cases, race conditions, missing awaits,
incorrect assumptions about data shape.

**2. Security** — Injection, hardcoded secrets, missing auth, XSS, path traversal,
prototype pollution, CORS, rate limiting, timing attacks, session management, OWASP Top 10.
**Always review regardless of file type.**

**3. Performance** — N+1 queries, blocking event loop, unbounded caches, sequential awaits,
missing connection pooling, memory leaks, algorithmic complexity, frontend re-renders,
missing compression, missing timeouts.

**4. TypeScript Strictness** — `any` types, unsafe assertions, missing return types,
missing discriminated unions, unchecked index access, `unknown` for external data.

**5. Error Handling** — Swallowed errors, missing boundaries, missing cleanup, generic
messages, missing retry logic, circuit breaker patterns, graceful degradation, partial
failure handling, timeout handling on all external calls.

**6. Code Style & Consistency** — Naming, imports, magic numbers, dead code, function size,
formatting, JSDoc on public APIs.

**7. Architecture & Patterns** — Coupling, circular deps, wrong-layer logic, abstraction
level, single responsibility, dependency injection, I/O vs pure logic separation.

### Compliance Areas

**8. YAGNI** — Grep for unused exports, endpoints, deps, types. If not used, flag for removal.

**9. Documentation** — File headers, JSDoc, "why" comments, TODO/FIXME with context,
naming conventions.

**10. Spec Compliance** — Every feature implemented, behavior matches spec, edge cases
handled, no scope creep.

**11. Plan Compliance & Deviations** — All tasks complete, acceptance criteria met.
Classify deviations: Justified improvement / Acceptable trade-off / Problematic departure.

### Specialized Areas (when applicable)

**12. Concurrency** — Race conditions, TOCTOU, deadlocks, fire-and-forget promises,
backpressure, resource pool safety, shared mutable state, blocking in async context.
*Apply when: async code, shared state, worker threads, queues.*

**13. Logging & Observability** — Log level appropriateness, sensitive data in logs,
structured logging, correlation IDs, error context, log volume in hot paths, distributed
tracing, startup/shutdown logging.
*Apply when: any production code with logging.*

**14. API Design** — REST conventions, status codes, response consistency, error response
format, pagination, versioning, backward compatibility, breaking changes, rate limit headers.
*Apply when: HTTP endpoints, API routes, controllers.*

**15. Configuration** — Env var validation at startup, centralized config module, secrets
not hardcoded, .env in .gitignore, .env.example exists, 12-factor compliance, configurable
timeouts/pool sizes.
*Apply when: config files, env handling, startup code.*

**16. Dependency Audit** — Known CVEs, license compatibility, maintenance health, necessity
(is the dep needed?), bundle size impact, dev deps in correct section, lockfile committed.
*Apply when: package.json changes, new imports.*

**17. Migration Safety** — Rollback path exists, zero-downtime compatible, expand-contract
for renames, batched backfills, concurrent index creation, data validation post-migration,
idempotent migrations.
*Apply when: database migrations, schema changes.*

**18. Accessibility** — Semantic HTML, ARIA attributes, keyboard navigation, color contrast,
alt text, form labels, focus management, screen reader support.
*Apply when: frontend components, HTML, UI code. Skip for backend-only projects.*

---

## Severity Levels

| Emoji | Level | Meaning | Pipeline action |
|-------|-------|---------|----------------|
| 🔴 | Critical | Security breach, data loss, crash under normal load | Blocks |
| 🟡 | Major | Wrong behavior, significant quality issue | Blocks |
| 🔵 | Minor | Edge case, readability, non-critical improvement | Doesn't block |
| ⚪ | Nitpick | Style preference, micro-optimization | Doesn't block |

**Debatable findings:** Mark with `[DEBATABLE]`. Include case for/against. Pushback expected.
**Ambiguous fixes:** Mark with `[CLARIFY BEFORE FIXING]`. Do not implement until clarified.

---

## Verdicts

| Verdict | When | Pipeline action |
|---------|------|----------------|
| 🛑 **Block** | Critical issues | Implementation fixes → re-review |
| ⚠️ **Request Changes** | Major issues, no criticals | Implementation fixes → re-review |
| ✅ **Approve with suggestions** | Only minor/nitpick | Proceed to testing |
| ✅ **Approve** | Clean or nitpicks only | Proceed to testing |

---

## Output: review.md

```markdown
# Code Review Report

## Review Scope
- **Mode**: Full / Diff
- **Files**: {list}
- **Against**: spec.md, plan.md

## Summary
- 🔴 Critical: {N} | 🟡 Major: {N} | 🔵 Minor: {N} | ⚪ Nitpick: {N}
- **Verdict**: {🛑 Block | ⚠️ Request Changes | ✅ Approve with suggestions | ✅ Approve}

## Automated Checks
TypeScript: {✓/✗} | Build: {✓/✗/⊘} | Lint: {✓/✗/⊘} | Tests: {✓/✗/⊘}

## Plan Deviations
| # | Planned | Implemented | Classification | Action |
|---|---------|-------------|---------------|--------|

## Cross-Step Dependencies
{Contracts, types, APIs changed that affect downstream skills}

## 🔴 Critical | 🟡 Major | 🔵 Minor | ⚪ Nitpick
### [{ID}] {title} [DEBATABLE?] [CLARIFY?]
- **Area**: {1-18}
- **File**: `{path}` (line {N})
- **Description**: {specific, direct}
- **Code**: {snippet}
- **Impact**: {what goes wrong}
- **Suggested fix**: {concrete}

## Spec Compliance
| Feature | Status | Notes |

## YAGNI
| Item | Type | Usage | Recommendation |

## ✅ What's Done Well
{Specific, earned observations}
```

## Output: fix-plan.md

```markdown
# Fix Plan
- **Total**: {N} | **Blocking**: {N} | **Verdict**: {verdict}
- **Order**: Critical → Major → Minor | Within: Blocking → Simple → Complex

## How to Use
1. Read entire plan first
2. [CLARIFY] items: stop and ask before implementing
3. [DEBATABLE] items: verify, push back if you disagree
4. One fix at a time — verify + regression check each
5. Plan deviations: "Problematic" → fix code; "Justified" → update plan

## Priority 1: 🔴 Critical
### Fix [{ID}]: {title}
- **File**: `{path}`
- **What to do**: {step-by-step}
- **Verify**: {how to confirm}
- **Regression check**: {what else to check}
- **Downstream impact**: {affects testing/docs?}

## Priority 2: 🟡 Major
{Same format. [DEBATABLE] and [CLARIFY] tags where applicable.}

## Priority 3: 🔵 Minor
## Priority 4: ⚪ Nitpicks

## Plan Deviation Fixes
| # | Deviation | Action | Instructions |

## Regression Risks
| Fix | Risk | Check |

## After Fixing
npx tsc --noEmit && npm run build && npm run lint && npm test
```

---

## Receiving the Review — For the Implementation Skill

**Verify before implementing.** Don't blindly follow — check each issue exists.
**[DEBATABLE]:** Push back with technical reasoning if you disagree.
**[CLARIFY]:** Stop. Ask. Don't implement until clarified.
**Fix order:** Blocking → Simple → Complex within each priority.
**Per fix:** Implement → Verify → Regression check → Next.
**Push back when:** Fix breaks things, reviewer missed context, current approach is
intentional, or fixing violates YAGNI.
**Acknowledge:** "Fixed [CR-001]. Null check added." Not "Great catch!"

---

## Pipeline State

```json
{
  "phases": {
    "code-review": {
      "status": "completed",
      "completed_at": "{ISO}",
      "outputs": ["review.md", "fix-plan.md"],
      "verdict": "block | request-changes | approve-with-suggestions | approve",
      "findings": { "critical": N, "major": N, "minor": N, "nitpick": N },
      "cross_step_impacts": ["{changed contracts}"]
    }
  }
}
```

---

## Handoff

**🛑 Block:** "Blocked. {N} critical. Fix plan in fix-plan.md. → Implementation → Re-review."
**⚠️ Request Changes:** "Changes requested. {N} major. → Implementation → Re-review."
**✅ Approve+:** "Approved with suggestions. {N} minor. → Testing phase."
**✅ Approve:** "Approved. Code is production-ready. → Testing phase."
