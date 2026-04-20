# Review Checklists — Deep Reference

Detailed checklists for each of the 18 review areas. Load this before reviewing.
Each section includes what to check, common patterns to flag, and severity tables.

---

## Area 1: Correctness — Deep Checklist

- Off-by-one errors in loops, slicing, pagination
- Incorrect boolean logic (De Morgan's law violations, inverted conditions)
- Race conditions in async code (check-then-act without atomicity)
- Missing `await` on promises (fire-and-forget without error handling)
- `==` vs `===` (loose equality with type coercion)
- Variable shadowing in nested scopes
- Unreachable code after returns/throws/breaks
- Incorrect assumptions about data shape (array always has elements, object always has key)
- Mutation of shared/global state from multiple call sites
- Incorrect regex (wrong escaping, missing anchors, greedy vs lazy)
- Division by zero without guard
- Integer overflow for large numbers (Number.MAX_SAFE_INTEGER)
- Off-by-one in string slicing or array boundaries
- Incorrect ternary operator precedence
- Missing break in switch statements (unintentional fallthrough)

---

## Area 2: Security — Deep Checklist (OWASP Top 10)

### Injection
- SQL injection: string concatenation/template literals in queries → use parameterized
- NoSQL injection: user input as query operators (`{ "$gt": "" }`)
- Command injection: `exec`, `spawn`, `system` with user args
- XSS: unescaped user content in HTML (`innerHTML = userInput`)
- LDAP/XPath injection: dynamic query with user input
- Header injection: user input in HTTP headers (CRLF)
- ReDoS: user-supplied regex or input triggering catastrophic backtracking

### Authentication & Authorization
- Auth checks on all protected endpoints (not just frontend)
- Session management: HttpOnly, Secure, SameSite cookie flags
- Password hashing: bcrypt/argon2/scrypt (NOT MD5/SHA1)
- Rate limiting on login/signup endpoints
- JWT validation: algorithm, expiry, issuer, audience
- RBAC/ABAC enforcement (not just "is logged in")
- OAuth flows using PKCE where applicable

### Sensitive Data
- Hardcoded API keys, tokens, passwords, connection strings
- Secrets in committed config files
- Sensitive data in logs (passwords, tokens, PII, credit cards)
- Sensitive data in error messages returned to clients
- Missing encryption at rest or in transit
- Credentials in URL query parameters
- Session IDs in URLs

### Input Validation
- All external input validated (HTTP params, headers, body, file uploads)
- File uploads restricted by type, size, content validation
- Open redirect: redirects validated against allowlist
- Deserialization of untrusted data avoided or sandboxed

### Security Headers & Config
- CORS: not `Access-Control-Allow-Origin: *` with credentials
- CSP, X-Frame-Options, X-Content-Type-Options headers
- HTTPS enforcement and HSTS
- Debug mode disabled in production
- Timing-safe comparison for secrets (`crypto.timingSafeEqual`)

| Issue | Severity |
|---|---|
| SQL/NoSQL/command injection | 🔴 Critical |
| Hardcoded secret | 🔴 Critical |
| Missing auth check on endpoint | 🔴 Critical |
| XSS via unescaped output | 🔴 Critical |
| Weak password hashing (MD5/SHA1) | 🔴 Critical |
| Open redirect | 🟡 Major |
| Overly permissive CORS | 🟡 Major |
| Missing rate limiting on login | 🟡 Major |
| Debug mode in production config | 🟡 Major |
| Missing security headers | 🔵 Minor |
| Unpinned dependencies | ⚪ Nitpick |

---

## Area 3: Performance — Deep Checklist

### Database & Queries
- N+1 queries (query inside loop → batch/join)
- Missing indexes on WHERE/JOIN/ORDER BY columns
- `SELECT *` when specific columns suffice
- Unbounded queries (no LIMIT on user-facing queries)
- Multiple queries combinable into one
- Missing connection pooling
- Missing query timeouts

### Algorithmic Complexity
- Nested loops producing O(n²) where O(n) is possible
- Linear search where Set/Map lookup is O(1)
- Repeated computation (should memoize)
- String concatenation in loops (use join/builder)
- Sorting where partial sort/heap suffices

### Memory & Resources
- Large collections loaded into memory (should stream/paginate)
- Unnecessary cloning in hot paths
- Unbounded caches without eviction
- Missing resource cleanup (unclosed streams, connections, handles)

### Network & I/O
- Sequential async calls that could be `Promise.all`
- Missing timeouts on HTTP/DB calls
- Large payloads without pagination or compression
- Blocking I/O in async contexts

### Caching
- Repeated identical queries/computations (should cache)
- Missing HTTP cache headers (ETag, Cache-Control)
- Cache without TTL or eviction (unbounded growth)
- Missing cache invalidation on mutation

### Frontend (when applicable)
- Unnecessary re-renders (unstable references, missing useMemo/useCallback)
- Large bundle imports (should tree-shake or lazy load)
- Unoptimized images/assets
- Missing virtualization for long lists
- Layout thrashing (interleaved DOM reads/writes)

| Issue | Severity |
|---|---|
| N+1 query pattern | 🟡 Major |
| Unbounded query without LIMIT | 🟡 Major |
| O(n²) where O(n) is possible | 🟡 Major |
| Missing timeout on external call | 🟡 Major |
| Sequential awaits (could parallelize) | 🔵 Minor |
| SELECT * in production | 🔵 Minor |
| Missing cache headers | ⚪ Nitpick |

---

## Area 4: TypeScript Strictness — Deep Checklist

- `any` types — every instance needs explicit justification
- Type assertions (`as`) without runtime validation
- `!` (non-null assertion) without guaranteed context
- Missing return types on public/exported functions
- `@ts-ignore`/`@ts-expect-error` without comment explaining why
- Loose generics where specific types are possible
- Missing discriminated unions for state (boolean flags instead)
- Unchecked index access (without `noUncheckedIndexedAccess`)
- Improper narrowing (typeof for objects, missing type guards)
- Missing `readonly` on data that shouldn't mutate
- `object` or `{}` instead of specific types
- Missing `unknown` for external data (API responses, file reads, user input)
- Enums used where `as const` objects would be more type-safe

| Issue | Severity |
|---|---|
| `any` hiding a runtime bug | 🔴 Critical |
| `any` on significant code path | 🟡 Major |
| Missing type annotation (inference works) | 🔵 Minor |
| Type could be more precise | ⚪ Nitpick |

---

## Area 5: Error Handling — Deep Checklist

### Propagation
- Errors propagated to appropriate level (not swallowed)
- Errors wrapped with context when re-thrown (preserving cause via ES2022 `cause`)
- Clear error type hierarchy (recoverable vs fatal, user vs internal)
- Typed errors (custom classes, Result types) where appropriate

### Boundaries
- Error boundaries at system edges (API handlers, message consumers, background jobs)
- Boundaries catch, log, and convert to appropriate responses
- Unexpected errors → generic 500 (no internal leaks)
- Domain errors → specific HTTP codes (400, 404, 409, 422)
- Global error handler as safety net

### Retry & Resilience
- Retries only on idempotent operations (**NEVER** retry payment charges)
- Exponential backoff with jitter (not fixed-interval)
- Max retry count (no infinite loops)
- Non-retryable errors excluded (4xx not retried)
- Retries logged with attempt count

### Circuit Breakers
- External calls protected by circuit breakers
- Fallback when circuit is open (cache, default, degradation)
- State changes logged and monitored

### Resource Cleanup
- Resources released in `finally` blocks
- Transactions rolled back on error
- Temp files cleaned up on failure
- Connections returned to pools in error paths

### Graceful Degradation
- System continues when non-critical dependency fails
- Timeouts on all external calls
- Partial failure handling in batch operations (report which items failed)
- Fallback behavior when optional features unavailable

| Issue | Severity |
|---|---|
| Silently swallowed exception | 🔴 Critical |
| Missing error boundary (crash) | 🔴 Critical |
| Retry on non-idempotent op | 🔴 Critical |
| Lost error context on re-throw | 🟡 Major |
| Missing resource cleanup | 🟡 Major |
| Fixed-interval retries | 🟡 Major |
| Generic "something went wrong" | 🔵 Minor |
| Missing timeout on external call | 🔵 Minor |

---

## Area 6: Code Style — Deep Checklist

- Inconsistent naming conventions (mixing camelCase/snake_case)
- Inconsistent file naming (mixing kebab-case/camelCase)
- Inconsistent import ordering and style
- Magic numbers without named constants
- Functions >30 lines that should be split
- Dead code: commented-out code, unused imports/variables
- Poor naming (abbreviations, single-letter outside loops, misleading)
- Inconsistent async patterns (mixing `.then()` and `await`)
- Missing JSDoc on public API functions

---

## Area 7: Architecture — Deep Checklist

- Tight coupling (reaching into module internals)
- Circular dependencies
- Business logic in wrong layer (SQL in route handlers)
- Missing abstraction (duplicated logic)
- Over-abstraction (unnecessary indirection)
- Single responsibility violation
- God objects/files (>300 lines, mixed responsibilities)
- Missing dependency injection (hard to test)
- Missing I/O vs pure logic separation
- Improper design pattern usage (pattern where simpler code works)

---

## Area 8: YAGNI — Deep Checklist

### Unused Code
- Unused exports (exported but never imported elsewhere)
- Unused function parameters (present in signature, never read)
- Unused variables, constants, or type aliases
- Dead code paths (conditions that can never be true)
- Commented-out code blocks (should be deleted, not preserved)
- Unused dependencies in `package.json` (installed but never imported)
- Unused dev dependencies (listed but not referenced in scripts or config)

### Over-Engineering
- Abstractions with only one implementation (premature interface/factory)
- Configuration options nobody sets (only defaults used)
- Generic solutions for specific, one-off problems
- Feature flags for features that are always on or always off
- Wrapper classes that add no behavior (pass-through layers)
- Utility files with a single function (inline instead)
- Premature optimization without profiling evidence

### Speculative Features
- Endpoints with no frontend consumer or documented API client
- Database columns populated but never read
- Event handlers registered but never triggered
- Error handling for impossible states (e.g., validating already-validated data)
- Backward-compatibility code for hypothetical migration paths
- Extensibility hooks with no planned extensions

### Unnecessary Complexity
- Design patterns where plain functions suffice
- Dependency injection frameworks when constructor injection works
- Complex state machines for linear workflows
- Event-driven architecture for synchronous, single-step operations
- Microservice boundaries within a monolith

| Issue | Severity |
|---|---|
| Unused dependency with known CVE | 🔴 Critical |
| Dead code hiding a bug (unreachable fix) | 🟡 Major |
| Entire unused endpoint or module | 🟡 Major |
| Abstraction with single implementation | 🔵 Minor |
| Unused import or variable | 🔵 Minor |
| Commented-out code block | ⚪ Nitpick |
| Premature abstraction (one-off helper) | ⚪ Nitpick |

---

## Area 9: Documentation — Deep Checklist

### Public API Documentation
- All exported functions have JSDoc with `@param`, `@returns`, `@throws`
- Type-level documentation for complex types and interfaces
- Module-level doc comment explaining purpose and usage
- Examples in JSDoc for non-obvious APIs (`@example`)
- Deprecation notices with migration path (`@deprecated`)

### Inline Comments
- "Why" comments for non-obvious decisions (not "what" comments restating code)
- Business rule comments linking to specs or requirements
- Algorithm explanations for complex logic
- Workaround comments with issue/ticket references
- No misleading or outdated comments (worse than no comment)

### TODO/FIXME Tracking
- Every `TODO` has context: who, why, and ticket/issue reference
- Every `FIXME` has severity indication and timeline
- No stale TODOs (older than current sprint/milestone)
- No `HACK` without explanation and planned resolution
- `@ts-ignore`/`@ts-expect-error` has accompanying explanation

### File & Module Organization
- File headers for non-obvious modules (purpose, dependencies, constraints)
- README in non-trivial directories (e.g., `scripts/`, `migrations/`)
- Naming conventions self-documenting (file names match exports)
- Barrel files (`index.ts`) don't obscure module structure

### Accuracy
- Comments match actual behavior (no stale descriptions)
- Parameter descriptions match actual types and constraints
- Examples compile and produce described output
- Referenced tickets/URLs still valid

| Issue | Severity |
|---|---|
| Misleading comment contradicting code | 🟡 Major |
| Missing JSDoc on public API consumed externally | 🟡 Major |
| Stale TODO referencing completed work | 🔵 Minor |
| Missing "why" comment on workaround | 🔵 Minor |
| Missing module-level doc | ⚪ Nitpick |
| "What" comment restating obvious code | ⚪ Nitpick |

---

## Area 10: Spec Compliance — Deep Checklist

### Feature Completeness
- Every feature in `spec.md` has a corresponding implementation
- All user-facing behaviors described in spec are present and reachable
- Required edge cases from spec are handled (not just happy path)
- All specified input validations are enforced
- Default values match spec (not silently different)
- Missing features cataloged as explicit findings (not silently skipped)

### Behavioral Correctness
- Output format matches spec (field names, types, structure)
- Error messages and codes match spec definitions
- Status codes match spec for each endpoint/action
- Pagination, sorting, filtering work as specified
- Rate limits, timeouts, and quotas match spec values

### Scope Discipline
- No unrequested features added (scope creep)
- No spec requirements reinterpreted without documented justification
- Optional/nice-to-have items from spec clearly marked as not-yet-implemented
- Deviations from spec documented in `review.md` with rationale

### Edge Cases & Boundaries
- Empty input/collection cases handled as spec describes
- Maximum/minimum values from spec enforced
- Concurrent access scenarios from spec addressed
- Error recovery flows match spec expectations

| Issue | Severity |
|---|---|
| Feature from spec completely missing | 🔴 Critical |
| Behavior contradicts spec | 🔴 Critical |
| Edge case from spec unhandled | 🟡 Major |
| Scope creep (unrequested feature) | 🟡 Major |
| Default value differs from spec | 🟡 Major |
| Spec ambiguity not flagged `[CLARIFY BEFORE FIXING]` | 🔵 Minor |
| Optional spec item not implemented (and not noted) | 🔵 Minor |

---

## Area 11: Plan Compliance & Deviations — Deep Checklist

### Task Completion
- Every task in `plan.md` has corresponding code changes
- Acceptance criteria from each task are satisfied
- Tasks marked as completed actually pass their criteria (not just "code exists")
- No tasks silently skipped or deferred without documentation
- Dependency order from plan was respected (no broken assumptions)

### Deviation Classification
- **Justified Improvement**: Better approach discovered during implementation. Document why it's better. Verify it still meets the spec. *(Acceptable — note in review)*
- **Acceptable Trade-off**: Pragmatic simplification that doesn't harm quality. Document what was traded and why. *(Acceptable — note in review)*
- **Problematic Departure**: Plan was ignored without good reason. Original approach may have been better. *(Flag for discussion — may block)*

### File Structure
- Files created match planned file structure (or deviations justified)
- No orphan files outside planned structure (unexplained extras)
- Module boundaries respect planned architecture
- Shared types/interfaces placed where plan specified

### Technical Decisions
- Tech stack matches plan (or deviation documented)
- Libraries/packages match plan recommendations
- Patterns (repository, factory, etc.) used as planned
- Database schema matches planned design (or migration plan updated)

### Plan Gaps
- Ambiguities in plan flagged during implementation (not silently resolved)
- Missing tasks discovered during implementation added to findings
- Plan assumptions that proved wrong documented
- Plan tasks that were infeasible documented with alternatives

| Issue | Severity |
|---|---|
| Problematic departure from plan architecture | 🟡 Major |
| Task from plan completely skipped | 🟡 Major |
| Acceptance criteria not met | 🟡 Major |
| Plan gap not flagged during implementation | 🔵 Minor |
| Justified improvement not documented | 🔵 Minor |
| File structure differs from plan (minor) | ⚪ Nitpick |

---

## Area 12: Concurrency — Deep Checklist

### Race Conditions
- Shared mutable state accessed from multiple handlers without sync
- TOCTOU without atomic operations
- Compound operations on shared data not atomic
- File/DB operations racing between concurrent requests

### Deadlocks
- Multiple locks acquired in inconsistent order
- Circular dependencies between locks/resources
- Lock timeouts not configured
- Locks held during I/O (unnecessary contention)

### Async/Await Correctness
- Async functions properly awaited (no fire-and-forget without error handling)
- Concurrent operations correctly combined (`Promise.all`, etc.)
- Backpressure handled (producing faster than consuming)
- Async resources cleaned up on cancellation/timeout
- No blocking I/O in async contexts

### Thread Safety
- Thread-safe collections for shared state
- Lazy init patterns thread-safe
- Singletons safe for concurrent access
- No modification during iteration on shared collections

### Resource Pools
- Pooled resources thread-safe
- Resources returned in all paths (including error)
- Pool exhaustion handled gracefully
- Pool sizes appropriate for expected concurrency

| Issue | Severity |
|---|---|
| Shared mutable state without sync | 🔴 Critical |
| Unawaited async (fire-and-forget) | 🔴 Critical |
| Deadlock risk | 🔴 Critical |
| Blocking call in async context | 🟡 Major |
| TOCTOU without atomicity | 🟡 Major |
| Unbounded queue/channel | 🟡 Major |
| Missing cancellation handling | 🔵 Minor |

---

## Area 13: Logging & Observability — Deep Checklist

### Log Levels
- **ERROR**: genuine failures needing attention (NOT expected 404s)
- **WARN**: degraded but handled conditions
- **INFO**: significant business events
- **DEBUG**: diagnostic, disabled in production

### Sensitive Data
- No passwords, tokens, API keys, session IDs in logs
- No PII without masking (emails, phone, SSN)
- Request/response bodies sanitized before logging
- Connection strings sanitized

### Structured Logging
- JSON/key-value format (not `console.log` strings)
- Correlation/request IDs for tracing
- Consistent field names across codebase
- Context enrichment (timestamp, service, environment)

### Completeness
- Error logs include stack traces and context
- Business-critical ops logged (auth, payments, data changes)
- Retry attempts and circuit breaker changes logged
- Startup/shutdown logged with config summary

### Volume
- Debug logs guarded in hot paths
- No excessive logging inside loops
- No large objects serialized unnecessarily

| Issue | Severity |
|---|---|
| Passwords/tokens in logs | 🔴 Critical |
| PII logged without masking | 🔴 Critical |
| Missing error context | 🟡 Major |
| Wrong log level | 🟡 Major |
| Unstructured production logs | 🔵 Minor |
| Missing correlation ID | 🔵 Minor |

---

## Area 14: API Design — Deep Checklist

### REST Conventions
- Methods match semantics (GET/POST/PUT/PATCH/DELETE)
- URLs noun-based, plural (`/users`)
- Status codes correct (201, 204, 404, 409)
- Query params for filtering/sorting/pagination

### Request & Response
- Bodies validated with field-level error messages
- Consistent response envelope across endpoints
- Collections paginated with metadata
- Field naming consistent (camelCase OR snake_case)
- No internal IDs or sensitive data in responses

### Error Format (consistent everywhere)
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }
```
No stack traces in production.

### Versioning
- API versioned
- Breaking changes flagged (removed fields, new required fields, type changes)
- Deprecated fields with removal timeline

| Issue | Severity |
|---|---|
| Breaking change without versioning | 🔴 Critical |
| Internal errors exposed | 🔴 Critical |
| Wrong status codes | 🟡 Major |
| Inconsistent response structure | 🟡 Major |
| Missing pagination | 🟡 Major |
| Verb in URL | 🔵 Minor |

---

## Area 15: Configuration — Deep Checklist

### Secrets
- In env vars or secrets manager (not hardcoded)
- `.env` in `.gitignore`, `.env.example` exists
- Different secrets per environment
- Rotatable without code deploy

### Environment Variables
- Validated at startup (fail fast)
- Typed/parsed centrally (not raw strings)
- Sensible documented defaults
- No scattered `if prod` checks

### 12-Factor
- Config externalized | Port configurable | Backing services configurable
- Same code across environments | Logs to stdout

| Issue | Severity |
|---|---|
| Hardcoded secret | 🔴 Critical |
| .env committed | 🔴 Critical |
| Missing env validation | 🟡 Major |
| Magic config numbers | 🟡 Major |
| No .env.example | 🔵 Minor |

---

## Area 16: Dependency Audit — Deep Checklist

### Vulnerabilities
- Known CVEs (run `npm audit`)
- Lockfile committed and current
- Post-install scripts reviewed

### Licenses
- MIT/BSD/Apache: safe | LGPL: careful | GPL/AGPL: incompatible with proprietary

### Health
- Actively maintained | Not single-maintainer | Not deprecated

### Necessity
- Trivially implementable? | Duplicates existing dep? | Using tiny part of large lib?

| Issue | Severity |
|---|---|
| Known CVE | 🔴 Critical |
| GPL in proprietary | 🔴 Critical |
| Post-install scripts | 🟡 Major |
| Unmaintained (2+ years) | 🟡 Major |
| Trivial dep | 🔵 Minor |

---

## Area 17: Migration Safety — Deep Checklist

### Rollback
- Every migration has down/rollback
- Rollback tested | Point-of-no-return documented

### Zero-Downtime
- Add column: nullable or default
- Remove column: code-first (deploy, then migrate)
- Rename: expand-contract
- Constraint: NOT VALID then VALIDATE
- Type change: parallel column → backfill → cutover → drop

### Data Integrity
- Transformations idempotent | Large migrations batched
- Constraints validated against existing data | Post-migration validation
- Orphaned records handled

| Issue | Severity |
|---|---|
| No rollback | 🔴 Critical |
| NOT NULL without default | 🔴 Critical |
| Table lock on large table | 🔴 Critical |
| Column drop without code-first | 🟡 Major |
| Unbatched large migration | 🟡 Major |

---

## Area 18: Accessibility — Deep Checklist
*Frontend/UI code only.*

### Semantic HTML
- `<nav>`, `<main>`, `<button>` (not generic divs)
- Headings h1→h2→h3 | Lists with `<ul>`/`<ol>`

### ARIA
- `aria-label`/`aria-labelledby` on unlabeled elements
- `aria-live` for dynamic regions | `aria-hidden` on decorative

### Keyboard
- All interactive elements Tab-reachable | Logical tab order
- Focus styles visible | Focus trapped in modals, restored on close

### Contrast & Color
- 4.5:1 normal text, 3:1 large | Color not sole info carrier

### Forms
- All inputs labeled | Required fields accessible | Errors via `aria-describedby`

| Issue | Severity |
|---|---|
| Missing alt on image | 🔴 Critical |
| Click on div without role | 🔴 Critical |
| Missing form labels | 🔴 Critical |
| Focus outline removed | 🟡 Major |
| Insufficient contrast | 🟡 Major |
| Missing skip-nav | 🔵 Minor |
