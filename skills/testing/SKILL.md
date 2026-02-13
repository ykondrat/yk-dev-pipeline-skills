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

## The Process

### Step 1: Load Context

- Read `spec.md` — requirements that need test coverage
- Read `plan.md` — acceptance criteria become test cases
- Read `review.md` / `fix-plan.md` — any findings about error handling, edge cases, or
  security issues that should have dedicated tests
- Read `pipeline-state.json` — check cross-step impacts from code review
- Read the best practices reference for testing patterns

### Step 2: Detect Test Stack

Check the project for existing test configuration:

```bash
# Check package.json for test runner
cat package.json | grep -E "(vitest|jest|mocha|ava|tap)"

# Check for existing config files
ls vitest.config.* jest.config.* .mocharc.* 2>/dev/null

# Check for existing test files
find src tests __tests__ -name "*.test.*" -o -name "*.spec.*" 2>/dev/null

# Check tsconfig for test paths
cat tsconfig.json | grep -i test
```

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

**4a. Inventory all source files:**
```bash
find src -name "*.ts" -not -name "*.test.*" -not -name "*.d.ts" | sort
```

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

---

## Test Writing Standards

### Test Structure

Every test file follows this structure:
```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';

describe('ModuleName', () => {
  // Setup shared across tests in this group
  beforeEach(() => { /* reset state */ });
  afterEach(() => { /* cleanup */ });

  describe('functionName', () => {
    // Happy path
    it('should {expected behavior} when {condition}', () => {
      // Arrange
      const input = buildInput({ /* specific values */ });

      // Act
      const result = functionName(input);

      // Assert
      expect(result).toEqual(expectedOutput);
    });

    // Edge cases
    it('should handle empty input', () => { /* ... */ });
    it('should handle null/undefined', () => { /* ... */ });
    it('should handle boundary values', () => { /* ... */ });

    // Error cases
    it('should throw ValidationError when input is invalid', () => {
      expect(() => functionName(invalidInput)).toThrow(ValidationError);
    });

    it('should throw NotFoundError when resource does not exist', async () => {
      await expect(functionName('nonexistent')).rejects.toThrow(NotFoundError);
    });
  });
});
```

### Naming Convention

Test names must clearly describe scenario and expected outcome:

```typescript
// ✅ Good — describes what happens and when
it('should return empty array when no users match the filter', () => {});
it('should throw ConflictError when email already exists', () => {});
it('should paginate results with correct cursor when limit is reached', () => {});

// ❌ Bad — vague, doesn't describe behavior
it('should work correctly', () => {});
it('test createUser', () => {});
it('handles errors', () => {});
```

### Test Factories

Create factories for reusable test data:

```typescript
// tests/factories.ts
import { faker } from '@faker-js/faker'; // install if not present

export function buildUser(overrides: Partial<CreateUserInput> = {}): CreateUserInput {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: 'user',
    ...overrides,
  };
}

export function buildPost(overrides: Partial<CreatePostInput> = {}): CreatePostInput {
  return {
    title: faker.lorem.sentence(),
    content: faker.lorem.paragraphs(2),
    published: false,
    ...overrides,
  };
}
```

**Why factories:** Hard-coded test data is fragile and hard to read. Factories generate
realistic data with sensible defaults, and overrides make each test's unique conditions
explicit. If faker is too heavy, use simple inline factories with static defaults.

### Mocking Rules

```typescript
// ✅ Mock external dependencies (DB, APIs, file system)
const mockUserRepo: UserRepository = {
  findById: vi.fn(),
  create: vi.fn(),
  // ...
};

// ✅ Mock time when testing time-dependent logic
vi.useFakeTimers();
vi.setSystemTime(new Date('2025-01-15T10:00:00Z'));

// ✅ Mock environment variables
vi.stubEnv('API_KEY', 'test-key');

// ❌ Don't mock the module under test
// ❌ Don't mock internal implementation details
// ❌ Don't over-mock — if more mocks than real code, test is worthless

// Verify mock calls
expect(mockUserRepo.create).toHaveBeenCalledOnce();
expect(mockUserRepo.create).toHaveBeenCalledWith(
  expect.objectContaining({ email: 'test@example.com' })
);
```

### Assertion Quality

```typescript
// ✅ Specific assertions — test exact values
expect(result.status).toBe(201);
expect(result.body.user.email).toBe('test@example.com');

// ✅ Structural assertions — when exact values vary
expect(result.body).toMatchObject({
  user: expect.objectContaining({
    id: expect.any(String),
    email: 'test@example.com',
    createdAt: expect.any(String),
  }),
});

// ✅ Error assertions — verify type AND message
expect(() => validate(badInput)).toThrow(ValidationError);
expect(() => validate(badInput)).toThrow(/email.*required/i);

// ❌ Weak assertions — don't catch real bugs
expect(result).toBeTruthy();        // too vague
expect(result).toBeDefined();       // too vague
expect(Array.isArray(result)).toBe(true);  // doesn't check contents
```

---

## Test Types — Detailed

### Unit Tests

Test individual functions/classes in isolation. Mock all external dependencies.

```typescript
// Testing a service that depends on a repository
describe('UserService', () => {
  let service: UserService;
  let mockRepo: MockedObject<UserRepository>;

  beforeEach(() => {
    mockRepo = {
      findById: vi.fn(),
      findByEmail: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    };
    service = new UserService(mockRepo);
  });

  describe('getById', () => {
    it('should return user when found', async () => {
      const user = buildUser({ id: '123' });
      mockRepo.findById.mockResolvedValue(user);

      const result = await service.getById('123');

      expect(result).toEqual(user);
      expect(mockRepo.findById).toHaveBeenCalledWith('123');
    });

    it('should throw NotFoundError when user does not exist', async () => {
      mockRepo.findById.mockResolvedValue(null);

      await expect(service.getById('nonexistent'))
        .rejects.toThrow(NotFoundError);
    });
  });
});
```

### Integration Tests

Test real interactions between components. Use real databases (in-memory or containers),
real file system, etc.

```typescript
// Database integration test
describe('UserRepository (integration)', () => {
  let db: Database; // real test database

  beforeAll(async () => {
    db = await setupTestDatabase(); // creates test DB, runs migrations
  });

  afterEach(async () => {
    await db.truncateAll(); // clean between tests
  });

  afterAll(async () => {
    await db.close();
  });

  it('should create and retrieve a user', async () => {
    const repo = new UserRepository(db);
    const input = buildUser({ email: 'integration@test.com' });

    const created = await repo.create(input);
    const found = await repo.findById(created.id);

    expect(found).not.toBeNull();
    expect(found!.email).toBe('integration@test.com');
  });

  it('should throw ConflictError on duplicate email', async () => {
    const repo = new UserRepository(db);
    const input = buildUser({ email: 'dupe@test.com' });

    await repo.create(input);
    await expect(repo.create(input)).rejects.toThrow(ConflictError);
  });
});
```

### E2E / API Tests

Test the full request/response cycle through the HTTP layer.

```typescript
// API test using supertest or similar
import { createApp } from '../src/app.js';

describe('POST /api/users', () => {
  let app: App;

  beforeAll(async () => {
    app = await createApp({ database: testDb });
  });

  afterAll(async () => {
    await app.close();
  });

  it('should create a user and return 201', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: buildUser({ email: 'api@test.com' }),
    });

    expect(response.statusCode).toBe(201);
    expect(response.json().user).toMatchObject({
      email: 'api@test.com',
      id: expect.any(String),
    });
  });

  it('should return 400 with validation errors for invalid input', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: { email: 'not-an-email' },
    });

    expect(response.statusCode).toBe(400);
    expect(response.json().error.code).toBe('VALIDATION_ERROR');
  });

  it('should return 409 when email already exists', async () => {
    const user = buildUser({ email: 'exists@test.com' });
    await app.inject({ method: 'POST', url: '/api/users', payload: user });

    const response = await app.inject({
      method: 'POST', url: '/api/users', payload: user,
    });

    expect(response.statusCode).toBe(409);
  });

  it('should return 401 for unauthenticated requests to protected endpoints', async () => {
    const response = await app.inject({
      method: 'GET',
      url: '/api/users/me',
      // no auth header
    });

    expect(response.statusCode).toBe(401);
  });
});
```

### Edge Case & Error Path Tests

Dedicated tests for boundary conditions and failure modes:

```typescript
describe('Edge Cases', () => {
  // Boundary values
  it('should handle empty string input', () => {});
  it('should handle maximum length input', () => {});
  it('should handle zero quantity', () => {});
  it('should handle negative numbers', () => {});
  it('should handle Number.MAX_SAFE_INTEGER', () => {});

  // Null/undefined
  it('should handle null input gracefully', () => {});
  it('should handle undefined optional fields', () => {});

  // Empty collections
  it('should return empty array when no results match', () => {});
  it('should handle pagination on empty dataset', () => {});

  // Concurrent operations
  it('should handle concurrent creates without duplicate entries', async () => {});

  // Timeout/failure
  it('should timeout after configured duration', async () => {});
  it('should retry on transient failure', async () => {});
  it('should not retry on 4xx errors', async () => {});
});
```

### Performance Benchmarks

Basic benchmarks for critical operations. Not load testing — just establishing baselines.

```typescript
describe('Performance', () => {
  it('should process 1000 items in under 100ms', () => {
    const items = Array.from({ length: 1000 }, (_, i) => buildItem({ id: String(i) }));

    const start = performance.now();
    const result = processItems(items);
    const duration = performance.now() - start;

    expect(result).toHaveLength(1000);
    expect(duration).toBeLessThan(100);
  });

  it('should handle large payload without excessive memory', () => {
    const before = process.memoryUsage().heapUsed;
    const largeInput = generateLargeInput(10_000);
    processLargeInput(largeInput);
    const after = process.memoryUsage().heapUsed;

    const memoryIncrease = (after - before) / 1024 / 1024; // MB
    expect(memoryIncrease).toBeLessThan(50); // less than 50MB
  });
});
```

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
- Never weaken assertions to make a test pass ("oh the status is 500 not 404, let me just change the assertion")
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

## Test Anti-Patterns to Avoid

| Anti-Pattern | Problem | Instead |
|---|---|---|
| **Test the implementation** | Breaks when refactoring | Test observable behavior |
| **Assertion-free test** | Never fails, false confidence | Every test needs specific assertions |
| **God test** | Tests 20 things, hard to debug | One behavior per test |
| **Sleep in tests** | Slow, flaky | Use fake timers or proper async waiting |
| **Shared mutable state** | Tests depend on order | Reset state in beforeEach |
| **Over-mocking** | Tests verify mocks, not code | Mock only external boundaries |
| **Copy-paste tests** | Duplication, maintenance pain | Use factories and helpers |
| **Testing framework code** | Zero value | Test YOUR code, not the ORM/framework |
| **Ignoring/skipping without reason** | Hidden failures | Remove or fix, never `.skip` silently |
| **Brittle snapshots** | Break on any change, approve blindly | Use specific assertions |

---

## Handling Special Cases

### No Database Available for Integration Tests

Use in-memory alternatives:
- **SQLite** in-memory for SQL (via Prisma or Drizzle)
- **mongodb-memory-server** for MongoDB
- **DynamoDB Local** (Docker) for DynamoDB
- **ioredis-mock** for Redis

If none available, mock the repository layer and note in test report that integration
tests are mocked, not real.

### Frontend Components

If the project has frontend code:
- Use `@testing-library/react` (or framework equivalent)
- Test user interactions, not implementation
- Use `jsdom` environment in vitest config
- Test accessibility (roles, labels) in component tests

### Existing Tests

If the project already has tests:
- Run existing tests first — verify they pass
- Don't rewrite existing tests unless they're broken or meaningless
- Add tests for uncovered code
- Match existing conventions (naming, location, patterns)

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
