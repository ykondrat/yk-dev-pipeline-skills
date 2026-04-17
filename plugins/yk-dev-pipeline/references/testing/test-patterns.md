# Test Writing Patterns & Standards

Deep reference for test structure, naming, factories, mocking, assertions, test types,
anti-patterns, and special cases. Loaded by the testing skill on demand.

---

## Vitest — Framework Reference

Vitest is the default test runner for this pipeline. It's Jest-compatible, ESM-native,
and uses Vite's transform pipeline for speed.

### Configuration Patterns

```typescript
// vitest.config.ts — Full backend + frontend config
import { defineConfig } from 'vitest/config';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths()],
  test: {
    globals: true,                              // no need to import describe/it/expect
    environment: 'node',                        // 'jsdom' for frontend
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
    exclude: ['node_modules', 'dist', 'e2e'],
    setupFiles: ['./tests/setup.ts'],
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
    pool: 'forks',                              // 'threads' for faster but less isolated
    reporters: ['default'],
    typecheck: {
      enabled: false,                           // enable for type-level testing
    },
  },
});
```

```typescript
// Workspace config — for monorepos or mixed backend/frontend
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  {
    extends: './vitest.config.ts',
    test: {
      name: 'backend',
      environment: 'node',
      include: ['src/server/**/*.test.ts'],
    },
  },
  {
    extends: './vitest.config.ts',
    test: {
      name: 'frontend',
      environment: 'jsdom',
      include: ['src/client/**/*.test.ts'],
      setupFiles: ['./tests/setup-dom.ts'],
    },
  },
]);
```

### Mocking with `vi`

```typescript
import { vi, describe, it, expect, beforeEach, afterEach } from 'vitest';

// --- Function mocks ---
const mockFn = vi.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ id: '1' });    // async
mockFn.mockRejectedValue(new Error('fail'));
mockFn.mockImplementation((x: number) => x * 2);

// --- Module mocks ---
vi.mock('./database', () => ({
  getConnection: vi.fn().mockReturnValue(mockConnection),
}));

// Partial mock (keep real implementations, override specific exports)
vi.mock('./utils', async (importOriginal) => {
  const actual = await importOriginal<typeof import('./utils')>();
  return {
    ...actual,
    sendEmail: vi.fn(),  // mock only this
  };
});

// --- Timer mocks ---
beforeEach(() => vi.useFakeTimers());
afterEach(() => vi.useRealTimers());

it('should debounce calls', () => {
  const callback = vi.fn();
  const debounced = debounce(callback, 300);

  debounced();
  debounced();
  debounced();

  expect(callback).not.toHaveBeenCalled();

  vi.advanceTimersByTime(300);

  expect(callback).toHaveBeenCalledOnce();
});

// --- Date mocks ---
vi.setSystemTime(new Date('2026-01-15T10:00:00Z'));

// --- Environment variable mocks ---
vi.stubEnv('API_KEY', 'test-key');
vi.stubEnv('NODE_ENV', 'test');

// --- Spy on existing methods ---
const logSpy = vi.spyOn(console, 'error').mockImplementation(() => {});
// ... test that logs error ...
expect(logSpy).toHaveBeenCalledWith(expect.stringContaining('failed'));
logSpy.mockRestore();
```

### Assertions — Vitest Matchers

```typescript
// Equality
expect(value).toBe(42);                    // strict ===
expect(obj).toEqual({ id: '1' });          // deep equality
expect(obj).toStrictEqual({ id: '1' });    // deep + no extra properties

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeDefined();
expect(value).toBeUndefined();

// Numbers
expect(value).toBeGreaterThan(10);
expect(value).toBeLessThanOrEqual(100);
expect(value).toBeCloseTo(0.3, 5);         // floating point

// Strings
expect(str).toContain('hello');
expect(str).toMatch(/pattern/);

// Arrays / Iterables
expect(arr).toContain(item);
expect(arr).toHaveLength(3);
expect(arr).toEqual(expect.arrayContaining([1, 2]));

// Objects
expect(obj).toHaveProperty('key');
expect(obj).toHaveProperty('nested.key', 'value');
expect(obj).toMatchObject({ id: expect.any(String) });

// Errors
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(SpecificError);
expect(() => fn()).toThrow(/message/);
await expect(asyncFn()).rejects.toThrow(SpecificError);

// Mock assertions
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith('arg1', expect.any(Number));
expect(mockFn).toHaveBeenLastCalledWith('arg');
expect(mockFn).toHaveBeenNthCalledWith(1, 'first-call-arg');
```

### Test Lifecycle Hooks

```typescript
describe('UserService', () => {
  // Runs once before all tests in this describe
  beforeAll(async () => {
    await setupTestDatabase();
  });

  // Runs before each test
  beforeEach(() => {
    vi.clearAllMocks();  // reset call counts, not implementations
  });

  // Runs after each test
  afterEach(async () => {
    await cleanupTestData();
    vi.restoreAllMocks(); // restore original implementations
  });

  // Runs once after all tests in this describe
  afterAll(async () => {
    await teardownTestDatabase();
  });
});
```

**`clearAllMocks` vs `resetAllMocks` vs `restoreAllMocks`:**
- `clearAllMocks` — resets call history, keeps mock implementations
- `resetAllMocks` — resets call history AND mock implementations (returns `undefined`)
- `restoreAllMocks` — restores original implementations (for `vi.spyOn` only)

### Concurrent Tests

```typescript
// Run independent tests in parallel for speed
describe('independent operations', () => {
  it.concurrent('should process item A', async () => { /* ... */ });
  it.concurrent('should process item B', async () => { /* ... */ });
  it.concurrent('should process item C', async () => { /* ... */ });
});

// Or mark the entire suite
describe.concurrent('parallel suite', () => {
  it('test 1', async () => { /* ... */ });
  it('test 2', async () => { /* ... */ });
});
```

**Warning:** Don't use concurrent tests when tests share mutable state (database, files,
global variables). Use concurrent only for pure/isolated tests.

### Test Filtering & Organization

```bash
# Run specific test file
npx vitest run src/services/user.test.ts

# Run tests matching pattern
npx vitest run --testNamePattern="should create user"

# Run only tests marked with .only (dev workflow)
# it.only('focused test', () => {});  — NEVER commit .only

# Skip tests conditionally
it.skipIf(process.env.CI)('slow test', () => {});
it.runIf(process.env.DATABASE_URL)('needs real DB', () => {});

# Mark tests as todo (placeholder)
it.todo('should handle rate limiting');
```

### Inline Snapshots

```typescript
// Inline snapshots update themselves in the source file
it('should format user for API response', () => {
  const result = formatUser(testUser);

  expect(result).toMatchInlineSnapshot(`
    {
      "email": "test@example.com",
      "id": "123",
      "name": "Test User",
    }
  `);
});

// Update snapshots: npx vitest run --update
```

---

## Test Structure

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

## Naming Convention

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

## Test Factories

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

## Mocking Rules

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

## Assertion Quality

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

If the project has frontend code, use `@testing-library/react` (or Vue/Svelte equivalent)
with `jsdom` environment. Also read `implementation/references/frameworks-frontend.md`
Section 7 for full framework-specific examples.

#### Vitest Config for Frontend

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./tests/setup.ts'],
    css: { modules: { classNameStrategy: 'non-scoped' } },
  },
});
```

```typescript
// tests/setup.ts
import '@testing-library/jest-dom/vitest';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => cleanup());
```

#### Component Testing Patterns

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('SearchInput', () => {
  it('should call onSearch with debounced value', async () => {
    const onSearch = vi.fn();
    const user = userEvent.setup();
    render(<SearchInput onSearch={onSearch} debounceMs={100} />);

    await user.type(screen.getByRole('searchbox'), 'hello');

    // Wait for debounce
    await waitFor(() => {
      expect(onSearch).toHaveBeenCalledWith('hello');
    });
  });

  it('should show loading state during search', async () => {
    render(<SearchInput isLoading />);
    expect(screen.getByRole('progressbar')).toBeInTheDocument();
  });
});
```

**Key queries** (in order of preference):
1. `getByRole` — buttons, inputs, headings, links (best for accessibility)
2. `getByLabelText` — form fields (tests label association)
3. `getByText` — non-interactive text content
4. `getByTestId` — last resort only (not what users see)

#### Hook Testing

```typescript
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './use-counter';

describe('useCounter', () => {
  it('should increment counter', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => result.current.increment());

    expect(result.current.count).toBe(1);
  });

  it('should reset to initial value', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(5);
  });
});
```

#### Snapshot Testing Guidance

- **Use sparingly** — Snapshots break on any change and get approved blindly
- **Good for**: Small, stable UI components (icons, badges, formatted text)
- **Bad for**: Large components, dynamic content, components under active development
- **Prefer inline snapshots** over file snapshots (easier to review)
- **Never**: Snapshot entire pages or complex component trees

```typescript
// ✅ Good — small, stable component
it('should render badge with correct label', () => {
  const { container } = render(<Badge variant="success">Active</Badge>);
  expect(container.firstChild).toMatchInlineSnapshot(`
    <span class="badge badge-success">Active</span>
  `);
});

// ❌ Bad — large, frequently changing component
it('should render user profile', () => {
  const { container } = render(<UserProfile user={mockUser} />);
  expect(container).toMatchSnapshot(); // will break constantly
});
```

#### API Mocking with MSW (Mock Service Worker)

```typescript
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';

const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json({
      data: [{ id: '1', name: 'Alice' }, { id: '2', name: 'Bob' }],
    });
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ data: { id: '3', ...body } }, { status: 201 });
  }),
  http.get('/api/users/:id', ({ params }) => {
    if (params.id === 'not-found') {
      return HttpResponse.json(
        { error: { code: 'NOT_FOUND' } },
        { status: 404 },
      );
    }
    return HttpResponse.json({ data: { id: params.id, name: 'Alice' } });
  }),
];

const server = setupServer(...handlers);

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Override handlers per test
it('should show error state on 500', async () => {
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json(null, { status: 500 });
    }),
  );

  render(<UserList />);
  expect(await screen.findByText(/something went wrong/i)).toBeInTheDocument();
});
```

**Why MSW over vi.mock**: MSW intercepts at the network layer, so your components use their
real fetch/axios code. Tests are more realistic and less brittle than module-level mocks.

#### Accessibility Testing in Components

```typescript
// Test that interactive elements are accessible
it('should have accessible form fields', () => {
  render(<LoginForm />);

  // All inputs have labels
  expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
  expect(screen.getByLabelText(/password/i)).toBeInTheDocument();

  // Submit button is identifiable
  expect(screen.getByRole('button', { name: /sign in/i })).toBeInTheDocument();
});

it('should show accessible error messages', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.click(screen.getByRole('button', { name: /sign in/i }));

  // Errors announced to screen readers
  const alerts = screen.getAllByRole('alert');
  expect(alerts.length).toBeGreaterThan(0);
});
```

#### Frontend Test Anti-Patterns

| Anti-Pattern | Problem | Instead |
|---|---|---|
| `container.querySelector('.btn')` | Tied to CSS class, not behavior | `screen.getByRole('button')` |
| `fireEvent.change(input, ...)` | Doesn't simulate real typing | `userEvent.type(input, ...)` |
| Testing state values directly | Implementation detail | Test visible output |
| Wrapping every assertion in `act()` | Misunderstanding React batching | `userEvent` handles this |
| `await new Promise(r => setTimeout(r, 500))` | Slow, flaky | `waitFor()` or `findBy*` queries |
| Mocking child components | Over-isolation | Render real children, mock network |

### Existing Tests

If the project already has tests:
- Run existing tests first — verify they pass
- Don't rewrite existing tests unless they're broken or meaningless
- Add tests for uncovered code
- Match existing conventions (naming, location, patterns)
