# Investigation Patterns Reference

This reference is loaded during Step 3 of the investigation skill. It provides systematic
approaches for debugging, root cause analysis, refactoring assessment, and performance
investigation in JavaScript/TypeScript projects.

---

## Bug Investigation Workflow

Follow this cycle until the root cause is confirmed:

```
Reproduce → Isolate → Trace → Hypothesize → Verify → Document
    ↑                                           |
    └───────────── (if hypothesis wrong) ───────┘
```

### 1. Reproduce

Before any analysis, confirm you can trigger the issue:

- **Exact reproduction**: Follow the reported steps verbatim
- **Minimal reproduction**: Strip away unrelated parts until you have the smallest case
- **Automated reproduction**: Write a failing test if possible — this becomes your
  verification test later

```typescript
// A failing test IS a reproduction
it('should not throw when input is empty array', () => {
  // This is the reported bug — crashes on empty input
  expect(() => processItems([])).not.toThrow();
});
```

If you can't reproduce:
- Check environment differences (Node version, OS, env vars)
- Check data differences (production data vs test data)
- Check timing (race conditions only appear under specific conditions)
- Ask for logs, stack traces, or screenshots from where it does reproduce

### 2. Isolate

Narrow down the failure to the smallest possible scope:

- **Binary search**: Comment out half the code. Does the problem persist? Narrow further.
- **Input isolation**: What's the minimal input that triggers the issue?
- **Component isolation**: Does the component fail independently or only in context?
- **Dependency isolation**: Does the issue persist with mocked dependencies?

### 3. Trace

Follow the execution path step by step:

- Start from the entry point (HTTP handler, CLI command, event listener, user action)
- Read each function in the call chain
- Note every data transformation
- Document the path with `file:line` references

```
Request flow for POST /api/users:
1. src/routes/users.ts:15 — Route handler receives request
2. src/routes/users.ts:18 — Calls validateUserInput(body)
3. src/validation/user.ts:42 — Validates email format ← BUG: regex doesn't handle + alias
4. src/services/user.ts:23 — Never reached due to validation error
```

### 4. Hypothesize

Generate multiple hypotheses for the root cause:

| Hypothesis | Evidence For | Evidence Against | How to Verify |
|-----------|-------------|-----------------|---------------|
| H1: Regex doesn't handle + in email | Fails with user+tag@email.com | Works for simple emails | Test with + character |
| H2: Input encoding issue | Error mentions invalid chars | Works in Postman | Compare raw request bodies |
| H3: Middleware strips + | Express URL-decodes by default | Other fields with + work | Log body before/after middleware |

**Rule: Consider at least 2-3 hypotheses before investigating the first one.**

### 5. Verify

Confirm the root cause definitively:

- Can you make the bug appear by changing only the root cause?
- Can you make the bug disappear by fixing only the root cause?
- Does the root cause explain ALL the reported symptoms?
- Are there cases where the root cause exists but the bug doesn't appear? (Explain why.)

### 6. Document

Record everything in the investigation report:
- Reproduction steps
- Code path trace
- Hypotheses considered (including rejected ones)
- Root cause with evidence
- Verification method

---

## Debugging Strategies

### Binary Search Debugging

When you don't know where the problem is, bisect:

**Code bisection:**
- Comment out the second half of the suspect function
- Does the problem persist? It's in the first half. Otherwise, second half.
- Repeat until you find the exact line

**Git bisection:**
```bash
git bisect start
git bisect bad HEAD                    # Current commit is broken
git bisect good abc1234                # This commit was working
# Git checks out a middle commit — test it, then:
git bisect good   # or
git bisect bad
# Repeat until git identifies the first bad commit
```

**Dependency bisection:**
- Remove half the dependencies (or mock them)
- Does the problem persist? Narrow further.

### Trace Logging

Add strategic log points to track data flow:

```typescript
// Trace a value through transformations
function processOrder(order: Order): Result {
  console.log('[TRACE] processOrder input:', JSON.stringify(order));

  const validated = validateOrder(order);
  console.log('[TRACE] after validation:', JSON.stringify(validated));

  const enriched = enrichWithPricing(validated);
  console.log('[TRACE] after enrichment:', JSON.stringify(enriched));

  const result = submitOrder(enriched);
  console.log('[TRACE] final result:', JSON.stringify(result));

  return result;
}
```

**Key points for trace logging:**
- Log at function boundaries (entry/exit)
- Log before and after transformations
- Include enough context to identify the request/flow
- Use structured format for easy parsing
- **Remove all trace logging before committing**

### Rubber Duck Analysis

When stuck, explain the problem systematically:

1. State the expected behavior precisely
2. State the actual behavior precisely
3. List everything you know is working correctly
4. List everything you haven't verified
5. Identify the gap — what haven't you checked?

### Stack Trace Reading

When analyzing a stack trace:

1. **Start at the top** — the thrown error and message
2. **Find your code** — skip framework/library frames to find the first line of your code
3. **Read the call chain** — understand how execution got there
4. **Check the bottom** — what initiated this call chain (HTTP request, timer, event)

```
Error: Cannot read property 'name' of undefined
    at formatUser (src/formatters/user.ts:15:23)        ← Bug is here
    at processUsers (src/services/user.ts:42:12)        ← Called from here
    at Array.map (<anonymous>)                           ← Framework/runtime
    at handleGetUsers (src/routes/users.ts:18:28)       ← Entry point
    at Layer.handle (node_modules/express/lib/router/layer.js:95:5)
```

**Common patterns:**
- `Cannot read property X of undefined` — Something upstream returned undefined
- `Maximum call stack size exceeded` — Infinite recursion
- `ECONNREFUSED` — Service not running or wrong port
- `ETIMEOUT` — Network or DNS issue
- `ENOMEM` — Memory exhaustion

---

## Common Root Cause Categories

### Race Conditions

**Symptoms:** Intermittent failures, works locally but fails in CI, timing-dependent

**Look for:**
- Shared mutable state accessed from async code
- Missing `await` on promises
- Event ordering assumptions
- Database read-then-write without transactions
- Concurrent requests modifying the same resource

```typescript
// Race condition: two requests can read the same counter
async function incrementCounter(id: string) {
  const current = await db.counter.findUnique({ where: { id } });  // Read
  // ← Another request can read the same value here
  await db.counter.update({
    where: { id },
    data: { value: current.value + 1 }  // Write stale value
  });
}

// Fix: use atomic operation
async function incrementCounter(id: string) {
  await db.counter.update({
    where: { id },
    data: { value: { increment: 1 } }
  });
}
```

### Off-by-One Errors

**Symptoms:** Missing first/last item, array index out of bounds, fence-post errors

**Look for:**
- Loop bounds: `<` vs `<=`, `0` vs `1` start index
- Array slicing: inclusive vs exclusive end
- Pagination: offset calculations
- String operations: index vs length confusion

### Null/Undefined Errors

**Symptoms:** "Cannot read property X of undefined", unexpected NaN, "undefined is not a function"

**Look for:**
- Optional chaining missing on nullable values
- API responses that can return null/empty
- Destructuring without defaults
- Type assertions hiding nullable values (`as NonNullable<T>`)
- Map/object lookups that miss

### State Management Issues

**Symptoms:** Stale UI, data out of sync, "it works after refresh"

**Look for:**
- Stale closures capturing old state
- Missing dependency arrays in useEffect/useMemo
- Mutating state directly instead of creating new references
- Race conditions between state updates
- Derived state that doesn't update when source changes

```typescript
// Stale closure bug
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      // Bug: captures initial count (0) forever
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(timer);
  }, []); // Missing count dependency — or use functional update

  // Fix: use functional update
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(prev => prev + 1);  // Always uses latest value
    }, 1000);
    return () => clearInterval(timer);
  }, []);
}
```

### Async/Promise Errors

**Symptoms:** Unhandled rejections, operations in wrong order, missing data

**Look for:**
- Missing `await` keywords
- `.catch()` missing or swallowing errors
- Promise.all vs Promise.allSettled choice
- Async functions in non-async contexts (forEach, map without Promise.all)
- Error handling in async generators/iterators

```typescript
// Silent failure: forEach doesn't await
items.forEach(async (item) => {
  await processItem(item);  // These run concurrently, errors are lost
});

// Fix: use for...of for sequential, Promise.all for parallel
for (const item of items) {
  await processItem(item);  // Sequential, errors propagate
}
// Or:
await Promise.all(items.map(item => processItem(item)));  // Parallel, errors propagate
```

### Type Coercion Issues

**Symptoms:** Unexpected equality results, wrong branch taken, NaN propagation

**Look for:**
- `==` instead of `===`
- String/number confusion from URL params, env vars, JSON
- Boolean coercion of empty arrays/objects (they're truthy!)
- parseInt without radix, or parseFloat on non-numeric strings
- Template literal coercion hiding undefined

---

## Refactoring Analysis

### Code Smell Catalog

When investigating code for refactoring, identify these patterns:

| Smell | Symptom | Impact | Action |
|-------|---------|--------|--------|
| **Long function** | >50 lines, multiple levels of abstraction | Hard to test, hard to understand | Extract functions |
| **God object** | Class/module does everything | Tight coupling, hard to change | Split by responsibility |
| **Feature envy** | Function uses more of another module's data than its own | Wrong responsibility placement | Move to correct module |
| **Primitive obsession** | Using strings/numbers for domain concepts | No validation, no type safety | Create value objects/types |
| **Shotgun surgery** | One change requires editing many files | High coupling | Consolidate related logic |
| **Divergent change** | One file changes for many different reasons | Mixed responsibilities | Split by reason for change |
| **Data clumps** | Same group of params passed together | Missing abstraction | Create a type/interface |
| **Switch statements** | Repeated switch/if-else on same type | Missed polymorphism | Use strategy or map pattern |
| **Speculative generality** | Abstractions for unused flexibility | Unnecessary complexity | Remove until needed |
| **Dead code** | Unreachable code, unused exports | Confusion, maintenance burden | Delete it |

### Coupling Analysis

Assess coupling between components:

- **Afferent coupling (Ca)**: How many modules depend ON this one? (high = hard to change)
- **Efferent coupling (Ce)**: How many modules does this one depend on? (high = fragile)
- **Instability = Ce / (Ca + Ce)**: 0 = maximally stable, 1 = maximally unstable

**Look for:**
- Circular dependencies (`A → B → C → A`)
- God modules with high Ca (everything depends on them)
- Fragile modules with high Ce (break when anything changes)
- Dependency direction violations (domain depending on infrastructure)

### Complexity Analysis

Assess code complexity:

- **Cyclomatic complexity**: Count decision points (if, else, for, while, case, &&, ||, ?:).
  >10 per function is a warning, >20 is critical.
- **Nesting depth**: >3 levels of nesting is a warning. Consider early returns or extraction.
- **Cognitive complexity**: How hard is it to understand the control flow? Nested conditions
  and loops compound difficulty beyond their count.

---

## Performance Investigation

### Profiling Approach

Don't optimize without measuring. Follow this order:

1. **Measure** — Establish baseline metrics (response time, memory usage, CPU time)
2. **Identify** — Find the bottleneck (it's usually not where you think)
3. **Optimize** — Fix the bottleneck only
4. **Verify** — Confirm improvement with same measurements
5. **Repeat** — Next bottleneck may be somewhere else

### Common Performance Issues

#### N+1 Queries

**Symptom:** API slows linearly with data size, database shows many small queries

```typescript
// N+1: one query per user
const users = await db.user.findMany();
for (const user of users) {
  user.posts = await db.post.findMany({ where: { userId: user.id } });  // N queries
}

// Fix: eager loading / join
const users = await db.user.findMany({
  include: { posts: true }  // Single query with JOIN
});
```

#### Memory Leaks

**Symptom:** Memory grows over time, eventual OOM crash, GC pauses increase

**Look for:**
- Event listeners never removed
- Growing caches without eviction
- Closures capturing large objects
- Global arrays/maps that accumulate
- Unclosed streams, connections, or file handles
- `setInterval` without cleanup

```typescript
// Memory leak: listener accumulates on every call
function subscribe(emitter: EventEmitter) {
  emitter.on('data', (chunk) => {
    processChunk(chunk);
  });
  // Never calls emitter.off() — listeners accumulate
}

// Fix: return cleanup function
function subscribe(emitter: EventEmitter) {
  const handler = (chunk: Buffer) => processChunk(chunk);
  emitter.on('data', handler);
  return () => emitter.off('data', handler);
}
```

#### Event Loop Blocking

**Symptom:** All requests slow down simultaneously, server becomes unresponsive

**Look for:**
- Synchronous file I/O (`readFileSync` in request handlers)
- CPU-intensive computation in the main thread (crypto, parsing, sorting large arrays)
- `JSON.parse` / `JSON.stringify` on very large objects
- Synchronous regular expressions on long strings (ReDoS)

#### Bundle Size

**Symptom:** Slow page load, large JavaScript download

**Look for:**
- Barrel file imports pulling in entire modules
- Missing tree-shaking (default imports from large libraries)
- Uncompressed assets
- Duplicate dependencies in bundle
- Missing code splitting / lazy loading

```typescript
// Pulls in entire lodash (~70KB)
import _ from 'lodash';
_.debounce(fn, 300);

// Only pulls debounce (~1KB)
import debounce from 'lodash/debounce';
debounce(fn, 300);
```

#### Re-render Storms (React/Vue)

**Symptom:** UI jank, slow interactions, high CPU on user actions

**Look for:**
- Missing memoization (React.memo, useMemo, useCallback)
- Object/array literals in props (new reference every render)
- Context providers re-rendering all consumers
- State stored too high in the tree
- Missing keys or wrong keys in lists

### Benchmarking

When comparing approaches, use consistent benchmarks:

```typescript
// Simple timing for Node.js
const start = performance.now();
await operation();
const elapsed = performance.now() - start;
console.log(`Operation took ${elapsed.toFixed(2)}ms`);

// Multiple iterations for accuracy
const times: number[] = [];
for (let i = 0; i < 100; i++) {
  const start = performance.now();
  await operation();
  times.push(performance.now() - start);
}
const avg = times.reduce((a, b) => a + b, 0) / times.length;
const p95 = times.sort((a, b) => a - b)[Math.floor(times.length * 0.95)];
console.log(`Avg: ${avg.toFixed(2)}ms, P95: ${p95.toFixed(2)}ms`);
```

---

## Dependency & Security Audit

When the investigation involves dependency issues:

### Outdated Dependencies

```bash
npm outdated                    # Show outdated packages
npm audit                       # Check for known vulnerabilities
npx npm-check-updates           # Show available updates
```

**Check before updating:**
- Read the changelog for breaking changes
- Check if the major version bump requires code changes
- Look for peer dependency conflicts
- Run tests after each significant update

### Common Dependency Issues

- **Version conflicts**: Two packages require different versions of a shared dependency
- **Missing peer dependencies**: Warnings during install that cause runtime errors
- **Deprecated packages**: Using abandoned packages with no security patches
- **Type definition mismatch**: `@types/foo` version doesn't match `foo` version
- **Lock file drift**: `package-lock.json` doesn't match `package.json`

---

## Evidence Documentation Patterns

### File Reference Format

Always use this format for referencing code:

```
src/api/routes.ts:42           — Single line
src/api/routes.ts:42-58        — Line range
src/api/routes.ts:42 (handler) — Line with context
```

### Data Flow Documentation

When tracing how data moves through the system:

```
Input: POST /api/orders { items: [...], userId: "abc" }
  → src/routes/orders.ts:15     validateRequest(body)
  → src/validation/order.ts:8   checks items array ← MISSING: empty array check
  → src/services/order.ts:23    calculateTotal(items)
  → src/services/pricing.ts:45  items.reduce(...) ← CRASHES: reduce on empty array
  → Error: "Cannot read property 'price' of undefined"
```

### Timeline Documentation

For intermittent issues, document the timeline:

```
Timeline:
- 2026-02-20 14:00 — Deploy v2.3.1 (includes dependency update)
- 2026-02-20 14:15 — First error report from monitoring
- 2026-02-20 14:30 — Error rate increases to 5%
- 2026-02-20 15:00 — Rollback to v2.3.0, errors stop
- Root cause: Breaking change in dependency v3.0 not caught by tests
```

---

## Investigation Anti-Patterns

Avoid these common mistakes during investigation:

### Assumptions Without Evidence
- **Wrong:** "It's probably a race condition" (without evidence)
- **Right:** "I see two async operations modifying `userSession` without synchronization at `src/auth.ts:42` and `src/auth.ts:67`"

### Fixing Symptoms Instead of Root Causes
- **Wrong:** Adding a null check to silence the error
- **Right:** Understanding why the value is null in the first place

### Scope Creep
- **Wrong:** "While I'm here, let me also refactor this entire module"
- **Right:** Note the improvement opportunity in the report, stay focused on the reported issue

### Premature Optimization
- **Wrong:** "This loop is O(n²), let me optimize it" (without measuring)
- **Right:** "This loop processes 10 items max. The actual bottleneck is the database query at line 42"

### Tunnel Vision
- **Wrong:** Spending hours on one hypothesis without considering alternatives
- **Right:** After 15 minutes without progress, step back and consider other hypotheses

### Blaming the Framework
- **Wrong:** "React/Express/Prisma must have a bug"
- **Right:** Check your usage first. Framework bugs exist but are rare — your code is the most likely culprit

### Ignoring the Simple Explanation
- **Wrong:** Building an elaborate theory involving timing and state
- **Right:** Check if someone just typo'd a variable name or passed arguments in the wrong order

### Not Verifying the Fix
- **Wrong:** "I found the bug, here's the fix" (without proving the fix works)
- **Right:** Show that the fix resolves the reproduction case AND doesn't break existing tests
