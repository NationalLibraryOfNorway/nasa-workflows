# Backend API Reference

Language-agnostic rules for HTTP/REST APIs. Apply on top of [`../AGENTS.md`](../AGENTS.md). Combine with the relevant framework ref ([`spring-mvc.md`](spring-mvc.md), [`spring-webflux.md`](spring-webflux.md), or your stack).

## URL design

- Resources are nouns, plural: `/users`, `/orders`. Verbs only for actions that don't fit CRUD: `POST /orders/{id}/cancel`.
- Hierarchy reflects ownership: `/users/{id}/orders`. Don't go deeper than 2 levels — flatten with query params.
- Lowercase, hyphen-separated: `/payment-methods`, not `/PaymentMethods`.
- Filtering, sorting, pagination via query string: `?status=open&sort=-createdAt&page=2&size=20`.

## HTTP methods

| Method | Purpose | Idempotent | Safe |
| --- | --- | --- | --- |
| `GET` | Read | Yes | Yes |
| `POST` | Create / non-idempotent action | No | No |
| `PUT` | Replace | Yes | No |
| `PATCH` | Partial update | No (typically) | No |
| `DELETE` | Remove | Yes | No |

- Don't use `GET` for state changes.
- `PUT` replaces the whole resource; `PATCH` updates fields. Be consistent across the API.

## Status codes

Use a small, predictable set:

- `200 OK` — success with body.
- `201 Created` — resource created. Include `Location` header.
- `202 Accepted` — async accepted; processing later.
- `204 No Content` — success, no body (typical for `DELETE`).
- `400 Bad Request` — malformed/invalid input.
- `401 Unauthorized` — missing/invalid auth.
- `403 Forbidden` — authenticated but not allowed.
- `404 Not Found` — resource doesn't exist.
- `409 Conflict` — state conflict (duplicate, version mismatch).
- `422 Unprocessable Entity` — semantic validation failure (optional, vs `400`).
- `429 Too Many Requests` — rate-limited. Include `Retry-After`.
- `500 Internal Server Error` — unexpected failure.
- `503 Service Unavailable` — downstream / overloaded.

Don't invent codes. Don't return `200` with `{"error": ...}`.

## Request / response

- JSON by default. `application/json; charset=utf-8`.
- Field naming: `camelCase` is most common — match existing API style if extending.
- Timestamps: ISO 8601 in UTC (`2026-04-26T10:15:30Z`). Don't return epoch millis unless the client expects it.
- Money: minor units as integers (`1499` for 14.99 NOK) + currency code, or string decimal — never float.
- Enums: stable string values (`"PENDING"`, `"APPROVED"`). Don't change them silently.
- Don't expose internal IDs (DB sequence) when an opaque ID would do.

## Validation

- Validate at the boundary. Reject early with `400`/`422`.
- Return all validation errors at once when feasible — don't stop at the first.

## Error responses

Use a consistent shape. [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html) is the current standard (it supersedes RFC 7807). For Spring services, see [`spring-error-handling.md`](spring-error-handling.md). Example body:

```json
{
  "type": "https://example.com/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields are invalid",
  "instance": "/users",
  "errors": [
    { "field": "email", "message": "must be a valid email" }
  ]
}
```

- Don't leak stack traces, SQL, or internal paths in production responses.
- Log the full error server-side with a correlation ID; return the ID to the client.

## Pagination

- Cursor-based for large/changing datasets: `?cursor=abc&limit=50` → `{ "items": [...], "nextCursor": "..." }`.
- Offset-based is fine for small/stable sets: `?page=0&size=50`.
- Always return total only when cheap — counting can be expensive.
- Cap `limit`/`size` server-side.

## Versioning

- URL prefix (`/v1/users`) — simplest, most common.
- Header (`Accept: application/vnd.example.v1+json`) — cleaner URLs, harder to debug.
- Pick one. Don't break v1 — add v2.
- Deprecation: `Deprecation` and `Sunset` headers, plus changelog.

## Idempotency

- `GET`, `PUT`, `DELETE` are idempotent by HTTP definition — design accordingly.
- For `POST` create operations exposed to retries (payments, orders), accept an `Idempotency-Key` header and dedupe server-side.

## Auth

- Bearer tokens (JWT or opaque) in `Authorization: Bearer ...`. No tokens in URLs.
- Validate signature, expiry, audience, and issuer on every request.
- Authorization (what user can do) is separate from authentication (who they are). Enforce both.
- CORS: explicit origin allowlist. No `Access-Control-Allow-Origin: *` for credentialed endpoints.

## Security

- HTTPS only in production. Redirect / HSTS.
- Rate-limit per principal and per IP.
- Validate `Content-Type` and `Content-Length`. Cap body size.
- Never log secrets, tokens, or full PII. Mask or redact.
- SQL: parameterized only. Same for NoSQL queries built from user input.
- See OWASP API Security Top 10 — broken object-level auth (BOLA) is the most common API bug.

## Observability

- Structured JSON logs with `traceId`, `spanId`, `userId` (when safe) — see [`structured-logging.md`](structured-logging.md).
- Metrics: request rate, error rate, latency (p50/p95/p99) per endpoint.
- Health endpoints: liveness (process up) vs. readiness (deps reachable). Don't conflate.

## Documentation

- OpenAPI / Swagger spec, generated from code or hand-written but kept in sync.
- Document every status code an endpoint can return, not just `200`.
- Examples for request and response bodies.

## Common pitfalls

- Returning DB entities directly — couples API to schema.
- Inconsistent naming/casing across endpoints.
- 200 with error body — clients can't rely on status codes.
- Leaking pagination internals (DB offset) as cursor.
- Forgetting `OPTIONS` / CORS preflight on new endpoints.
- N+1 queries inside response serialization.
- Breaking changes shipped without a version bump.
