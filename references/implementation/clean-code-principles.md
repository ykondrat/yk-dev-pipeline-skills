# Clean Code Principles for JS/TS Implementation

Based on Robert C. Martin's "Clean Code". These principles apply to every file you write
during implementation. The SKILL.md code quality standards are the checklist — this
reference explains the *why* and *how* behind each standard.

---

## 1. Meaningful Names

Names are the most important tool for readable code. Every name should reveal intent.

### Rules

- **Intention-revealing names** — The name should answer: why it exists, what it does, how
  it's used. If a name requires a comment to explain it, change the name.
  ```typescript
  // Bad
  const d = Date.now() - start; // elapsed time in ms
  // Good
  const elapsedTimeMs = Date.now() - startTime;
  ```

- **Avoid disinformation** — Don't use `accountList` for something that isn't a `List`.
  Don't use names that vary in subtle ways (`XYZControllerForHandlingOfStrings` vs
  `XYZControllerForStorageOfStrings`).

- **Meaningful distinctions** — Number-series names (`a1`, `a2`) and noise words (`data`,
  `info`, `theMessage`) add nothing. If names must differ, they should differ in a way
  that tells the reader what the difference means.
  ```typescript
  // Bad
  function copyChars(a1: string[], a2: string[]): void
  // Good
  function copyChars(source: string[], destination: string[]): void
  ```

- **Pronounceable names** — If you can't pronounce it, you can't discuss it without
  sounding foolish. `genymdhms` → `generationTimestamp`.

- **Searchable names** — Single-letter names and numeric constants are hard to find.
  `MAX_CLASSES_PER_STUDENT` is searchable; `7` is not. The length of a name should
  correspond to the size of its scope.

- **One word per concept** — Pick one word for one abstract concept: `fetch`, `retrieve`,
  or `get` — not all three. Be consistent across the codebase.

- **Domain names** — Use solution domain terms (`factory`, `visitor`, `queue`, `subscriber`)
  when the concept is well-known. Use problem domain terms when there's no programmer
  equivalent. The reader will be a programmer — don't be afraid of technical names.

- **Context from enclosing structure** — Use class/module names to provide context rather
  than prefixing every variable. `Address.state` instead of `addrState`. But don't add
  gratuitous context — `GSDAccountAddress` for every class in the "Gas Station Deluxe" app
  is noise.

---

## 2. Functions

Functions are the first line of organization in any program.

### Rules

- **Small** — Functions should be small. 5-15 lines is ideal. They should rarely exceed 20
  lines and should almost never exceed 30. If a function is long, it's doing too much.

- **Do one thing** — A function should do one thing, do it well, and do it only. If you
  can extract another function with a name that isn't merely a restatement of its
  implementation, the original is doing more than one thing.

- **One level of abstraction** — Statements within a function should all be at the same
  level of abstraction. Don't mix high-level intent (`getHtml()`) with low-level detail
  (`append("\n")`) in the same function.

- **The Stepdown Rule** — Code should read like a top-down narrative. Each function should
  be followed by the functions at the next level of abstraction, so the program reads
  top-to-bottom as a set of "TO paragraphs":
  > "To do X, we first do A, then B, then C. To do A, we..."

- **Few arguments** — Zero arguments (niladic) is ideal, one (monadic) is fine, two (dyadic)
  is acceptable. Three (triadic) should be rare. More than three requires very special
  justification. When a function needs many values, group them into an object.
  ```typescript
  // Bad
  function createMenu(title: string, body: string, buttonText: string, cancellable: boolean): Menu
  // Good
  function createMenu(config: MenuConfig): Menu
  ```

- **No side effects** — A function that promises to do one thing but also does hidden things
  (modifies a global, changes a passed-in argument, does I/O) is lying. Name functions
  honestly about what they do.

- **Command-query separation** — A function should either do something (command) or answer
  something (query), but not both. `setName()` should set; `isValid()` should check.
  `if (set("username", "bob"))` is confusing.

- **Prefer exceptions to error codes** — Error codes force callers into immediate handling
  and create deep nesting. Exceptions let you separate the happy path from error handling.
  In TypeScript: throw typed errors, catch at boundaries.

- **DRY** — Duplication is the root of all evil in software. If you find the same code in
  two places, extract it into a function.

---

## 3. Comments

Comments are not "good." They are a necessary failure to express ourselves in code.

### Good Comments (use these)

- **Legal comments** — Copyright/license headers when required.
- **Intent** — Explain *why* a decision was made, not *what* code does.
  ```typescript
  // We use a sorted set here because membership checks are O(log n)
  // and we need ordered iteration for the leaderboard display.
  ```
- **Clarification** — When translating obscure arguments or return values from a library
  you can't modify.
- **Warning** — Warn of consequences: `// WARNING: this test takes 10 minutes to run`.
- **TODO** — Mark genuine incomplete work: `// TODO(#123): handle pagination`.
- **Amplification** — Emphasize importance of something that might seem minor.

### Bad Comments (never write these)

- **Redundant** — Comments that say what the code already says.
  ```typescript
  // Bad — the name already tells us
  /** Returns the day of the month */
  getDayOfMonth(): number
  ```
- **Misleading** — Comments that don't precisely describe what the code does.
- **Mandated/noise** — JSDoc for every internal function just because. No `/** Default constructor */`.
- **Journal/changelog** — That's what git is for. No `// Added by John on 2024-01-15`.
- **Commented-out code** — Delete it. It's in git history if you need it.
- **Position markers** — No `// ============ PRIVATE METHODS ============`.

### The Rule

If you feel the need to write a comment, first try to refactor the code so the comment
becomes unnecessary. Rename the variable, extract the function, make the code speak for
itself. Only write the comment if the code *cannot* express the intent.

---

## 4. Error Handling

Error handling is important, but if it obscures logic, it's wrong.

### Rules

- **Use exceptions, not error codes** — In TypeScript, throw errors (custom error classes
  extending `Error`) rather than returning error objects or null to indicate failure.
  Exceptions separate the happy path from error handling.

- **Write try-catch-finally first** — When writing code that could throw, start with the
  try-catch structure. This defines the scope and expectations for the caller.

- **Provide context** — Include informative error messages. Mention the operation, the
  input that caused the failure, and the type of failure. A stack trace tells where;
  a good message tells why.
  ```typescript
  // Bad
  throw new Error('Failed');
  // Good
  throw new DatabaseError(`Failed to fetch user by id=${userId}: connection refused`);
  ```

- **Define exceptions by caller's needs** — Wrap third-party errors into your own error
  types that make sense for your application. The caller shouldn't need to know about
  the internal library you use.

- **Don't return null** — Returning null forces callers to check for null everywhere. One
  missing check = runtime error. Return empty arrays, empty objects, throw an error, or
  use the Null Object / Special Case pattern instead.
  ```typescript
  // Bad — forces every caller to null-check
  function getUsers(): User[] | null
  // Good — callers can safely iterate
  function getUsers(): User[]
  ```

- **Don't pass null** — Forbid null as an argument by default. Use TypeScript's
  `strictNullChecks` to catch these at compile time.

- **Separate error handling from logic** — Error handling is one thing. A function that
  handles errors should not also handle business logic. Extract the try-catch into its
  own function if the error handling obscures the algorithm.

---

## 5. Classes and Modules

### Rules

- **Small and focused** — Classes/modules should be small. Measure by responsibility, not
  line count. You should be able to describe a class in ~25 words without using "and,"
  "or," "but," or "if."

- **Single Responsibility Principle (SRP)** — A class should have one, and only one, reason
  to change. A `UserService` that handles auth AND email AND profile updates has three
  reasons to change — split it.

- **High cohesion** — Methods should use most of the class's instance variables. If a
  subset of methods uses a subset of variables, that's a new class waiting to be extracted.

- **Organize for change** — When you know a class will change, structure it so that changes
  are isolated. Use interfaces and dependency injection to decouple components. Prefer
  composition over inheritance.

- **Open/Closed Principle (OCP)** — Classes should be open for extension but closed for
  modification. When a new feature requires changing many existing switch/if-else
  statements, consider using polymorphism instead.

---

## 6. Boundaries

### Rules

- **Wrap third-party APIs** — Don't let third-party types and interfaces spread through
  your codebase. Wrap them in your own abstractions. This limits the blast radius when
  a dependency changes.

- **Don't return raw third-party types** — If a library returns `AxiosResponse`, your
  service should return your own `ApiResponse` type. Callers shouldn't know or care
  about implementation details.

- **Learning tests** — When integrating a new dependency, write tests that verify your
  understanding of the API. These also catch breaking changes on upgrade.

---

## 7. Formatting

### Rules

- **Vertical openness** — Separate concepts with blank lines. Group related code together.
  Each blank line is a visual cue that a new concept begins.

- **Vertical density** — Lines of code that are tightly related should appear close
  together. Don't separate them with comments or blank lines.

- **Declare close to use** — Variables should be declared as close to their usage as
  possible. Instance variables at the top of a class. Local variables at the top of
  each function.

- **Dependent functions close** — If one function calls another, keep them vertically
  close in the file. The caller should be above the callee (Stepdown Rule).

- **Conceptual affinity** — Functions that do similar things or share naming patterns
  should be grouped together.

---

## 8. Four Rules of Simple Design (Kent Beck)

In priority order:

1. **Runs all the tests** — A system that cannot be verified is a system that can never
   be deployed. Make code testable as you write it.

2. **No duplication** — DRY. Duplication represents missed abstraction opportunities.
   Even subtle duplication in structure or approach should be eliminated.

3. **Expresses intent** — Code should clearly communicate the programmer's intent.
   Choose good names, keep functions small, use standard patterns. The next reader
   should easily understand what you were thinking.

4. **Minimizes classes and methods** — Don't create abstractions "in case we need them
   later." Don't make a class for every concept. Keep the number of classes, functions,
   and modules as low as possible while honoring rules 1-3. YAGNI.

---

## 9. Code Smells Quick Reference

Check for these during self-review:

| Smell | Description | Fix |
|-------|-------------|-----|
| **Long function** | Function > 20-30 lines | Extract smaller functions |
| **Too many arguments** | Function takes > 3 params | Use config/options object |
| **Boolean arguments** | `render(true)` | Split into two functions |
| **Dead code** | Unreachable/unused code | Delete it (git has history) |
| **Duplication** | Same logic in 2+ places | Extract shared function |
| **Feature envy** | Function uses another module's data more than its own | Move it |
| **Data clumps** | Same group of variables appears together repeatedly | Extract into type/class |
| **Primitive obsession** | Using strings/numbers for domain concepts | Create types/value objects |
| **Divergent change** | One module changes for multiple unrelated reasons | Split responsibilities |
| **Shotgun surgery** | One change requires editing many files | Consolidate logic |
| **Long chain** | `a.b().c().d()` — Law of Demeter violation | Tell, don't ask |
| **Commented-out code** | Dead code in comments | Delete it |
| **Magic numbers/strings** | Literals without names | Extract named constants |
| **Inappropriate naming** | Names don't reveal intent | Rename |
| **Inconsistency** | Same concept, different patterns | Pick one and align |
| **Hidden temporal coupling** | Functions must be called in order but nothing enforces it | Make the dependency explicit |