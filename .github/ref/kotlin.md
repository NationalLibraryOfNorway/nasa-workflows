# Kotlin Reference

Rules for Kotlin work. Apply on top of [`../AGENTS.md`](../AGENTS.md). Many JVM rules from [`java.md`](java.md) still apply (logging, build tools, security) — this file covers what's Kotlin-specific.

## Style

- Idiomatic Kotlin. Don't write Java-in-Kotlin.
- Prefer `val` over `var`. Mutable state only when necessary.
- Use data classes for value types. `copy()` for derived instances.
- Expression bodies for short single-purpose functions: `fun area() = width * height`.
- Top-level / extension functions over static utility classes.
- Scope functions (`let`, `apply`, `also`, `run`, `with`) — use intentionally, not reflexively.

## Null safety

- Don't use `!!` to silence the compiler. If you must, comment why.
- Prefer `?.`, `?:`, `requireNotNull(...)`, and smart-casts.
- Platform types from Java APIs: annotate or wrap at the boundary. Don't propagate `String!` through your own code.
- Public APIs: be explicit about nullability.

## Collections & sequences

- Standard collection ops (`map`, `filter`, `groupBy`) over manual loops.
- Use `Sequence` for long pipelines over large collections to avoid intermediate allocations.
- Prefer immutable collection types (`List`, `Map`, `Set`) on APIs. Use `Mutable*` only when you actually mutate.

## Coroutines

- `suspend` functions don't block — never call blocking IO inside without `withContext(Dispatchers.IO)`.
- Structured concurrency: launch in a `CoroutineScope` you own. Don't use `GlobalScope`.
- Cancellation is cooperative — check `isActive` or use cancellable suspending calls in long loops.
- `Flow` for streams. `StateFlow`/`SharedFlow` for hot streams. Don't expose `MutableStateFlow` publicly.
- Tests: `runTest { }` from `kotlinx-coroutines-test`. Don't use `runBlocking` in tests.

## Logging

- Same rules as [`java.md`](java.md) § Logging (SLF4J, `{}` placeholders, no `println`). Emit JSON in production — see [`structured-logging.md`](structured-logging.md).
- Coroutines lose the thread-local MDC on dispatch. Use `MDCContext()` from `kotlinx-coroutines-slf4j` to propagate trace fields: `withContext(MDCContext()) { … }`.

## Testing

- **JUnit 5** + **MockK** (preferred over Mockito for Kotlin) + **AssertJ** or **Kotest assertions**.
- Backtick-quoted test names allowed and encouraged: `` fun `should return 404 when user missing`() ``.
- **Kotest** is fine if the project already uses it — match what's there.
- Spring: `@WebMvcTest` / `@SpringBootTest` work the same as Java (see [`spring-mvc.md`](spring-mvc.md) / [`spring-webflux.md`](spring-webflux.md)).

## Build / Test commands

```bash
./mvnw test                      # unit tests
./mvnw -pl <module> test         # single module
./mvnw verify                    # tests + integration tests
./mvnw spotless:apply            # format, if configured
./mvnw detekt:check              # if detekt plugin configured
```

Use **Maven** by default for Kotlin projects as well. If you're in an existing Gradle Kotlin project, follow the existing build for that repository.

## Interop with Java

- Use `@JvmStatic`, `@JvmOverloads`, `@JvmField` only when Java callers actually need them.
- Default arguments don't generate Java overloads without `@JvmOverloads`.
- `companion object` members aren't `static` from Java's view by default.

## Common pitfalls

- `lateinit var` for non-null mutable that's set later — only when you control init. Don't use for primitives (use `Delegates.notNull()`).
- `object` (singleton) hides global state — use sparingly.
- Overusing extension functions on common types (`String`, `Any?`) pollutes autocomplete.
- `==` is structural (`equals`), `===` is referential. Java's `==` is referential — don't translate literally.
- Forgetting `suspend` propagates: a sync wrapper around a suspend function usually means you're doing it wrong.
