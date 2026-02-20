# Test Writing Patterns & Standards

Deep reference for test structure, naming, factories, mocking, assertions, test types,
anti-patterns, and special cases. Loaded by the testing skill on demand.

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
