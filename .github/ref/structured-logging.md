# Structured Logging Reference

How to produce logs that are machine-parseable, searchable, and correlatable across services. Apply on top of [`../AGENTS.md`](../AGENTS.md) and the relevant language ref ([`java.md`](java.md), [`kotlin.md`](kotlin.md), [`python.md`](python.md), [`typescript.md`](typescript.md)).

## Why structured logging

Plain text logs are fine for one process you're tailing on your laptop. They fall apart the moment you have:

- Multiple services that need to be correlated (one user request → 5 systems).
- A log aggregator (ELK, Loki, Datadog, Splunk) that indexes by field.
- Alerts that trigger on a specific field, not a regex over a free-form string.
- Compliance requirements that demand consistent fields (`userId`, `traceId`, `eventType`).

Structured logs are emitted as **one JSON object per log event**. The aggregator parses it once; queries operate on fields, not substrings.

```json
{"timestamp":"2026-04-26T10:15:30.123Z","level":"INFO","service":"orders","traceId":"a1b2…","spanId":"c3d4…","userId":"u-42","event":"order.placed","orderId":"o-998","amountMinor":1499,"currency":"NOK","durationMs":42}
```

vs.

```text
2026-04-26 10:15:30 INFO  Order placed for user 42, order id o-998, amount 14.99 NOK in 42ms
```

The first is queryable. The second needs a regex per field, breaks the day someone reformats the message, and can't reliably be split when fields contain spaces or quotes.

## Standard fields

Every log event in production should include these where applicable:

| Field | Purpose |
| --- | --- |
| `timestamp` | ISO 8601 UTC with milliseconds. Set by the logging framework. |
| `level` | `ERROR` / `WARN` / `INFO` / `DEBUG` / `TRACE`. |
| `service` | Service / application name. Constant per process. |
| `version` | Deployed version / git SHA. |
| `env` | `prod` / `staging` / `dev`. |
| `traceId` | Distributed-trace ID (W3C `traceparent`). |
| `spanId` | Current span. |
| `message` | Human-readable summary. Stable per event type, not interpolated with high-cardinality data. |
| `event` | Stable event identifier (`order.placed`, `auth.failed`). Optional but valuable for analytics. |
| Domain fields | `orderId`, `accountId`, `requestId`, etc. — flat keys, not nested. |

Keep keys consistent across services. Document them in a small shared registry if you have many.

## Log levels

- **`ERROR`** — something is broken; an operator may need to act. One per failing operation, not one per stack frame.
- **`WARN`** — recoverable, but worth noticing. Retries, fallbacks, deprecation hits.
- **`INFO`** — meaningful state changes (request started/finished, job done, config loaded). Default level in prod.
- **`DEBUG`** — diagnostic detail. Off in prod by default; toggle per service/component.
- **`TRACE`** — verbose flow. Local development only.

Don't log `INFO` for things that happen thousands of times per second — that's `DEBUG`.

## Correlation across services

Carry a trace context end-to-end so logs from different services line up.

- Use **W3C Trace Context** (`traceparent` header) — supported by OpenTelemetry, Micrometer Tracing, etc.
- Put `traceId` and `spanId` on **every** log event in a request scope.
- Propagate manually across async boundaries (queues, scheduled jobs, coroutines) — context doesn't follow magically.
- Map MDC / context fields onto the JSON output so you don't have to log them by hand.

## Per-language tooling

### Java / Kotlin (Spring Boot 3.4+)

- **SLF4J** as the API. **Logback** with [logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder) for JSON — the standard combination.
- Spring Boot 3.4+ has built-in structured logging: `logging.structured.format.console=ecs` (Elastic Common Schema) or `logstash`. Prefer this over hand-rolling encoders.
- Use `MDC.put("orderId", id)` for request-scoped fields; clear in a filter / `finally`.
- Always use `{}` placeholders, never string concatenation: `log.info("order placed orderId={}", id)`. The arg is only formatted if the level is enabled.
- **Kotlin coroutines:** MDC is thread-local and doesn't survive dispatch. Add `kotlinx-coroutines-slf4j` and wrap with `withContext(MDCContext())` to carry trace fields across suspensions.
- Never `System.out.println` / `e.printStackTrace()` — see [`java.md`](java.md).

### Python

- **stdlib `logging`** + `python-json-logger` for JSON output, or **`structlog`** for a more ergonomic API with field binding.
- Module-level logger: `logger = logging.getLogger(__name__)`.
- Lazy formatting: `logger.info("user %s logged in", user_id)` — keeps formatting cheap when level is disabled.
- For async (`asyncio`), use `contextvars` to propagate request fields.

### Node / Bun (TypeScript)

- **`pino`** is the default — JSON-first, fast, low overhead. `winston` is fine if already in use.
- Library code shouldn't `console.log`. Application entrypoints / CLIs may. See [`typescript.md`](typescript.md).
- Use child loggers (`logger.child({ requestId })`) per request to inherit context.
- Pretty-print only in dev (`pino-pretty`); ship raw JSON in production.

### Other runtimes

Same principles: pick the JSON-emitting logger that's idiomatic, configure level via env, propagate trace context.

## What NOT to log

- **Secrets, tokens, passwords, API keys** — mask or omit. Even at `DEBUG`. Even "just temporarily."
- **PII** — full name, email, phone, address, national ID, payment data. Mask (`u***@example.com`) or omit. If a field is required for support, log a stable opaque ID and look up the PII out-of-band.
- **Full request / response bodies** — they often contain the above. Log size, content-type, status, and a few selected fields. If you must dump bodies (debugging), do it locally and never commit it.
- **High-cardinality data in the `message`** — interpolating an ID into the message defeats event grouping. Put IDs in their own fields.
- **`println` / unstructured prints** — they bypass the logger and break JSON output.

## Output destination

- **Containers / cloud:** log to **stdout**. The platform collects it. Don't write to files inside the container.
- **Long-running on-prem boxes:** files with rotation (`logrotate`) are still fine, but ship to an aggregator.
- One JSON object per line (newline-delimited). No multi-line stack traces split across log lines — fold them into a `stackTrace` field if needed.

## Performance

- Async / non-blocking appender for hot paths (Logback `AsyncAppender`, Log4j2 async loggers, pino's default).
- Don't compute log payloads at levels that are disabled — use placeholders / lazy lambdas.
- Sample very-high-volume events (e.g. one log per 100 cache hits at `DEBUG`).
- Don't serialize huge objects into a single field — pick the fields you need.

## Testing

- Verify shape with a capturing appender or by parsing stdout in an integration test.
- Java: [LogCaptor](https://github.com/Hakky54/log-captor) or a Logback `ListAppender`.
- Python: `caplog` pytest fixture.
- Node/Bun: pino has a built-in test mode (`pino({ level: 'silent' })` + assertions on the destination stream).
- Assert specific fields, not the full message string — message text changes; field semantics shouldn't.

## Exceptions

Log the exception object, not just its message — you need the stack trace. Add context fields that help reproduce the failure:

```java
log.error("payment failed orderId={} provider={}", orderId, provider, ex);
```

```python
logger.error("payment failed", extra={"orderId": order_id, "provider": provider}, exc_info=True)
```

```typescript
logger.error({ orderId, provider, err }, "payment failed");
```

Don't log the same exception at every layer it bubbles through — log it once, close to where it happens, with full context.

## Common pitfalls

- Building log strings with `+` / template literals — formats even when the level is disabled, can leak secrets when interpolated, and loses field structure.
- Logging the same event multiple times as it bubbles through layers — log it once, near the event, with full context.
- Missing `traceId` in async / queue handlers — propagate context explicitly across boundaries.
- Using `INFO` for what should be `DEBUG`, then drowning in volume in prod.
- Logging exception messages without the exception object — you lose the stack. Pass the throwable to the logger (`log.error("...", ex)`) instead of `ex.getMessage()`.
- Custom date formats per service — stick to ISO 8601 UTC.
- Pretty-printing JSON in production — wastes space and breaks line-based ingestion.
