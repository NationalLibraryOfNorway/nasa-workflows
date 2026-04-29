# Spring Boot MVC Reference

Servlet-based, blocking Spring Boot. Apply on top of [`../AGENTS.md`](../AGENTS.md) and the relevant language ref ([`java.md`](java.md) or [`kotlin.md`](kotlin.md)). For REST design itself, see [`backend-api.md`](backend-api.md).

## When to use MVC vs WebFlux

- MVC: blocking IO, JDBC/JPA, simpler mental model, most existing Spring code.
- WebFlux: non-blocking IO end-to-end, high-concurrency streams, reactive drivers (R2DBC, reactive Mongo). See [`spring-webflux.md`](spring-webflux.md).
- **Don't mix.** Calling blocking JDBC from a WebFlux app on the event loop kills throughput.

## Controllers

- `@RestController` for JSON APIs. `@Controller` only for server-rendered views (Thymeleaf, etc.).
- Thin controllers: parse/validate input → delegate to a service → map result to response. No business logic.
- Use `@RequestMapping` at class level for the base path; HTTP-verb annotations (`@GetMapping`, `@PostMapping`, ...) on methods.
- Bind input with `@RequestBody`, `@PathVariable`, `@RequestParam`. Validate with `@Valid` + Bean Validation annotations on the DTO.
- Return `ResponseEntity<T>` when you need to set status/headers; return the body directly otherwise.

## DTOs

- Separate DTOs from entities. Don't expose JPA entities over HTTP.
- Records (Java) or data classes (Kotlin) for request/response shapes.
- Validation annotations (`@NotNull`, `@Size`, `@Email`, custom) on DTO fields.

## Error handling

- Centralize with `@RestControllerAdvice` + `@ExceptionHandler`. Use `ProblemDetail` (RFC 9457) for the response body — see [`spring-error-handling.md`](spring-error-handling.md). General API error rules in [`backend-api.md`](backend-api.md).
- Don't catch broadly in controllers — let the advice handle it.
- Map domain exceptions to HTTP status codes in one place.

## Services & transactions

- `@Transactional` at the service layer, not the controller or repository. Read-only: `@Transactional(readOnly = true)`.
- Don't `@Transactional` private methods or self-invocations — proxy won't intercept.
- Keep transactions short. No remote calls inside a transaction unless necessary.

## Persistence (JPA)

- Use Spring Data repositories for CRUD. Custom queries via `@Query` or query methods.
- Watch for N+1: use `@EntityGraph` or `JOIN FETCH`.
- `LAZY` relations by default. Access lazy fields inside the transaction, or fetch eagerly when needed for the use case.
- Pagination: accept `Pageable`, return `Page<T>`.

## Testing

- **`@WebMvcTest`** — controller slice. Mocks `@Service` / repositories with `@MockBean`. Use `MockMvc` for requests.
- **`@DataJpaTest`** — repository slice with in-memory or Testcontainers DB.
- **`@SpringBootTest`** — full context. Use sparingly; slow.
- **Testcontainers** for real Postgres/MySQL/Redis in integration tests over H2 mocks.
- Assert HTTP status, body (JSONPath or full deserialization), and side effects separately.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService service;

    @Test
    void shouldReturn404WhenUserMissing() throws Exception {
        when(service.find("x")).thenThrow(new UserNotFoundException("x"));
        mvc.perform(get("/users/x"))
           .andExpect(status().isNotFound());
    }
}
```

## Configuration

- Type-safe config: `@ConfigurationProperties` over scattered `@Value`.
- Profiles (`application-{profile}.yml`) for env differences. Don't branch on profile in code.
- Secrets via env vars (`${MY_SECRET}`), never committed.

## Observability

- Actuator endpoints (`/actuator/health`, `/actuator/metrics`) — restrict exposure in prod.
- Micrometer for metrics. Add `@Timed` / counters on key paths.
- Correlate logs with `traceId`/`spanId` (Spring Boot 3 + Micrometer Tracing handles this).

## Common pitfalls

- Returning entities directly — leaks schema, triggers lazy-load surprises during serialization.
- Calling `@Async` / `@Transactional` methods from within the same class (proxy bypassed).
- Open Session in View — disable (`spring.jpa.open-in-view=false`) and fetch deliberately.
- Filter / interceptor ordering — make it explicit with `@Order`.
- Forgetting CSRF config when adding non-API endpoints.
