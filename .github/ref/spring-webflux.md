# Spring Boot WebFlux Reference

Reactive, non-blocking Spring Boot built on Reactor (Netty). Apply on top of [`../AGENTS.md`](../AGENTS.md) and the relevant language ref ([`java.md`](java.md) or [`kotlin.md`](kotlin.md)). For REST design itself, see [`backend-api.md`](backend-api.md). Compare with [`spring-mvc.md`](spring-mvc.md).

## When (and when not) to use WebFlux

- Use when: high-concurrency IO, long-lived streams (SSE, WebSocket), reactive drivers (R2DBC, reactive Mongo, reactive Kafka).
- Don't use when: stack is mostly JDBC/JPA. Wrapping blocking calls defeats the purpose.
- Pick **one** stack per service. Mixing MVC and WebFlux in the same app is supported but rarely worth the complexity.

## The cardinal rule: never block the event loop

- No `.block()` in request paths. Ever.
- No blocking JDBC, blocking HTTP clients, `Thread.sleep`, or filesystem reads on Netty threads.
- If you must call blocking code, isolate it: `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.
- Detect violations with **BlockHound** in tests.

## Controllers

- `@RestController` with handlers returning `Mono<T>` / `Flux<T>`.
- Functional routing (`RouterFunction`) is an alternative — pick one style per project.
- Bind input the same way as MVC: `@RequestBody`, `@PathVariable`, `@RequestParam`, `@Valid`.
- Stream responses with `Flux<T>` + `MediaType.TEXT_EVENT_STREAM_VALUE` for SSE.

```java
@GetMapping("/users/{id}")
Mono<User> get(@PathVariable String id) {
    return service.find(id)
        .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
}
```

## Reactor essentials

- `Mono<T>` — 0..1 element. `Flux<T>` — 0..N elements.
- Operators are lazy: nothing runs until subscribed. Returning the publisher *is* subscribing (Spring does it).
- Compose with `map`, `flatMap`, `concatMap`, `zip`, `then`. `flatMap` interleaves; `concatMap` preserves order.
- Side effects: `doOnNext`, `doOnError`, `doFinally` — for logging/metrics, not business logic.
- Errors: `onErrorResume`, `onErrorMap`. Don't swallow with bare `onErrorReturn` unless you mean it. For HTTP error responses, use `ProblemDetail` (RFC 9457) — see [`spring-error-handling.md`](spring-error-handling.md).
- Context propagation: use `Context` / `contextWrite` for tracing, auth — not `ThreadLocal`.

## Kotlin coroutines

- WebFlux supports `suspend` handlers and `Flow<T>` returns directly.
- Prefer coroutines over raw Reactor in Kotlin code — clearer error handling and cancellation.
- Bridge with `kotlinx-coroutines-reactor` (`awaitSingle`, `asFlow`).

## Persistence

- **R2DBC** for relational DBs reactively. **Reactive Mongo** for Mongo. No JPA — it's blocking.
- Transactions: `TransactionalOperator` or `@Transactional` (works with reactive transaction manager).
- Don't fall back to JDBC repositories in a WebFlux service — see "cardinal rule".

## HTTP client

- `WebClient`, not `RestTemplate` (blocking). Configure timeouts and connection pools explicitly.
- Retry: `.retryWhen(Retry.backoff(...))` with jitter. Limit attempts.
- Circuit-break with Resilience4j reactive operators when calling unreliable downstreams.

## Testing

- **`@WebFluxTest`** — handler slice with `WebTestClient`.
- **`StepVerifier`** for testing publishers directly.
- **`@SpringBootTest(webEnvironment = RANDOM_PORT)`** + `WebTestClient` for integration.
- **BlockHound** as a test dependency to fail fast on accidental blocking.

```java
@WebFluxTest(UserHandler.class)
class UserHandlerTest {
    @Autowired WebTestClient client;
    @MockBean UserService service;

    @Test
    void shouldReturn404WhenMissing() {
        when(service.find("x")).thenReturn(Mono.empty());
        client.get().uri("/users/x").exchange().expectStatus().isNotFound();
    }
}
```

## Observability

- Micrometer + Reactor metrics (`Hooks.enableAutomaticContextPropagation()` for tracing context).
- Log with reactive context — bare `MDC` won't follow a `Mono`.
- Backpressure visible in metrics; watch request rates vs. downstream capacity.

## Common pitfalls

- Calling `.block()` "just for now" — it leaks and breaks under load.
- Forgetting to subscribe (cold publisher never runs). If you build a chain and don't return it, it dies.
- Using `flatMap` where order matters — use `concatMap`.
- Mixing `ThreadLocal`-based libs (security, MDC) without context propagation.
- Unbounded `Flux` without backpressure — OOM under fast producers.
- Wrapping blocking JDBC in `Mono.fromCallable` and calling it "reactive" — only acceptable at the edge, on `boundedElastic`.
