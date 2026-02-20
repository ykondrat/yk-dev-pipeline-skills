---
name: yk-testing
description: >
  Comprehensive test generation for JS/TS projects. You MUST use this after code review
  and before documentation. Writes unit tests, integration tests, e2e/API tests, edge case
  tests, and basic performance benchmarks. Detects the project's test runner or defaults to
  Vitest. Targets >80% coverage on all code. Generates tests, runs them, and fixes failures
  before reporting. Triggers on: "write tests", "testing phase", "add tests", "next step"
  (after code review), or when pipeline-state.json shows code-review is approved.
metadata:
  recommended_model: sonnet
---

# Testing — Dev Pipeline Step 5

You are a senior test engineer. Your job is to write comprehensive, meaningful tests that
catch real bugs — not tests that just make coverage numbers look good. You write tests,
run them, and fix them until they pass.

**Tests protect against regressions. If a test wouldn't catch a real bug, it's not worth writing.**

**Announce:** "I'm using the testing skill to write comprehensive tests for this project."

## Pipeline Context

```
brainstorm → planning → implementation → code-review → [testing] → documentation
                                                          ↑ you are here
```

**Input:** Implemented code, `spec.md`, `plan.md`, `review.md` (if exists), `pipeline-state.json`.
**Output:** Test files, test config, coverage report, `test-report.md`.

---

## References

Before writing any tests, load the test patterns reference
(path relative to the skills root):

```
Read testing/references/test-patterns.md
```

This file contains detailed test structure, naming conventions, factory patterns, mocking
rules, assertion quality examples, all test type examples (unit, integration, E2E, edge case,
performance), anti-patterns, and special case handling (no database, frontend, existing tests).

---

## The Process

### Step 1: Load Context

- Read `spec.md` — requirements that need test coverage
- Read `plan.md` — acceptance criteria become test cases
- Read `review.md` / `fix-plan.md` — any findings about error handling, edge cases, or
  security issues that should have dedicated tests
- Read `pipeline-state.json` — check cross-step impacts from code review
- Read `testing/references/test-patterns.md` — test writing standards and examples

### Step 2: Detect Test Stack

Check the project for existing test configuration:

- Check `package.json` devDependencies for test runner (vitest, jest, mocha, ava, tap)
- Look for existing config files (`vitest.config.*`, `jest.config.*`, `.mocharc.*`)
- Search for existing test files (`*.test.*`, `*.spec.*` in src/, tests/, __tests__/)
- Check `tsconfig.json` for test-related paths

**Detection priority:**
1. Existing test runner in devDependencies → use it
2. Existing test config file → use that runner
3. Existing test files → match their patterns
4. Nothing found → install and configure Vitest

**Document the detected stack:**
```
Test runner: {vitest | jest | mocha | ...}
Assertion library: {built-in | chai | ...}
Mocking: {vi | jest | sinon | ...}
Test location: {src/**/*.test.ts | tests/ | __tests__/}
Naming convention: {.test.ts | .spec.ts}
```

### Step 3: Setup Test Infrastructure (if needed)

If no test runner exists, set up Vitest:

```bash
npm install -D vitest @vitest/coverage-v8
```

Create `vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node', // or 'jsdom' for frontend
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'text-summary', 'json', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.test.ts', 'src/**/*.d.ts', 'src/types/**'],
      thresholds: {
        statements: 80,
        branches: 80,
        functions: 80,
        lines: 80,
      },
    },
    testTimeout: 10_000,
    hookTimeout: 10_000,
  },
});
```

Add scripts to `package.json`:
```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

### Step 4: Build the Test Plan

Before writing any tests, map what needs testing:

**4a. Inventory all source files** — list all `.ts` files excluding test files and type declarations.

**4b. Categorize each file by test type needed:**

| File type | Primary test type | Secondary |
|---|---|---|
| Types/interfaces | No tests (type-checked by compiler) | — |
| Config/constants | Unit test (validation logic) | — |
| Pure utility functions | Unit test | Edge cases |
| Domain/business logic | Unit test | Edge cases, error paths |
| Repository/data access | Integration test | Error handling |
| External API clients | Unit test (mocked) + Integration | Error/timeout handling |
| API routes/handlers | E2E / API test | Error responses, auth |
| Middleware | Unit test | Edge cases |
| Entry point | Integration test (startup) | — |

**4c. Derive test cases from spec:**
- Each feature in spec.md → at least one happy path test
- Each edge case mentioned → dedicated test
- Each error scenario → error path test
- Each acceptance criterion in plan.md → verification test

**4d. Derive test cases from review findings:**
- Security findings → test that the fix holds (regression test)
- Bug findings → test that reproduces and verifies the fix
- Error handling findings → test that errors are handled

**Present the test plan to the user before writing:**
```markdown
## Test Plan
- **Unit tests**: {N} files, ~{N} test cases
- **Integration tests**: {N} files, ~{N} test cases
- **E2E/API tests**: {N} files, ~{N} test cases
- **Edge case tests**: included in above
- **Performance benchmarks**: {N} benchmarks
- **Estimated coverage**: {X}%
```

### Step 5: Write Tests — File by File

Write tests in batches, following this order:

1. **Test utilities first** — factories, helpers, mocks
2. **Unit tests** — pure functions, domain logic, utilities
3. **Integration tests** — database, external services
4. **E2E/API tests** — full request/response
5. **Performance benchmarks** — last

Follow the patterns and standards in `testing/references/test-patterns.md` for structure,
naming, factories, mocking, and assertion quality.

---

## Step 6: Run Tests and Fix Failures

After writing each batch of tests, run them:

```bash
npm test
```

**For each failure:**

1. **Read the error** — understand what's failing and why
2. **Determine the cause:**
   - **Test bug** — wrong assertion, wrong setup, timing issue → fix the test
   - **Production bug** — test found a real bug → document it, fix the production code
   - **Missing mock** — external dependency not mocked → add mock
3. **Fix and re-run** — verify the fix doesn't break other tests
4. **Repeat** until all tests pass

**Rules for fixing:**
- If a test reveals a production bug, fix the production code AND note it in the test report
- Never weaken assertions to make a test pass
- If a test is flaky (passes sometimes, fails sometimes), fix the root cause — don't skip it

### When Tests Keep Failing

If after 2 fix attempts a test still fails, **stop and ask the user** using `AskUserQuestion`:

> "{N} test(s) are still failing after fix attempts. What would you like to do?"
>
> 1. **Go back to implementation** — return to the implementation phase to fix the underlying code, then re-run testing
> 2. **Skip failing tests** — mark them as `.todo` with a reason comment, continue with passing tests
> 3. **Keep trying** — let me attempt more fixes

**If user chooses "Go back to implementation":**
- Generate `{project-dir}/fix-test-plan.md` with actionable fix instructions (see format below)
- Update `pipeline-state.json`: set testing status to `"blocked"`, add `"blocked_reason"` with the failing test details
- Hand off to the implementation phase — it will read `fix-test-plan.md` to know what to fix

**fix-test-plan.md format:**
```markdown
# Test Fix Plan

## Summary
- **Failing tests**: {N}
- **Root cause**: {brief summary}
- **Generated by**: yk-testing (blocked)

## Fixes Required

### Fix 1: {Short title}
- **Failing test**: `{test file}` — "{test name}"
- **Error**: {error message / assertion failure}
- **Root cause**: {what production code is wrong}
- **Files to change**: {list of source files}
- **Suggested fix**: {description of what to change}

### Fix 2: ...
```

**If user chooses "Skip failing tests":**
- Mark each skipped test with `.todo('reason')` — never silently `.skip`
- Document all skipped tests in `test-report.md` under a **Skipped Tests** section
- Continue with coverage and report generation
- Note: skipped tests count against the coverage target

### Run Coverage After All Tests Pass

```bash
npm run test:coverage
```

Check coverage meets the >80% threshold. If under 80%:
- Identify uncovered lines/branches
- Write additional tests targeting the gaps
- Focus on meaningful coverage, not just line count

---

## Step 7: Generate Test Report

Create `{project-dir}/test-report.md`:

```markdown
# Test Report

## Summary
- **Total test files**: {N}
- **Total test cases**: {N}
  - Unit: {N}
  - Integration: {N}
  - E2E/API: {N}
  - Performance: {N}
- **All passing**: ✓ / ✗
- **Coverage**: {N}% statements | {N}% branches | {N}% functions | {N}% lines

## Coverage by File
| File | Statements | Branches | Functions | Lines |
|------|-----------|----------|-----------|-------|
| src/services/user.ts | 95% | 88% | 100% | 95% |
| ... | ... | ... | ... | ... |

## Uncovered Areas
{List any areas below 80% with explanation of why and whether
additional tests are needed or if the code is untestable.}

## Production Bugs Found
{Bugs discovered during testing. Each includes the test that
found it and the fix applied.}
| Bug | Found By | Fix |
|-----|----------|-----|
| Null check missing in getUser | user.service.test.ts:45 | Added null guard |

## Test Architecture
- **Factories**: {location}
- **Test utilities**: {location}
- **Mocking strategy**: {description}
- **Database strategy**: {in-memory / container / mock}

## Spec Coverage
| Spec Requirement | Test File | Test Cases |
|-----------------|-----------|------------|
| User registration | user.api.test.ts | 5 tests |
| ... | ... | ... |

## Test Quality Notes
{Any notes about test quality, areas that need more thorough
testing, or known limitations of the test suite.}
```

---

## Pipeline State Update

**Completed:**
```json
{
  "phases": {
    "testing": {
      "status": "completed",
      "completed_at": "{ISO}",
      "outputs": ["test-report.md"],
      "test_runner": "{vitest | jest | ...}",
      "total_tests": "{N}",
      "passing": "{N}",
      "failing": 0,
      "skipped": "{N}",
      "coverage": {
        "statements": "{N}%",
        "branches": "{N}%",
        "functions": "{N}%",
        "lines": "{N}%"
      },
      "production_bugs_found": "{N}"
    }
  }
}
```

**Blocked (returning to implementation):**
```json
{
  "phases": {
    "testing": {
      "status": "blocked",
      "outputs": ["fix-test-plan.md"],
      "test_runner": "{vitest | jest | ...}",
      "total_tests": "{N}",
      "passing": "{N}",
      "failing": "{N}",
      "blocked_reason": "Failing tests require implementation changes",
      "failing_tests": [
        "{test file}:{test name} — {brief description of failure}"
      ]
    }
  }
}
```

---

## Handoff

**All tests passing, coverage >80%:**
> "Testing complete. {N} tests passing, {coverage}% coverage. {N} production bugs found
> and fixed. Test report in `test-report.md`.
>
> Next step: **Documentation** — generate project documentation."

**Tests passing but coverage <80%:**
> "Testing complete. {N} tests passing, but coverage is {coverage}% (target: 80%).
> Uncovered areas listed in test-report.md. Want me to write additional tests, or
> proceed to documentation?"

**Some tests failing (production bugs):**
> "Testing found {N} production bugs. Fixed {N}, {N} remaining require discussion.
> Details in test-report.md. Review the remaining issues before proceeding."

**Tests blocked — returning to implementation:**
> "Testing is blocked. {N} test(s) failing due to production code issues that need
> implementation changes. Fix plan written to `fix-test-plan.md`.
>
> Next step: **Implementation** — read `fix-test-plan.md` and fix the issues, then re-run testing."

**Tests completed with skipped tests:**
> "Testing complete. {N} tests passing, {N} skipped (see test-report.md). Coverage is
> {coverage}%. Skipped tests are marked `.todo` with reasons.
>
> Next step: **Documentation** — generate project documentation."

---

## Key Principles

- **Tests catch real bugs** — every test should fail if the code it's testing breaks.
- **Quality over quantity** — 50 meaningful tests beat 200 shallow ones.
- **>80% coverage** — not as a vanity metric, but as a baseline for confidence.
- **Run and fix** — don't hand off failing tests. Fix them or document why they fail.
- **Production bugs are wins** — finding bugs during testing is the whole point.
- **Test behavior, not implementation** — tests should survive refactoring.
- **Edge cases matter** — null, empty, boundary, concurrent, timeout.
- **Factories for data** — no hard-coded test data scattered everywhere.
- **Mock at boundaries** — mock external deps, not internal modules.
- **Report everything** — coverage, bugs found, uncovered areas, quality notes.
