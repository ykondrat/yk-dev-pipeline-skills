# JavaScript / TypeScript / Node.js Best Practices

A comprehensive reference for writing production-quality JS/TS code. Read this before
implementing any task. Each section explains not just WHAT to do, but WHY — so you can
apply the principles even in situations not explicitly covered.

---

## Table of Contents

1. [TypeScript Patterns](#1-typescript-patterns)
2. [Node.js Patterns](#2-nodejs-patterns)
3. [Modern JavaScript](#3-modern-javascript)
4. [Security](#4-security)
5. [Performance](#5-performance)
6. [Project Structure](#6-project-structure)
7. [Design Patterns](#7-design-patterns)
8. [Algorithms & Data Structures](#8-algorithms--data-structures)

---

## 1. TypeScript Patterns

### Strict Mode — Always

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

**Why:** `strict: true` enables a family of checks (`strictNullChecks`, `strictFunctionTypes`,
`strictBindCallApply`, `noImplicitAny`, `noImplicitThis`, `alwaysStrict`) that catch real bugs
at compile time. `noUncheckedIndexedAccess` is especially valuable — it makes array/object
index access return `T | undefined` instead of `T`, forcing you to handle the case where the
key doesn't exist. This alone prevents a class of runtime errors.

### Avoid `any` — Use `unknown` Instead

```typescript
// ❌ Bad — any disables all type checking downstream
function parseInput(data: any) {
  return data.name.toUpperCase(); // No error, but will crash if data has no name
}

// ✅ Good — unknown forces you to validate before using
function parseInput(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'name' in data) {
    const { name } = data as { name: unknown };
    if (typeof name === 'string') {
      return name.toUpperCase();
    }
  }
  throw new InvalidInputError('Expected object with string "name" property');
}
```

**Why:** `any` is a trapdoor — it silently disables type checking for everything it touches,
and the unsafety propagates through every function that uses the value. `unknown` is the
type-safe counterpart: it accepts any value but forces you to narrow it before using it.
The only legitimate uses of `any` are in type assertions for library interop where the
types are genuinely wrong, and they should always have a comment explaining why.

### Discriminated Unions for State Modeling

```typescript
// ❌ Bad — impossible states are representable
interface ApiResponse {
  loading: boolean;
  error: Error | null;
  data: User[] | null;
}
// loading: false, error: Error, data: User[] — what does this mean?

// ✅ Good — each state is explicit and exhaustive
type ApiResponse =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: User[] };

function handleResponse(response: ApiResponse) {
  switch (response.status) {
    case 'idle':
      return null;
    case 'loading':
      return <Spinner />;
    case 'error':
      return <ErrorMessage error={response.error} />;
    case 'success':
      return <UserList users={response.data} />;
    // TypeScript errors if you miss a case (with noImplicitReturns)
  }
}
```

**Why:** Discriminated unions make impossible states unrepresentable. Instead of having to
reason about which combinations of booleans and nulls are valid, each state is a single
object with exactly the fields that make sense for that state. TypeScript's narrowing
ensures you can only access `error` in the error state and `data` in the success state.
This eliminates an entire class of bugs where code accesses data that doesn't exist in
the current state.

### Type Guards for Safe Narrowing

```typescript
// Custom type guard — use when you need to narrow in reusable ways
function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

// Assertion function — use when failure should throw
function assertDefined<T>(
  value: T,
  message: string
): asserts value is NonNullable<T> {
  if (value === null || value === undefined) {
    throw new Error(message);
  }
}

// Usage
const users = [getUser(1), getUser(2), getUser(3)];
const validUsers = users.filter(isNonNullable); // Type: User[]

const config = loadConfig();
assertDefined(config, 'Config file not found'); // narrows to NonNullable<Config>
```

**Why:** Type guards let you create reusable narrowing logic that TypeScript understands.
Without them, you end up with repeated `if (x !== null)` checks or unsafe type assertions.
Assertion functions (`asserts`) are particularly useful at function boundaries — they narrow
the type AND throw if the assertion fails, combining validation and type narrowing in one step.

### Generics — Use Meaningfully, Not Everywhere

```typescript
// ❌ Bad — generic adds no value, just complexity
function getLength<T extends { length: number }>(arr: T): number {
  return arr.length;
}

// ✅ Good — generic preserves the specific type through transformation
function first<T>(arr: readonly T[]): T | undefined {
  return arr[0];
}

// ✅ Good — generic creates a relationship between parameters and return type
function groupBy<T, K extends string | number>(
  items: readonly T[],
  keyFn: (item: T) => K
): Record<K, T[]> {
  return items.reduce((acc, item) => {
    const key = keyFn(item);
    (acc[key] ??= []).push(item);
    return acc;
  }, {} as Record<K, T[]>);
}

// ✅ Good — generic constrains related parameters
function merge<T extends object>(target: T, source: Partial<T>): T {
  return { ...target, ...source };
}
```

**Why:** Generics are valuable when they create a relationship — between input and output,
between multiple parameters, or between a function and its caller. If a generic parameter
is only used once, or if replacing it with a concrete type loses nothing, it's adding
complexity without value. The test: "Does this generic help TypeScript infer something
useful for the caller?" If not, remove it.

### Utility Types — Know the Built-ins

```typescript
// Partial<T> — all properties optional (good for update payloads)
type UserUpdate = Partial<User>;

// Required<T> — all properties required
type CompleteConfig = Required<Config>;

// Pick<T, K> / Omit<T, K> — select or exclude properties
type PublicUser = Omit<User, 'password' | 'email'>;
type Credentials = Pick<User, 'email' | 'password'>;

// Record<K, V> — typed key-value maps
type FeatureFlags = Record<string, boolean>;

// ReturnType<T> / Parameters<T> — extract from function signatures
type QueryResult = ReturnType<typeof executeQuery>;

// Extract<T, U> / Exclude<T, U> — filter union types
type NumericEvent = Extract<Event, { value: number }>;

// Readonly<T> — immutable version
type FrozenConfig = Readonly<Config>;

// Awaited<T> — unwrap Promise types
type ResolvedData = Awaited<ReturnType<typeof fetchData>>;
```

**Why:** Built-in utility types eliminate entire categories of manual type definitions.
They're also more maintainable — `Omit<User, 'password'>` automatically updates when
`User` changes, while a manually defined `PublicUser` would need manual updating.
Know these cold; they're the vocabulary of TypeScript type manipulation.

### `as const` and Template Literal Types

```typescript
// as const — create narrow literal types from values
const ROUTES = {
  home: '/',
  users: '/users',
  user: '/users/:id',
} as const;

type Route = typeof ROUTES[keyof typeof ROUTES]; // '/' | '/users' | '/users/:id'

// Template literal types — type-safe string patterns
type EventName = `on${Capitalize<'click' | 'hover' | 'focus'>}`;
// 'onClick' | 'onHover' | 'onFocus'

// Combine with generics for powerful patterns
type Getter<T extends string> = `get${Capitalize<T>}`;
type Setter<T extends string> = `set${Capitalize<T>}`;
```

**Why:** `as const` bridges the gap between runtime values and compile-time types,
letting you define constants once and derive types from them. This is the "single source
of truth" principle applied to types — no chance of the type and the value getting out
of sync. Template literal types extend this to string patterns, enabling type-safe event
names, route patterns, and similar conventions.

---

## 2. Node.js Patterns

### Async/Await — Always, Correctly

```typescript
// ❌ Bad — unhandled promise rejection (crashes Node 15+)
async function processFile(path: string) {
  const data = fs.promises.readFile(path, 'utf-8'); // missing await!
  return JSON.parse(data); // data is a Promise, not a string
}

// ❌ Bad — catching too broadly, losing context
async function fetchUser(id: string) {
  try {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json();
    return processUserData(data);
  } catch (error) {
    console.log('Something went wrong'); // which step failed? what error?
    return null; // caller has no idea what happened
  }
}

// ✅ Good — specific error handling, proper propagation
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new HttpError(response.status, `Failed to fetch user ${id}`);
  }

  const data: unknown = await response.json();
  return parseUser(data); // throws ValidationError if invalid
}
```

**Why:** Async/await is syntactic sugar over Promises, but it changes error handling
semantics significantly. Missing `await` silently creates a detached promise whose
rejection crashes the process. Catching too broadly hides the actual failure. The
principle: handle errors at the level where you can do something meaningful about them,
and let everything else propagate with full context.

### Error Handling — Typed, Structured, Intentional

```typescript
// Define an error hierarchy for your domain
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly cause?: Error
  ) {
    super(message);
    this.name = this.constructor.name;
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} not found: ${id}`, 'NOT_FOUND', 404);
  }
}

class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly field: string,
    public readonly value: unknown
  ) {
    super(message, 'VALIDATION_ERROR', 400);
  }
}

class ExternalServiceError extends AppError {
  constructor(service: string, cause: Error) {
    super(`${service} request failed: ${cause.message}`, 'EXTERNAL_ERROR', 502, cause);
  }
}

// Use at boundaries
async function getUserHandler(req: Request): Promise<Response> {
  try {
    const user = await userService.getById(req.params.id);
    return Response.json(user);
  } catch (error) {
    if (error instanceof AppError) {
      return Response.json(
        { error: error.code, message: error.message },
        { status: error.statusCode }
      );
    }
    // Unknown errors — log full detail, return generic message
    logger.error('Unhandled error in getUserHandler', { error });
    return Response.json(
      { error: 'INTERNAL_ERROR', message: 'An unexpected error occurred' },
      { status: 500 }
    );
  }
}
```

**Why:** Unstructured error handling (bare `throw new Error('something went wrong')`)
makes debugging hard and error responses inconsistent. A typed error hierarchy lets you:
(1) distinguish between expected errors (not found, validation) and unexpected ones (bugs),
(2) map errors to HTTP status codes at the boundary, (3) include machine-readable error
codes for API consumers, (4) chain causes for debugging without exposing internals to users.
The `cause` property (ES2022) preserves the original error while wrapping it in domain context.

### Process Signal Handling

```typescript
// Graceful shutdown — critical for production services
function setupGracefulShutdown(
  server: http.Server,
  cleanup: () => Promise<void>
) {
  let isShuttingDown = false;

  const shutdown = async (signal: string) => {
    if (isShuttingDown) return;
    isShuttingDown = true;

    logger.info(`Received ${signal}, starting graceful shutdown...`);

    // Stop accepting new connections
    server.close();

    // Set a hard deadline
    const forceTimeout = setTimeout(() => {
      logger.error('Graceful shutdown timed out, forcing exit');
      process.exit(1);
    }, 30_000);

    try {
      // Run cleanup (close DB connections, flush logs, etc.)
      await cleanup();
      clearTimeout(forceTimeout);
      logger.info('Graceful shutdown complete');
      process.exit(0);
    } catch (error) {
      logger.error('Error during shutdown cleanup', { error });
      process.exit(1);
    }
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));

  // Catch unhandled errors — log and exit (don't try to recover)
  process.on('unhandledRejection', (reason) => {
    logger.fatal('Unhandled promise rejection', { reason });
    process.exit(1);
  });

  process.on('uncaughtException', (error) => {
    logger.fatal('Uncaught exception', { error });
    process.exit(1);
  });
}
```

**Why:** In production, processes receive SIGTERM (from orchestrators like Kubernetes or
Docker) and SIGINT (from Ctrl+C). Without handling these, the process dies immediately,
potentially leaving database connections open, in-flight requests dropped, and data
inconsistent. Graceful shutdown stops accepting new work, finishes in-flight work, cleans
up resources, then exits. The hard timeout prevents hanging forever if cleanup gets stuck.
Unhandled rejections and uncaught exceptions should always crash the process — a process
in an unknown state is more dangerous than a restarted process.

### Streams — Use for Large Data

```typescript
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { Transform } from 'node:stream';

// ❌ Bad — loads entire file into memory
async function processLargeFile(path: string) {
  const content = await fs.promises.readFile(path, 'utf-8');
  const lines = content.split('\n'); // entire file in memory
  return lines.filter(line => line.includes('ERROR'));
}

// ✅ Good — processes line by line, constant memory
async function processLargeFile(inputPath: string, outputPath: string) {
  const filterErrors = new Transform({
    transform(chunk, encoding, callback) {
      const lines = chunk.toString().split('\n');
      const errors = lines.filter((line: string) => line.includes('ERROR'));
      if (errors.length > 0) {
        callback(null, errors.join('\n') + '\n');
      } else {
        callback();
      }
    },
  });

  await pipeline(
    createReadStream(inputPath, { encoding: 'utf-8' }),
    filterErrors,
    createWriteStream(outputPath)
  );
}
```

**Why:** Node.js streams process data in chunks, keeping memory usage constant regardless
of input size. `readFile` loads the entire file into a single Buffer — fine for config
files, catastrophic for log files or data dumps. `pipeline` (from `node:stream/promises`)
handles backpressure automatically and cleans up resources if any stream errors. Rule of
thumb: if the data could be larger than ~50MB, use streams.

### Worker Threads — CPU-Intensive Work

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'node:worker_threads';

// Heavy computation blocks the event loop — move it to a worker
if (isMainThread) {
  async function hashPasswords(passwords: string[]): Promise<string[]> {
    return new Promise((resolve, reject) => {
      const worker = new Worker(new URL(import.meta.url), {
        workerData: { passwords },
      });
      worker.on('message', resolve);
      worker.on('error', reject);
    });
  }
} else {
  // Worker thread
  const { passwords } = workerData as { passwords: string[] };
  const hashed = passwords.map(pw => bcrypt.hashSync(pw, 12));
  parentPort?.postMessage(hashed);
}
```

**Why:** Node.js is single-threaded for JavaScript execution. CPU-heavy work (hashing,
image processing, complex calculations) blocks the event loop, making the entire process
unresponsive to I/O. Worker threads run JavaScript in parallel OS threads, keeping the
main thread free for I/O. Use them for any computation that takes >100ms. For simpler
cases, consider `node:child_process` with `fork()`.

---

## 3. Modern JavaScript

### ESM — Use It by Default

```json
// package.json
{
  "type": "module"
}
```

```typescript
// ✅ Named imports — tree-shakeable, explicit
import { readFile, writeFile } from 'node:fs/promises';

// ✅ Use node: protocol for built-in modules — disambiguates from npm packages
import { join } from 'node:path';
import { setTimeout } from 'node:timers/promises';

// ✅ Dynamic import for conditional/lazy loading
const chalk = await import('chalk');
```

**Why:** ESM (ECMAScript Modules) is the standard module system for JavaScript. CJS
(`require`) is legacy. ESM enables static analysis for tree-shaking (removing unused code
from bundles), has cleaner semantics (no circular dependency footguns), and is the only
module system supported in browsers. The `node:` protocol explicitly marks built-in modules,
preventing confusion with npm packages that might share the same name.

### Structured Clone — Deep Copy Without Hacks

```typescript
// ❌ Bad — JSON round-trip loses dates, undefined, functions, Maps, Sets
const copy = JSON.parse(JSON.stringify(original));

// ❌ Bad — spread only does shallow copy
const copy = { ...original }; // nested objects are still references

// ✅ Good — true deep copy preserving types
const copy = structuredClone(original);
// Works with: Date, RegExp, Map, Set, ArrayBuffer, Error, etc.
// Does NOT work with: Functions, DOM nodes, Symbols
```

**Why:** `structuredClone` (available since Node 17) is the correct way to deep copy
objects. The JSON hack silently corrupts `Date` objects (become strings), drops `undefined`
values, `Map`/`Set` (become `{}`), and throws on circular references. Spread only copies
one level deep. `structuredClone` handles all built-in types correctly and supports
circular references.

### AbortController — Cancellable Operations

```typescript
// Timeout with cancellation
async function fetchWithTimeout(url: string, timeoutMs: number): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } finally {
    clearTimeout(timeoutId);
  }
}

// Cancellable batch processing
async function processBatch(
  items: string[],
  signal?: AbortSignal
): Promise<Result[]> {
  const results: Result[] = [];

  for (const item of items) {
    signal?.throwIfAborted(); // Check before each item
    results.push(await processItem(item));
  }

  return results;
}

// Usage — cancel on shutdown
const controller = new AbortController();
process.on('SIGTERM', () => controller.abort());

await processBatch(items, controller.signal);
```

**Why:** `AbortController` is the standard mechanism for cancelling async operations in
JavaScript. It works with `fetch`, Node.js streams, and any custom async code. Without
cancellation, long-running operations continue consuming resources after the result is no
longer needed (e.g., the user navigated away, the request timed out, the process is
shutting down). Pass `AbortSignal` into any function that might run for a while.

### Optional Chaining & Nullish Coalescing — Safe Access

```typescript
// Optional chaining — short-circuit on null/undefined
const city = user?.address?.city; // undefined if any step is nullish

// Nullish coalescing — default only for null/undefined (NOT for 0, '', false)
const port = config.port ?? 3000; // uses 3000 only if port is null/undefined
const name = user.name ?? 'Anonymous';

// ❌ Bad — || treats 0, '', false as falsy
const port = config.port || 3000; // port 0 becomes 3000!
const verbose = options.verbose || true; // can never be false!

// Optional chaining with method calls
const result = api?.getData?.();

// Optional chaining with bracket access
const value = data?.[dynamicKey];
```

**Why:** Optional chaining eliminates verbose null-check chains. Nullish coalescing (`??`)
is the correct default operator for most cases — `||` was the old workaround but it treats
all falsy values (0, empty string, false) as "missing," which is usually wrong. The rule:
use `??` unless you specifically want to replace ALL falsy values.

### Promise Combinators — Concurrent Operations

```typescript
// Promise.all — fail fast, all must succeed
const [users, posts, settings] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
  fetchSettings(),
]);

// Promise.allSettled — get results of all, even if some fail
const results = await Promise.allSettled([
  fetchUsers(),
  fetchPosts(),
  fetchSettings(),
]);

for (const result of results) {
  if (result.status === 'fulfilled') {
    processData(result.value);
  } else {
    logger.warn('Fetch failed', { reason: result.reason });
  }
}

// Promise.race — first to resolve/reject wins (useful for timeouts)
const data = await Promise.race([
  fetchData(),
  timeout(5000), // rejects after 5s
]);

// Promise.any — first to succeed wins (ignores rejections)
const fastest = await Promise.any([
  fetchFromPrimary(),
  fetchFromFallback(),
  fetchFromCache(),
]);
```

**Why:** Sequential `await` is a common performance mistake. If three API calls are
independent, running them sequentially takes 3x as long as running them concurrently
with `Promise.all`. Choose the combinator based on your requirements: `all` when every
result is needed, `allSettled` when partial results are acceptable, `race` for timeouts,
`any` for fallback strategies.

### Using `Map` and `Set` Over Plain Objects

```typescript
// ❌ Bad — objects as maps have footguns
const cache: Record<string, Data> = {};
cache['constructor']; // returns Object.constructor, not undefined!
cache['__proto__'];   // prototype pollution risk

// ✅ Good — Map is purpose-built for key-value storage
const cache = new Map<string, Data>();
cache.set('key', data);
cache.get('key');
cache.has('key');
cache.delete('key');
cache.size; // O(1), not Object.keys(obj).length

// ✅ Set for unique collections
const uniqueIds = new Set<string>();
uniqueIds.add(id);
uniqueIds.has(id); // O(1) lookup vs array.includes O(n)
```

**Why:** Plain objects inherit from `Object.prototype`, which means keys like `constructor`,
`toString`, `__proto__` collide with inherited properties. `Map` has no inherited keys,
preserves insertion order, has O(1) `.size`, and supports any key type (not just strings).
`Set` is O(1) for `has`/`add`/`delete`, while `Array.includes` is O(n). Use `Map`/`Set`
whenever you need a collection; use plain objects for structured data with known keys.

---

## 4. Security

### Input Validation — Trust Nothing

```typescript
import { z } from 'zod'; // or similar validation library

// Define schemas at the boundary
const CreateUserSchema = z.object({
  name: z.string().min(1).max(200),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;

// Validate at the entry point — never deeper
async function createUserHandler(req: Request): Promise<Response> {
  const parsed = CreateUserSchema.safeParse(await req.json());

  if (!parsed.success) {
    return Response.json(
      { error: 'VALIDATION_ERROR', issues: parsed.error.issues },
      { status: 400 }
    );
  }

  // From here on, parsed.data is fully typed and validated
  const user = await userService.create(parsed.data);
  return Response.json(user, { status: 201 });
}
```

**Why:** Every external input — HTTP requests, file contents, environment variables,
database results, API responses — can contain unexpected data. Validating at the boundary
(where external data enters your system) creates a trust perimeter: code inside the
perimeter can assume data is valid. Zod (or similar) combines validation with TypeScript
type inference, so you get runtime safety and compile-time types from a single schema
definition.

### Secrets Management

```typescript
// ❌ NEVER hardcode secrets
const API_KEY = 'sk-1234567890';

// ❌ NEVER commit .env files
// .env should always be in .gitignore

// ✅ Load from environment with validation
const ConfigSchema = z.object({
  DATABASE_URL: z.string().url(),
  API_KEY: z.string().min(1),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
});

function loadConfig() {
  const result = ConfigSchema.safeParse(process.env);
  if (!result.success) {
    console.error('Invalid environment configuration:');
    console.error(result.error.format());
    process.exit(1);
  }
  return result.data;
}

// ✅ Never log secrets
function sanitizeForLogging(config: Record<string, unknown>): Record<string, unknown> {
  const sensitive = ['password', 'secret', 'token', 'key', 'authorization'];
  return Object.fromEntries(
    Object.entries(config).map(([k, v]) =>
      sensitive.some(s => k.toLowerCase().includes(s))
        ? [k, '[REDACTED]']
        : [k, v]
    )
  );
}
```

**Why:** Hardcoded secrets end up in version control, logs, error messages, and stack
traces. Environment variables are the standard mechanism for injecting secrets at runtime.
Validating them at startup (fail fast) is better than crashing later when a missing key
is first used. Never log, serialize, or include secrets in error responses — attackers
commonly exploit error messages and logs to extract credentials.

### SQL/NoSQL Injection Prevention

```typescript
// ❌ NEVER interpolate user input into queries
const user = await db.query(`SELECT * FROM users WHERE id = '${userId}'`);

// ✅ Use parameterized queries
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// ✅ Use an ORM/query builder — they parameterize automatically
const user = await prisma.user.findUnique({ where: { id: userId } });

// ✅ For dynamic queries, use allowlists
const ALLOWED_SORT_FIELDS = ['name', 'created_at', 'email'] as const;
type SortField = typeof ALLOWED_SORT_FIELDS[number];

function buildQuery(sortBy: string): string {
  if (!ALLOWED_SORT_FIELDS.includes(sortBy as SortField)) {
    throw new ValidationError(`Invalid sort field: ${sortBy}`);
  }
  return `SELECT * FROM users ORDER BY ${sortBy}`;
}
```

**Why:** Injection attacks remain the most common web vulnerability. Any place where user
input is concatenated into a query string (SQL, NoSQL, LDAP, OS commands) is a potential
injection point. Parameterized queries separate data from code, making injection impossible.
ORMs handle this automatically but can be bypassed with raw query methods — be extra
careful with those.

### Path Traversal Prevention

```typescript
import { resolve, join, relative } from 'node:path';

// ❌ Dangerous — user can escape the directory
function readUserFile(filename: string) {
  return fs.readFile(`./uploads/${filename}`);
  // filename = "../../etc/passwd" → reads /etc/passwd
}

// ✅ Safe — resolve and verify the path stays in bounds
function readUserFile(filename: string): Promise<Buffer> {
  const baseDir = resolve('./uploads');
  const filePath = resolve(baseDir, filename);

  // Verify the resolved path is still within the base directory
  const relativePath = relative(baseDir, filePath);
  if (relativePath.startsWith('..') || resolve(filePath) !== filePath) {
    throw new ValidationError(`Invalid file path: ${filename}`);
  }

  return fs.promises.readFile(filePath);
}
```

**Why:** Path traversal (directory traversal) lets attackers read or write files outside
the intended directory by using `../` sequences or absolute paths. Always resolve the
final path and verify it's still within the expected directory. This applies to any
operation that takes a user-supplied filename or path: file reads, writes, deletions,
directory listings, and ZIP extraction.

### Dependency Security

```bash
# Audit dependencies regularly
npm audit

# Use lockfiles — always commit them
# package-lock.json (npm), yarn.lock (yarn), pnpm-lock.yaml (pnpm)

# Pin exact versions for critical dependencies
npm install --save-exact critical-package
```

```typescript
// Be cautious with dynamic imports of user-specified modules
// ❌ Never do this
const mod = await import(userInput);

// ✅ Allowlist specific modules
const ALLOWED_PLUGINS = new Map([
  ['markdown', () => import('./plugins/markdown.js')],
  ['csv', () => import('./plugins/csv.js')],
]);

function loadPlugin(name: string) {
  const loader = ALLOWED_PLUGINS.get(name);
  if (!loader) throw new Error(`Unknown plugin: ${name}`);
  return loader();
}
```

**Why:** Your dependencies are part of your attack surface. A compromised or malicious
dependency executes with your application's full permissions. Lockfiles ensure reproducible
installs (preventing supply chain attacks via unpinned versions). `npm audit` catches known
vulnerabilities. Never dynamically import user-specified module names — this is equivalent
to arbitrary code execution.

---

## 5. Performance

### Event Loop — Don't Block It

```typescript
// ❌ Bad — blocks the event loop for the duration of the computation
function fibonacci(n: number): number {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

app.get('/fib/:n', (req, res) => {
  res.json({ result: fibonacci(Number(req.params.n)) }); // blocks ALL requests
});

// ✅ Good — break computation into chunks
async function fibonacciAsync(n: number): Promise<number> {
  if (n <= 1) return n;

  // Yield to event loop periodically
  if (n % 10 === 0) {
    await new Promise(resolve => setImmediate(resolve));
  }

  return await fibonacciAsync(n - 1) + await fibonacciAsync(n - 2);
}

// ✅ Better — use worker threads for truly heavy computation
// (See Node.js Patterns → Worker Threads section)
```

**Why:** Node.js processes all I/O callbacks, timers, and request handlers on a single
thread (the event loop). If one handler blocks for 500ms with synchronous computation,
ALL other requests wait 500ms. Even 50ms of blocking is noticeable under load. For
CPU-heavy work: use worker threads, break into chunks with `setImmediate`, or use
streaming/incremental algorithms.

### Memory — Avoid Leaks, Manage Growth

```typescript
// ❌ Memory leak — unbounded cache
const cache: Map<string, Data> = new Map();
function getCached(key: string, fetcher: () => Promise<Data>): Promise<Data> {
  if (cache.has(key)) return Promise.resolve(cache.get(key)!);
  return fetcher().then(data => { cache.set(key, data); return data; });
}
// Cache grows forever, eventually OOM

// ✅ Bounded cache with LRU eviction
class LRUCache<K, V> {
  private cache = new Map<K, V>();

  constructor(private maxSize: number) {}

  get(key: K): V | undefined {
    const value = this.cache.get(key);
    if (value !== undefined) {
      // Move to end (most recently used)
      this.cache.delete(key);
      this.cache.set(key, value);
    }
    return value;
  }

  set(key: K, value: V): void {
    this.cache.delete(key); // remove if exists (refresh position)
    this.cache.set(key, value);
    if (this.cache.size > this.maxSize) {
      // Delete oldest (first entry)
      const oldest = this.cache.keys().next().value;
      if (oldest !== undefined) this.cache.delete(oldest);
    }
  }
}

// ✅ WeakMap/WeakRef for metadata that shouldn't prevent GC
const metadata = new WeakMap<object, Metadata>();
```

**Why:** Memory leaks in Node.js are insidious — the process slowly consumes more memory
until it's killed by the OS or OOM killer. Common sources: unbounded caches, event listener
accumulation, closures capturing large objects, and global state that grows with requests.
Always bound any collection that grows with usage. `WeakMap`/`WeakRef` are useful for
metadata attached to objects — they don't prevent garbage collection.

### Lazy Loading and Code Splitting

```typescript
// ✅ Dynamic import for features used conditionally
async function generatePDF(data: ReportData) {
  // puppeteer is huge — only load when needed
  const { default: puppeteer } = await import('puppeteer');
  const browser = await puppeteer.launch();
  // ...
}

// ✅ Lazy initialization pattern
class DatabasePool {
  private pool: Pool | null = null;

  async getPool(): Promise<Pool> {
    if (!this.pool) {
      const { createPool } = await import('mysql2/promise');
      this.pool = createPool(this.config);
    }
    return this.pool;
  }
}
```

**Why:** Not everything needs to be loaded at startup. Large dependencies (PDF generators,
image processors, ML libraries) increase startup time and memory usage even if they're
rarely used. Dynamic `import()` loads modules on demand. This is especially important for
serverless functions where cold start time directly affects user experience, and for CLI
tools where most commands don't need most dependencies.

### Caching Strategies

```typescript
// In-memory caching with TTL
class TTLCache<K, V> {
  private cache = new Map<K, { value: V; expiresAt: number }>();

  get(key: K): V | undefined {
    const entry = this.cache.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return undefined;
    }
    return entry.value;
  }

  set(key: K, value: V, ttlMs: number): void {
    this.cache.set(key, { value, expiresAt: Date.now() + ttlMs });
  }
}

// Memoization for pure functions
function memoize<Args extends unknown[], Result>(
  fn: (...args: Args) => Result,
  keyFn: (...args: Args) => string = (...args) => JSON.stringify(args)
): (...args: Args) => Result {
  const cache = new Map<string, Result>();
  return (...args: Args) => {
    const key = keyFn(...args);
    if (cache.has(key)) return cache.get(key)!;
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```

**Why:** Caching avoids redundant computation and I/O. TTL (time-to-live) caches are
appropriate for data that changes periodically (API responses, config). Memoization is
appropriate for pure functions (same inputs always produce same outputs). Always consider:
what's the cache invalidation strategy? How much memory can the cache use? What happens
on cache miss? The hardest part of caching is knowing when the cache is stale.

---

## 6. Project Structure

### Layered Architecture

```
src/
├── index.ts              # Entry point — bootstraps the app
├── config/               # Environment, constants, feature flags
│   ├── index.ts
│   └── env.ts
├── types/                # Shared type definitions
│   ├── index.ts          # Re-exports
│   ├── user.ts
│   └── api.ts
├── domain/               # Business logic — no framework dependencies
│   ├── user/
│   │   ├── user.service.ts
│   │   ├── user.repository.ts  # Interface
│   │   └── user.errors.ts
│   └── order/
│       └── ...
├── infrastructure/       # External concerns — DB, APIs, file system
│   ├── database/
│   │   ├── client.ts
│   │   └── user.repository.impl.ts  # Implements domain interface
│   └── external-api/
│       └── payment.client.ts
├── api/                  # HTTP layer — routes, middleware, serialization
│   ├── routes/
│   │   ├── user.routes.ts
│   │   └── order.routes.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── error-handler.ts
│   │   └── validate.ts
│   └── server.ts
└── utils/                # Pure utility functions
    ├── date.ts
    └── string.ts
```

**Why:** Layered architecture creates a dependency direction: API → Domain → Infrastructure.
The domain layer contains business logic with no framework dependencies — it could work
with Express, Fastify, or a CLI. The infrastructure layer implements interfaces defined by
the domain. This makes the core logic testable without mocking HTTP servers or databases,
and makes it possible to swap infrastructure (e.g., PostgreSQL to MongoDB) without changing
business logic. The key rule: dependencies point inward, never outward.

### Barrel Exports

```typescript
// types/index.ts — re-exports from a directory
export type { User, UserRole } from './user.js';
export type { ApiResponse, PaginatedResponse } from './api.js';
export { UserStatus } from './user.js'; // enums are values, not just types

// Usage — clean imports from outside the directory
import type { User, ApiResponse } from './types/index.js';
```

**Why:** Barrel exports (index.ts re-export files) create a clean public API for each
directory. External code imports from the barrel, not from individual files. This means
you can reorganize files within the directory without breaking imports elsewhere. It also
makes it explicit which exports are "public" — anything not re-exported from the barrel
is an implementation detail.

**Caution:** Don't use barrels in leaf directories with only 1-2 files — it adds indirection
without benefit. Don't create circular barrel dependencies (A's barrel imports from B's
barrel which imports from A's barrel).

### Module Boundaries

```typescript
// ❌ Bad — reaching into another module's internals
import { hashPassword } from '../auth/utils/crypto.js';

// ✅ Good — import from the module's public API
import { hashPassword } from '../auth/index.js';

// Each module defines what it exposes
// auth/index.ts
export { authenticate, hashPassword } from './auth.service.js';
export type { AuthToken, AuthConfig } from './auth.types.js';
// Internal details like ./utils/crypto.js are not exported
```

**Why:** Module boundaries prevent spaghetti dependencies. If module A reaches into module
B's internal files, any refactoring of B's internals breaks A. By only importing from
the barrel, B can freely reorganize internally. This is the "information hiding" principle
applied to project structure. In larger projects, consider enforcing this with ESLint rules
(e.g., `eslint-plugin-import` boundaries).

### Configuration Pattern

```typescript
// config/env.ts — validated, typed configuration
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  CORS_ORIGIN: z.string().default('*'),
});

export type Env = z.infer<typeof envSchema>;

let _config: Env | null = null;

export function getConfig(): Env {
  if (!_config) {
    const result = envSchema.safeParse(process.env);
    if (!result.success) {
      console.error('❌ Invalid environment variables:');
      console.error(JSON.stringify(result.error.format(), null, 2));
      process.exit(1);
    }
    _config = result.data;
  }
  return _config;
}
```

**Why:** Configuration is a cross-cutting concern that every part of the application needs.
Validating it once at startup (fail fast) with a typed schema means: (1) missing or invalid
config crashes immediately with a clear error, not halfway through a request, (2) every
consumer gets typed access without parsing strings, (3) defaults are explicit and
centralized. The lazy singleton pattern ensures validation happens exactly once.

---

## 7. Design Patterns

### Repository Pattern — Abstract Data Access

```typescript
// Define the interface in the domain layer
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(data: CreateUserInput): Promise<User>;
  update(id: string, data: Partial<User>): Promise<User>;
  delete(id: string): Promise<void>;
}

// Implement in the infrastructure layer
class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async create(data: CreateUserInput): Promise<User> {
    return this.prisma.user.create({ data });
  }
  // ...
}

// The domain layer depends on the interface, not the implementation
class UserService {
  constructor(private userRepo: UserRepository) {} // interface, not Prisma

  async getUser(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError('User', id);
    return user;
  }
}
```

**Why:** The repository pattern decouples business logic from data access implementation.
`UserService` doesn't know or care whether data comes from PostgreSQL, MongoDB, or a
JSON file — it only knows the `UserRepository` interface. This makes business logic
testable (inject a mock repository), swappable (change databases without changing logic),
and readable (business methods express intent, not SQL).

### Factory Pattern — Complex Object Creation

```typescript
// When object creation involves logic, validation, or defaults
class NotificationFactory {
  static createEmail(to: string, subject: string, body: string): EmailNotification {
    return {
      type: 'email',
      to: to.toLowerCase().trim(),
      subject,
      body,
      createdAt: new Date(),
      retryCount: 0,
      priority: subject.includes('URGENT') ? 'high' : 'normal',
    };
  }

  static createSMS(phone: string, message: string): SMSNotification {
    if (message.length > 160) {
      throw new ValidationError('SMS message must be 160 characters or less');
    }
    return {
      type: 'sms',
      phone: phone.replace(/\D/g, ''),
      message,
      createdAt: new Date(),
      retryCount: 0,
    };
  }
}
```

**Why:** Factories centralize creation logic that would otherwise be duplicated everywhere
objects are created. They enforce invariants (SMS length), apply defaults (timestamps,
retry counts), and normalize input (lowercase email, strip phone formatting). When creation
logic changes, you update one place instead of every `new Notification(...)` call.

### Strategy Pattern — Swappable Algorithms

```typescript
// Define strategy interface
interface PricingStrategy {
  calculate(basePrice: number, quantity: number): number;
}

// Concrete strategies
const standardPricing: PricingStrategy = {
  calculate: (base, qty) => base * qty,
};

const bulkPricing: PricingStrategy = {
  calculate: (base, qty) => {
    const discount = qty >= 100 ? 0.2 : qty >= 50 ? 0.1 : 0;
    return base * qty * (1 - discount);
  },
};

const subscriptionPricing: PricingStrategy = {
  calculate: (base, qty) => base * qty * 0.8, // 20% subscriber discount
};

// Context uses strategy
class OrderService {
  constructor(private pricing: PricingStrategy) {}

  calculateTotal(items: OrderItem[]): number {
    return items.reduce(
      (total, item) => total + this.pricing.calculate(item.price, item.quantity),
      0
    );
  }
}

// Swap strategy at runtime
const pricing = user.isSubscriber ? subscriptionPricing : standardPricing;
const orderService = new OrderService(pricing);
```

**Why:** The strategy pattern replaces conditional logic (`if subscriber... else if bulk...`)
with composable objects. Each strategy is independently testable, new strategies can be
added without modifying existing code (open/closed principle), and the selection logic is
separated from the algorithm logic. In TypeScript, strategies can be simple objects with
methods — no need for class hierarchies.

### Builder Pattern — Complex Configuration

```typescript
class QueryBuilder<T> {
  private conditions: string[] = [];
  private params: unknown[] = [];
  private sortField?: string;
  private sortDir: 'ASC' | 'DESC' = 'ASC';
  private limitVal?: number;
  private offsetVal?: number;

  where(field: keyof T & string, op: '=' | '>' | '<' | 'LIKE', value: unknown): this {
    this.conditions.push(`${field} ${op} $${this.params.length + 1}`);
    this.params.push(value);
    return this;
  }

  orderBy(field: keyof T & string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.sortField = field;
    this.sortDir = direction;
    return this;
  }

  limit(n: number): this {
    this.limitVal = n;
    return this;
  }

  offset(n: number): this {
    this.offsetVal = n;
    return this;
  }

  build(): { query: string; params: unknown[] } {
    let query = `SELECT * FROM ${this.table}`;
    if (this.conditions.length) query += ` WHERE ${this.conditions.join(' AND ')}`;
    if (this.sortField) query += ` ORDER BY ${this.sortField} ${this.sortDir}`;
    if (this.limitVal) query += ` LIMIT ${this.limitVal}`;
    if (this.offsetVal) query += ` OFFSET ${this.offsetVal}`;
    return { query, params: this.params };
  }
}

// Usage — readable, chainable, safe
const { query, params } = new QueryBuilder<User>('users')
  .where('age', '>', 18)
  .where('status', '=', 'active')
  .orderBy('name')
  .limit(10)
  .build();
```

**Why:** The builder pattern shines when constructing complex objects with many optional
parameters. Without it, you'd either have a constructor with 10 parameters (confusing)
or a config object with 10 optional fields (no guidance on valid combinations). The
fluent interface (method chaining) makes construction readable and discoverable. Builders
also enforce construction order and can validate the final configuration in `build()`.

---

## 8. Algorithms & Data Structures

### Choose the Right Data Structure

| Need | Use | Why |
|------|-----|-----|
| Key-value storage | `Map<K, V>` | O(1) get/set, any key type, no prototype issues |
| Unique values | `Set<T>` | O(1) has/add/delete, auto-dedup |
| Ordered unique values | Sorted array + binary search | O(log n) search, O(n) insert |
| FIFO queue | Array with `shift()`/`push()`, or linked list for perf | Array is fine for small queues; linked list for large |
| LIFO stack | Array with `pop()`/`push()` | O(1) operations |
| Priority queue | Sorted insert or heap implementation | When items have priority ordering |
| Frequent lookups by multiple keys | Multiple `Map` instances indexing same data | Trade memory for O(1) multi-key lookup |

### Common Algorithmic Patterns

**Debounce — batch rapid calls**
```typescript
function debounce<T extends (...args: unknown[]) => void>(
  fn: T,
  delayMs: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delayMs);
  };
}
```

**Throttle — limit call frequency**
```typescript
function throttle<T extends (...args: unknown[]) => void>(
  fn: T,
  intervalMs: number
): (...args: Parameters<T>) => void {
  let lastCall = 0;
  return (...args) => {
    const now = Date.now();
    if (now - lastCall >= intervalMs) {
      lastCall = now;
      fn(...args);
    }
  };
}
```

**Retry with exponential backoff**
```typescript
async function retry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelayMs?: number;
    maxDelayMs?: number;
    shouldRetry?: (error: unknown) => boolean;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    baseDelayMs = 1000,
    maxDelayMs = 30000,
    shouldRetry = () => true,
  } = options;

  let lastError: unknown;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (attempt === maxRetries || !shouldRetry(error)) throw error;

      const delay = Math.min(baseDelayMs * 2 ** attempt, maxDelayMs);
      const jitter = delay * (0.5 + Math.random() * 0.5); // prevent thundering herd
      await new Promise(resolve => setTimeout(resolve, jitter));
    }
  }

  throw lastError;
}
```

**Batch processing — process N items at a time**
```typescript
async function processBatch<T, R>(
  items: T[],
  processor: (item: T) => Promise<R>,
  concurrency: number = 5
): Promise<R[]> {
  const results: R[] = [];

  for (let i = 0; i < items.length; i += concurrency) {
    const batch = items.slice(i, i + concurrency);
    const batchResults = await Promise.all(batch.map(processor));
    results.push(...batchResults);
  }

  return results;
}
```

**Pagination — cursor-based (preferred over offset)**
```typescript
interface PaginatedResult<T> {
  items: T[];
  cursor: string | null; // null means no more pages
}

async function paginatedQuery<T extends { id: string }>(
  query: (cursor: string | null, limit: number) => Promise<T[]>,
  limit: number = 20,
  cursor: string | null = null
): Promise<PaginatedResult<T>> {
  // Fetch one extra to check if there are more
  const items = await query(cursor, limit + 1);
  const hasMore = items.length > limit;
  const page = hasMore ? items.slice(0, limit) : items;

  return {
    items: page,
    cursor: hasMore ? page[page.length - 1].id : null,
  };
}
```

**Why cursor-based over offset?** Offset pagination (`OFFSET 100 LIMIT 20`) has two
problems: (1) performance degrades linearly — the database must scan and skip all offset
rows, (2) inserting/deleting items causes pages to shift, showing duplicates or skipping
items. Cursor-based pagination (`WHERE id > $cursor LIMIT 20`) is O(1) regardless of
page depth and stable under concurrent modifications.

### Complexity Awareness

When choosing approaches, consider the scale:

| Data size | Acceptable complexity | Example |
|-----------|----------------------|---------|
| < 100 items | O(n²) is fine | Nested loops, bubble sort |
| 100 - 10K | O(n log n) preferred | Use .sort(), Map/Set for lookups |
| 10K - 1M | O(n) or O(n log n) required | Streaming, indexed lookups |
| > 1M | O(n) with streaming | Process in chunks, use database |

**Practical rules:**
- If you're calling `.find()` or `.includes()` inside a loop, you likely have O(n²). Convert
  the search target to a `Set` or `Map` for O(n) total.
- If you're sorting, then searching, consider whether a single `Map` construction would be faster.
- If data fits in memory, in-memory operations are always faster than database queries.
- If data doesn't fit in memory, stream it — don't try to load it all at once.

---

## Quick Reference Card

**Always do:**
- `strict: true` in tsconfig
- Validate input at boundaries with schemas (Zod)
- Use `Map`/`Set` over plain objects for collections
- Handle errors with typed error classes
- Use `async`/`await`, never raw `.then()` chains
- Use `??` (nullish coalescing), not `||` for defaults
- Use `node:` protocol for built-in imports
- Bound all caches and growing collections
- Parameterize all queries

**Never do:**
- Use `any` (use `unknown` + narrowing)
- Hardcode secrets
- Concatenate user input into queries or commands
- Block the event loop with synchronous computation
- Swallow errors silently (`catch {}` or `catch (e) { /* ignore */ }`)
- Use `eval()` or `new Function()` with user input
- Trust client-side validation alone
- Use `var` (use `const` by default, `let` when needed)
