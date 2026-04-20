# Implementation Standards Reference

Quick-reference standards for the implementation skill. These are loaded once at the
start and consulted during coding (Clean Code summary, code style defaults, scaffolding)
and after each batch (self-review checklist).

**The full rationale for each item lives in `clean-code-principles.md` and
`js-ts-best-practices.md`. This file is the condensed, actionable version.**

---

## Clean Code Standards — Every File

Apply these to every file you write. See `clean-code-principles.md` for full rationale.

- **TypeScript strict mode** — No `any` types unless explicitly justified with a comment.
  Use proper generics, unions, and type guards.
- **Intention-revealing names** — Every name answers *why it exists, what it does, how
  it's used*. If a name needs a comment to explain it, rename it. One word per concept
  across the codebase (`fetch` or `get`, not both). Use domain vocabulary.
- **Small, focused functions** — Each function does one thing at one level of abstraction.
  5-15 lines ideal, rarely over 20, never over 30. Follow the Stepdown Rule: code reads
  top-down, each function followed by those it calls. Prefer 0-2 arguments; group 3+ into
  an options object. No hidden side effects.
- **Error handling as a separate concern** — Use typed error classes extending `Error`.
  Throw exceptions, don't return error codes or null. Provide context in error messages
  (what operation failed, what input caused it, why). Define errors by caller's needs,
  not by internal library types. Handle errors at boundaries; don't let try-catch obscure
  business logic.
- **No magic values** — Extract constants. `const MAX_RETRIES = 3` not `if (retries > 3)`.
- **Comments only for "why"** — If you need a comment, first try to make the code speak
  for itself. Only comment non-obvious intent, warnings, or clarifications. Never write
  redundant comments, journal comments, or commented-out code. JSDoc only for public APIs.
- **DRY** — If the same logic appears in two places, extract it. Duplication is the root
  of all evil in software. But don't over-abstract — three similar lines are better than
  a premature abstraction.
- **Small classes/modules** — Each module has one reason to change (SRP). High cohesion:
  methods use most of the module's state. If a subset of functions uses a subset of
  variables, that's a new module waiting to be extracted.
- **Clean boundaries** — Wrap third-party APIs in your own abstractions. Don't let external
  types leak through your interfaces. Callers shouldn't know your implementation details.
- **Consistent patterns** — Follow the detected tech stack patterns from Step 2. Don't
  introduce new patterns without reason. Team rules beat personal preference.
- **Imports** — External deps first, then internal modules, then relative imports.
  Named imports preferred.
- **Boy Scout Rule** — Leave every file a little cleaner than you found it. If you touch
  a file to add a feature, clean up any small issues you see (bad name, dead code,
  unclear logic). Don't go on a refactoring spree — small, incremental improvements.

---

## Self-Review Checklist

Run through this checklist for all code written in each batch. Don't just skim —
reason through each item against the actual code you wrote.

### Security Deep Review

Complements the security gate — think adversarially:

- Any user input used without sanitization?
- Any secrets or credentials hardcoded?
- Any SQL/NoSQL injection vectors?
- Any path traversal vulnerabilities?
- Any unsafe `eval()`, `Function()`, or dynamic code execution?
- Any sensitive data logged or exposed in error messages?
- Any missing authorization checks (IDOR — can user A access user B's resources)?
- Any prototype pollution vectors (recursive merge/deep clone on user input)?
- Any SSRF vectors (server fetching user-supplied URLs without validation)?
- Any XSS vectors (user content rendered without encoding)?
- Any CSRF exposure (cookie-based auth without SameSite or CSRF tokens)?
- Any timing-safe comparison needed (secret/token comparison using `===`)?

### Clean Code Smells

See `clean-code-principles.md` § Code Smells:

- Any function over 20 lines that should be split?
- Any function with 3+ arguments that should take an options object?
- Any names that don't reveal intent? Any inconsistent naming?
- Any commented-out code, redundant comments, or journal comments?
- Any duplicated logic across files?
- Any null returns that force callers to null-check? (Return empty collections instead)
- Any raw third-party types leaking through public interfaces?
- Any boolean flag arguments? (Split into two functions)

### Pattern Consistency

- Does all new code follow the patterns detected in Step 2?
- Are naming conventions consistent with the rest of the codebase?
- Are error handling patterns consistent?
- Are import styles consistent?

### Obvious Bugs

- Any unreachable code or dead code?
- Any missing null/undefined checks on values that could be nullish?
- Any off-by-one errors in loops or slicing?
- Any async functions missing `await`?
- Any event listeners or resources not cleaned up?
- Any hidden temporal coupling (functions that must be called in order)?

---

## Project Scaffolding Checklist

For the first batch (project setup):

**package.json:**
- `npm init -y` or create manually with correct metadata from spec
- `"type": "module"` for ESM (recommended)
- `main`, `types`, `exports` for libraries
- `bin` for CLI tools
- Scripts: `build`, `dev`, `test`, `lint` as relevant

**tsconfig.json:**
- `"strict": true` by default
- Appropriate `target` and `module` for the runtime
- `outDir`, `rootDir`, `declaration` as needed

**.gitignore:**
- node_modules/, dist/, .env, coverage/, *.log, .DS_Store

**Directory structure:** Create all directories from plan.md.

---

## Code Style Defaults

Unless project config says otherwise:

- **Semicolons**: Yes
- **Quotes**: Single quotes
- **Indentation**: 2 spaces
- **Trailing commas**: Yes (ES5+)
- **Line length**: ~100 characters soft limit
- **File naming**: kebab-case (`user-service.ts`), PascalCase for components
- **Exports**: Named exports preferred (except React components)
- **Async**: Always async/await over raw promises
- **Errors**: Custom error classes extending `Error` for typed handling

If the project has existing style (eslintrc, prettierrc, editorconfig), follow that instead.

---

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/) format. Present in a
fenced code block so the user can copy and paste.

1. **Type + scope**: `feat(module)`, `fix(module)`, or `refactor(module)`
2. **Short summary** (first line, max 72 chars)
3. **Body**: list of what was implemented, grouped logically
4. **Footer**: deviations, flagged issues, items for review

**Fresh implementation:**
```
feat(url-shortener): implement URL shortening API with analytics

- Add URL creation, resolution (302 redirect), and deletion endpoints
- Add click tracking with fire-and-forget analytics recording
- Set up PostgreSQL (Prisma) + Redis (ioredis) infrastructure
- Add centralized error handling with custom error classes

Deviations from plan:
- Used ioredis instead of node-redis (better TypeScript support)

Flagged for review:
- Rate limiter cleanup of expired sorted set members
```

**Fix mode** (applying `fix-plan.md` or `fix-test-plan.md`):
```
fix(url-shortener): address code review findings [CR-001..CR-004]

- Add timeout on redirect DB fallback (CR-001)
- Clean up expired rate limit entries (CR-002)
- Log failed click recordings (CR-003)
- Report Redis status in health check (CR-004)
```

> "Here's a commit message you can use. Want me to commit, or would you prefer to adjust it first?"

---

## Finding Outcome Tracking (Fix Mode)

After applying all fixes from `fix-plan.md`, produce a finding outcomes table documenting
what happened to every `[CR-NNN]`:

| Finding | Status | Reason |
|---------|--------|--------|
| [CR-001] | Fixed | Timeout added as specified |
| [CR-002] | Modified | Used interval-based cleanup instead of per-request |
| [CR-003] | Dismissed | Already handled by framework error middleware |
| [CR-004] | Deferred | Requires Redis Sentinel — tracked in backlog |

**Status values:**
- **Fixed** — applied exactly as suggested
- **Modified** — fixed the underlying issue but with a different approach (explain why)
- **Dismissed** — finding is incorrect or already handled (provide evidence)
- **Deferred** — valid but out of scope for this cycle (document where it's tracked)

Include this table in the batch report. The code review re-review reads these outcomes
to avoid re-raising dismissed findings and to track review effectiveness.