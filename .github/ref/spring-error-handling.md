# Spring Error Handling (RFC 9457)

How to do consistent HTTP error responses in Spring Web using `ProblemDetail` per [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html). Apply on top of [`../AGENTS.md`](../AGENTS.md), the relevant Spring ref ([`spring-mvc.md`](spring-mvc.md) / [`spring-webflux.md`](spring-webflux.md)), and [`backend-api.md`](backend-api.md) for general API rules.

> RFC 9457 (July 2023) supersedes RFC 7807. Wire format is largely the same, so existing Spring `ProblemDetail` code keeps working — RFC 9457 mainly clarifies extension behavior and security guidance. **Use RFC 9457 as the reference for new code.**

## Why ProblemDetail

- Single, predictable error shape across the whole API.
- Standardized fields → clients (and tooling like OpenAPI) can rely on them.
- Built into Spring 6 / Spring Boot 3+ — no custom error DTOs needed.
- Content-Type is set automatically: `application/problem+json`.

## Enable in Spring Boot

Default Spring exception handlers return plain error bodies until you opt in:

```yaml
# application.yml
spring:
  mvc:
    problemdetails:
      enabled: true       # MVC apps
  webflux:
    problemdetails:
      enabled: true       # WebFlux apps
```

Enable the property that matches your stack (`mvc` for MVC apps, `webflux` for WebFlux apps). Setting both is usually unnecessary.

With this on, exceptions like `ResponseStatusException`, `MethodArgumentNotValidException`, and others handled by `ResponseEntityExceptionHandler` / `ResponseStatusExceptionHandler` automatically return a `ProblemDetail` body.

## The `ProblemDetail` class

`org.springframework.http.ProblemDetail`. Standard fields:

| Field | Type | Notes |
| --- | --- | --- |
| `type` | URI | Stable identifier for the error class. Default `about:blank`. **Set this** for every non-trivial error. |
| `title` | String | Short, human-readable summary. Keep semantics stable per `type`; localization is allowed. |
| `status` | int | HTTP status. Set by factory methods. |
| `detail` | String | Longer explanation specific to *this* occurrence. Safe to localize. |
| `instance` | URI | Identifies *this* occurrence (often the request path). |

Plus arbitrary **extension members** via `setProperty(name, value)` — e.g. `errors`, `traceId`, `code`.

```java
ProblemDetail pd = ProblemDetail.forStatusAndDetail(
    HttpStatus.NOT_FOUND, "User %s does not exist".formatted(id));
pd.setType(URI.create("https://errors.example.com/user-not-found"));
pd.setTitle("User not found");
pd.setInstance(URI.create("/users/" + id));
pd.setProperty("traceId", MDC.get("traceId"));
return pd;
```

## Centralized handling: `@RestControllerAdvice`

One advice class per service. Map domain exceptions → `ProblemDetail`. Don't catch in controllers.

```java
@RestControllerAdvice
class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    ProblemDetail handleNotFound(UserNotFoundException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("https://errors.example.com/user-not-found"));
        pd.setTitle("User not found");
        pd.setInstance(URI.create(req.getRequestURI()));
        return pd;
    }

    @ExceptionHandler(OptimisticLockException.class)
    ProblemDetail handleConflict(OptimisticLockException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "Resource was modified by another request");
        pd.setType(URI.create("https://errors.example.com/version-conflict"));
        pd.setTitle("Version conflict");
        return pd;
    }
}
```

Extending `ResponseEntityExceptionHandler` gives you sane defaults for Spring's built-in exceptions (binding, validation, type mismatch, etc.); you only override what you need.

## Validation errors

`MethodArgumentNotValidException` (from `@Valid`) is handled by `ResponseEntityExceptionHandler`. Override to enrich with a structured `errors` extension:

```java
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(
        MethodArgumentNotValidException ex, HttpHeaders headers,
        HttpStatusCode status, WebRequest request) {

    List<Map<String, String>> errors = ex.getBindingResult().getFieldErrors().stream()
        .map(fe -> Map.of(
            "field", fe.getField(),
            "message", fe.getDefaultMessage()))
        .toList();

    ProblemDetail pd = ProblemDetail.forStatusAndDetail(
        status, "One or more fields are invalid");
    pd.setType(URI.create("https://errors.example.com/validation"));
    pd.setTitle("Validation failed");
    pd.setProperty("errors", errors);

    return ResponseEntity.status(status).body(pd);
}
```

## Custom exceptions: `ErrorResponseException` / `ErrorResponse`

For exceptions you throw yourself, prefer extending `ErrorResponseException` (or implementing `ErrorResponse`). The exception then *carries* its `ProblemDetail` — Spring renders it without an `@ExceptionHandler` per type.

```java
public class UserNotFoundException extends ErrorResponseException {
    public UserNotFoundException(String id) {
        super(HttpStatus.NOT_FOUND, problem(id), null);
    }

    private static ProblemDetail problem(String id) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, "User %s does not exist".formatted(id));
        pd.setType(URI.create("https://errors.example.com/user-not-found"));
        pd.setTitle("User not found");
        return pd;
    }
}
```

This works in both MVC and WebFlux without extra wiring.

## WebFlux

Same `ProblemDetail` API. Differences:

- Handler returns `Mono<ProblemDetail>` or `Mono<ResponseEntity<ProblemDetail>>`.
- Override `ResponseEntityExceptionHandler` (it works in WebFlux too) **or** `ResponseStatusExceptionHandler` for low-level customization.
- Don't block in handlers — building a `ProblemDetail` is sync and safe; downstream lookups must stay reactive.
- See [`spring-webflux.md`](spring-webflux.md) for non-blocking constraints.

## `type` URIs

- Use a stable, dereferenceable URI per error class — e.g. `https://errors.example.com/user-not-found`.
- Document each type in your API spec (OpenAPI `responses`, or a `/docs/errors/...` page).
- Don't change the URI without versioning — clients may match on it.
- `about:blank` (the default) is fine for one-off generic errors but useless to clients matching on type.

## Security: don't leak

- No stack traces in `detail`. No SQL, no file paths, no internal class names.
- Log the full exception server-side with a `traceId`; surface only the `traceId` in the response so support can correlate.
- For `4xx`: tell the client what's wrong with their request. For `5xx`: a generic message + `traceId`.
- Be careful with authorization errors — don't reveal whether a resource exists if the caller isn't allowed to know (use `404` over `403` when membership is sensitive).

## Testing

MVC:

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserService service;

    @Test
    void shouldReturnProblemDetailWhenMissing() throws Exception {
        when(service.find("x")).thenThrow(new UserNotFoundException("x"));

        mvc.perform(get("/users/x"))
           .andExpect(status().isNotFound())
           .andExpect(header().string("Content-Type", "application/problem+json"))
           .andExpect(jsonPath("$.type").value("https://errors.example.com/user-not-found"))
           .andExpect(jsonPath("$.title").value("User not found"))
           .andExpect(jsonPath("$.status").value(404));
    }
}
```

WebFlux: same shape, use `WebTestClient` and `.expectHeader().contentType(MediaType.APPLICATION_PROBLEM_JSON)`.

## Common pitfalls

- Forgetting `spring.mvc.problemdetails.enabled=true` (or WebFlux equivalent) → Spring's built-in handlers return the legacy error body.
- Mixing `ProblemDetail` and ad-hoc error DTOs in the same API — pick one, be consistent.
- Putting sensitive data in `detail` (stack trace, internal IDs, user PII).
- Changing `title` semantics for the same `type` across responses. Localization is fine, but meaning should stay stable per `type`.
- Setting `type` to `about:blank` everywhere — clients can't distinguish error classes.
- Returning `200 OK` with a `ProblemDetail` body. The `status` field must match the HTTP status.
- Forgetting `Content-Type: application/problem+json` when hand-rolling responses (factory methods + `ResponseEntity` set it correctly).
