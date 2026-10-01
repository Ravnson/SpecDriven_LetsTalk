# Craftsmanship rules for `/spec-implement`

The contract for every line of code this skill writes. Used at three points: the design
pass (Step 9), the refactor beat of each unit (Step 11.3), and the review gate (Step 12).

**Precedence when rules conflict:**

1. The repository's `.editorconfig`, analyzers, and `CLAUDE.md`.
2. The dominant convention of the surrounding code.
3. This document.

Never "fix" a repo-wide convention you disagree with inside a feature PBI.

**A rule with a number in it is a review trigger, not a hard fail.** Crossing the number
means you must justify it in the design note or fix it — it does not mean the code is
automatically wrong.

---

## 1. SOLID

### SRP — Single Responsibility

- A type has **one reason to change**. Name that reason in one sentence in the design
  note. If the sentence needs "and", the type is two types.
- Separate the four concerns that most often get fused: **decide** (business rules),
  **do** (I/O and side effects), **map** (translation between shapes), **wire** (DI and
  composition).
- Triggers: type > 300 lines, > 7 public members, > 4 constructor dependencies, a name
  containing `Manager` / `Helper` / `Utils` / `Processor` with no narrower meaning.
- Keep business rules free of I/O so they are testable without a fake.

### OCP — Open/Closed

- New behaviour should arrive as a new type, not as another arm of a `switch` on a type
  discriminator that lives in three places.
- Prefer strategy / polymorphism / a registered handler list where variation is *expected
  by the requirements*.
- **Counterweight (YAGNI):** introduce the abstraction only when at least one holds —
  there are already two implementations, the requirements name a second one, or the test
  needs the seam. One implementation and no seam means use the concrete type.
- `switch` over a closed, exhaustive set (an enum you own, a discriminated result) is fine
  and often clearer than polymorphism. Prefer C# switch expressions with an exhaustive
  arm.

### LSP — Liskov Substitution

- A derived type must be usable anywhere the base is, with no caller-side type checks.
- Forbidden in an override: `throw new NotImplementedException()` /
  `NotSupportedException`, strengthening a precondition, weakening a postcondition,
  returning `null` where the base contract does not allow it.
- If a subtype cannot honour the contract, the hierarchy is wrong: prefer composition, or
  split the interface (see ISP).
- `sealed` by default. Inheritance is a deliberate, documented decision.

### ISP — Interface Segregation

- Interfaces are defined by what the **consumer** needs, not by what the implementation
  happens to offer.
- A consumer that uses 2 of 9 members is a signal to split.
- Trigger: an implementation whose members throw or no-op for part of the interface.
- Prefer several small role interfaces (`IOrderValidator`, `IOrderStore`) over one
  `IOrderService`.

### DIP — Dependency Inversion

- High-level policy depends on abstractions; abstractions do not depend on details.
- **Every** non-deterministic or external touch point goes behind an abstraction owned by
  the consuming layer: clock (`TimeProvider`), randomness, file system, network, database,
  message bus, environment variables, `Guid.NewGuid`, process/exit.
- Constructor injection only. **No service locator**, no `IServiceProvider` injected into
  business types, no static singletons for mutable state.
- Register with the correct lifetime and know why: `Singleton` (stateless or
  thread-safe shared), `Scoped` (per request/unit of work), `Transient` (cheap, stateful).
  Never inject a `Scoped` service into a `Singleton`.
- Interfaces live with the consumer (the layer that needs them), implementations with the
  detail. Dependency arrows point inward.

---

## 2. Rules of simple design (in priority order)

1. **Passes all tests.**
2. **Reveals intention** — a reader learns the domain from the names.
3. **No duplication** — of knowledge, not just of characters. Two identical lines that
   change for different reasons are not duplication; one rule encoded in two places is.
4. **Fewest elements** — delete anything the first three do not require.

Apply the **Rule of Three** before extracting: two occurrences may be coincidence, three
is a pattern. Premature deduplication couples things that should evolve apart.

---

## 3. Clean code

### Naming

- **All identifiers, comments, XML docs, log message templates, JSON property names, and
  schema/DB field names are in English** — even when the domain, the specification, the
  work item, or the user's prompt is in another language. Domain terms that have no English equivalent
  (legal or regulatory names) keep the original word, in an otherwise-English identifier.
- Intention-revealing, pronounceable, searchable. No abbreviations beyond the repo's
  established ones.
- Booleans read as a predicate: `IsValid`, `HasExpired`, `CanExport`.
- Async methods end in `Async`. Interfaces start with `I`. Type parameters `T`-prefixed.
- No type names in variable names (`orderList` → `orders`), no Hungarian notation.
- The name states *what*, not *how*: `SortedOrders`, not `BubbleSortedOrders`.

### Functions

- One level of abstraction per method. A method either orchestrates or does work, never
  both.
- Triggers: method > 30 lines, > 3 parameters, cyclomatic complexity > 10, nesting depth
  > 2, a boolean parameter that selects behaviour (split into two methods instead).
- **Guard clauses first, happy path last, no `else` after a `return`.**
- Command/query separation: a method either changes state or returns a value, not both.
- No output parameters except the established `TryX(out T)` pattern.

### Comments

- The code says *what*; a comment explains *why* — a non-obvious constraint, a spec
  reference, a deliberate deviation.
- **Delete instead of commenting out.** Version control is the archive.
- No `TODO` left in a merged diff. It goes into `implementation.md` Follow-ups, or into an
  ADO task.
- XML docs on public API only when the repo already does it, and then in the repo's voice
  and verbosity.

### Errors

- Exceptions for genuinely exceptional conditions; an expected failure (validation,
  not-found, conflict) is a return value — follow whatever result idiom the repo already
  uses, do not introduce a second one.
- Never swallow: no empty `catch`, no `catch (Exception)` that logs and continues without
  a stated reason.
- Catch the narrowest type you can handle. Rethrow with bare `throw;`, never
  `throw ex;` (it erases the stack trace).
- Exception messages state what failed and with which input — never include secrets or
  personal data.
- Don't return `null` for a collection — return an empty one (`[]`, `Array.Empty<T>()`).

---

## 4. .NET / C# specifics

### Nullability and contracts

- Nullable reference types on. Zero new nullable warnings.
- `!` (null-forgiving) requires a comment justifying it. Prefer restructuring.
- Guard public entry points: `ArgumentNullException.ThrowIfNull(x)`,
  `ArgumentException.ThrowIfNullOrWhiteSpace(s)`,
  `ArgumentOutOfRangeException.ThrowIfNegative(n)`.
- Make illegal states unrepresentable: a validated value object beats a `string` plus a
  validation call at every use site.

### Immutability and types

- `readonly` fields, `init`-only or get-only properties by default; mutability is a
  deliberate choice.
- `record` / `record struct` for value-like data (DTOs, events, results, options);
  `class` for identity and behaviour; `struct` only when small, immutable, and measured.
- `sealed` classes by default.
- Prefer `IReadOnlyList<T>` / `IReadOnlyDictionary<K,V>` on public surfaces over `List<T>`.

### Async

- **No `.Result`, no `.Wait()`, no `.GetAwaiter().GetResult()`** in application code.
- Async all the way down. No `async void` except an event handler.
- Thread `CancellationToken` through every async method and honour it; pass it to the
  framework calls at the bottom.
- `ConfigureAwait(false)` in libraries; not needed in ASP.NET Core application code —
  follow the repo.
- `Task.WhenAll` for genuinely parallel work; no fire-and-forget without an owner.
- `IAsyncEnumerable<T>` for streaming; `ValueTask` only on measured hot paths.

### Resources and performance

- `using` declarations for anything `IDisposable`/`IAsyncDisposable`; a type holding a
  disposable is itself disposable.
- Inject `HttpClient` via `IHttpClientFactory`; never `new HttpClient()` per call.
- Enumerate a sequence once. Materialise with `ToList()` when it is consumed twice — but
  don't materialise a stream you only stream.
- No allocation micro-optimisation without a benchmark. If performance is an AC, prove it
  with `dotnet-diag: microbenchmarking`, don't assert it.
- EF Core: projections over full entities, `AsNoTracking()` for reads, no N+1, no
  `.ToList()` before a filter. Use `dotnet-data: optimizing-ef-core-queries`.

### Configuration, logging, security

- No magic strings or numbers: `const`, a strongly-typed options class bound with
  `IOptions<T>` and validated on start.
- `ILogger<T>` with **message templates and structured properties** —
  `logger.LogInformation("Exported {OrderCount} orders", count)`, never string
  interpolation or concatenation in a log call.
- Log levels mean something: `Error` = needs a human; `Warning` = degraded but handled;
  `Information` = business event; `Debug`/`Trace` = developer detail.
- Never log secrets, tokens, credentials, or personal data.
- Never commit secrets. Configuration values with real content stay out of the repo.
- Validate and encode at the boundary: parameterised queries only, no string-built SQL, no
  unencoded user input into HTML/paths/commands.
- Culture-aware where it matters: `StringComparison.Ordinal` for identifiers and keys,
  `InvariantCulture` for machine-readable formatting, user culture only for display.

### Style

- Match the file you are editing: file-scoped vs block namespaces, `var` usage, expression
  bodies, brace placement, `using` ordering, private field prefix.
- Use modern C# only up to the project's `LangVersion`, and only where the codebase already
  uses that construct (primary constructors, collection expressions, pattern matching,
  required members).

---

## 5. Tests

- **Test behaviour through the public surface, not implementation details.** A refactor
  with no behaviour change must not break a test.
- One behaviour per test. Multiple asserts are fine if they describe one behaviour.
- **AAA** with a blank line between Arrange, Act, Assert.
- Name: `Method_Scenario_ExpectedResult` unless the repo does otherwise.
- **No logic in tests** — no `if`, no loops, no arithmetic on expectations. Use data-driven
  cases (`[DataRow]`, `[Theory]`) instead.
- Deterministic: no `DateTime.Now`, no `Random` without a seed, no `Thread.Sleep`, no
  real network, no dependence on test ordering or on a shared mutable fixture.
- Prefer **fakes/stubs over mocks**; assert on outcomes, not on call counts. A mock
  verifying interaction order is testing the implementation.
- Never mock a type you don't own — wrap it and fake the wrapper.
- Assertion quality: assert the specific value and the specific exception type, not
  `Assert.IsNotNull` or a bare `Assert.IsTrue`.
- Test data via builders/object mothers, not copy-pasted literal blobs.
- Cover the boundaries the AC implies: empty, single, many, null, min/max, duplicate,
  cancellation, concurrent access, culture, time zone, DST.
- The test suite is production code: the naming, duplication, and clarity rules above
  apply to it unchanged.

---

## 6. Review checklist (Step 12)

Run over the **whole diff**. Every unchecked line either gets fixed or gets recorded in
`implementation.md` Follow-ups with a reason.

**Scope**
- [ ] Every change traces to an AC of this PBI; nothing extra.
- [ ] No unrelated bug fix, reformat, or package bump.
- [ ] Refactors are in their own commits, separate from behaviour change.

**Design**
- [ ] Each new type has one stateable reason to change.
- [ ] No abstraction with one implementation and no test seam.
- [ ] No new dependency pointing outward through a layer boundary.
- [ ] No override that throws, no caller-side type check.
- [ ] All non-determinism and I/O is behind an injected abstraction.
- [ ] DI lifetimes correct; no captive dependency.

**Code**
- [ ] Names reveal intent; all English.
- [ ] No duplicated knowledge; Rule of Three respected before extracting.
- [ ] Guard clauses at entry; nesting ≤ 2; methods within the triggers.
- [ ] No magic values; no commented-out code; no leftover `TODO`.
- [ ] Errors handled at the right level; nothing swallowed; `throw;` not `throw ex;`.
- [ ] Nullability clean; no unjustified `!`.
- [ ] Async correct: no sync-over-async, token threaded and honoured.
- [ ] Disposables disposed.
- [ ] Structured logging; no secrets or personal data logged or committed.

**Tests**
- [ ] Every AC has a test that would fail without the change.
- [ ] Tests assert behaviour, not interactions.
- [ ] Boundaries and failure paths covered, not just the happy path.
- [ ] Deterministic; no sleeps, no ordering dependence.
- [ ] Coverage of new code did not regress.

**Build**
- [ ] Clean build, zero new warnings, analyzers satisfied.
- [ ] Formatting matches the repo's tooling.
- [ ] Full suite green.
