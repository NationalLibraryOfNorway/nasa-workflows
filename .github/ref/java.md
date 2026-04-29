# Java Reference

Rules for Java work. Apply on top of [`../AGENTS.md`](../AGENTS.md).

## Style

- Clear, descriptive names. Small, readable methods.
- Side effects local and explicit.
- All public classes have Javadoc. Document *why*, not *what*.
- Prefer `Optional` over null returns at API boundaries.
- Prefer immutable types (`record`, `final` fields) where reasonable.

## Logging

- Use **SLF4J** (`org.slf4j.Logger`). Emit JSON in production — see [`structured-logging.md`](structured-logging.md).
- **Never** `System.out.println` / `System.err.println`. Flag if seen in existing code.
- Log levels: `error` (action needed), `warn` (recoverable), `info` (state change), `debug` (diagnostics).
- Log parameters with `{}` placeholders, not string concatenation: `log.info("user {} logged in", id)`.

## Comments

- No `TODO`/`FIXME`/`HACK` without an issue reference (e.g. `// TODO(SU-1234): ...`).
- Don't restate code. Explain non-obvious *why*.

## Testing

- **JUnit 5** + **Mockito** + **AssertJ** (preferred over Hamcrest).
- Spring controllers: `@WebMvcTest` (MVC) or `@WebFluxTest` (WebFlux) for slice tests, `@SpringBootTest` for full integration.
- Mock external services in integration tests.
- One scenario per test method. Cover happy path, errors, edge cases separately.
- Pattern: **Arrange-Act-Assert** with descriptive names (`shouldReturnXWhenY`).

## Build

- **Maven** — always. New Java projects use Maven; existing Gradle projects should be migrated. Until migrated, follow the project's existing build, but don't add new Gradle modules.
- Use the **Maven Wrapper** (`./mvnw`) committed at repo root so the build is reproducible without a system Maven install.
- Multi-module: parent `pom.xml` aggregates `<modules>`; share versions via `<dependencyManagement>` and `<properties>`.
- Format with **Spotless** (`spotless-maven-plugin`). Run in CI.
- Pin plugin versions explicitly. Don't rely on Maven defaults.

## NB Spring baseline (new projects and upgrades)

Use this baseline when creating new NB Spring services and when aligning existing services to the shared platform.
Supports both Spring MVC and Spring WebFlux services.

- Java: `25`
- Spring Boot parent: `org.springframework.boot:spring-boot-starter-parent:4.0.5`
- Spring Cloud BOM: `org.springframework.cloud:spring-cloud-dependencies:2025.1.1`
- Project layout: Maven multi-module (`*-model`, `*-rest`, optional `*-it`)

### Shared runtime stack

- `org.springframework.boot:spring-boot-starter-jackson`
- `org.springframework.boot:spring-boot-starter-hateoas`
- `org.springframework.cloud:spring-cloud-starter-kubernetes-client-all`
- `org.springframework.cloud:spring-cloud-starter-loadbalancer`
- `org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j`
- `org.springframework.cloud:spring-cloud-starter-circuitbreaker-spring-retry`
- `io.micrometer:micrometer-tracing`, `io.micrometer:micrometer-tracing-bridge-brave`, `io.zipkin.reporter2:zipkin-reporter-brave`
- `org.zalando:logbook-spring-boot-starter`
- `org.springframework:spring-oxm`, `jakarta.xml.bind:jakarta.xml.bind-api`, `org.glassfish.jaxb:jaxb-runtime`

### Web stack variant (choose one)

- MVC profile:
  - `org.springframework.boot:spring-boot-starter-webmvc` (legacy services may still use `spring-boot-starter-web`)
  - `org.springdoc:springdoc-openapi-starter-webmvc-ui`
- WebFlux profile:
  - `org.springframework.boot:spring-boot-starter-webflux`
  - `org.springdoc:springdoc-openapi-starter-webflux-ui`
  - `org.zalando:logbook-netty` (when Netty request/response logging is needed)

### WebFlux baseline additions

- Keep reactive data/messaging dependencies aligned with the service domain (for example Mongo Reactive, Reactor Kafka).
- Prefer WebFlux-native test setup (`@WebFluxTest`, `reactor-test`, and reactive Testcontainers modules as needed).
- See `ref/spring-webflux.md` for implementation patterns and non-blocking constraints.

### Shared test stack

- `org.springframework.boot:spring-boot-starter-test`
- `org.springframework.boot:spring-boot-resttestclient` (when using dedicated HTTP integration tests)
- `org.springframework.boot:spring-boot-restclient` (when using dedicated HTTP integration tests)
- `org.testcontainers:testcontainers`, `org.testcontainers:testcontainers-junit-jupiter`
- `com.squareup.okhttp3:mockwebserver3-junit5`

### Shared build/quality plugins

- `org.springframework.boot:spring-boot-maven-plugin`
- `org.jacoco:jacoco-maven-plugin`
- `com.diffplug.spotless:spotless-maven-plugin`
- `org.cyclonedx:cyclonedx-maven-plugin`
- `org.owasp:dependency-check-maven` (fail threshold CVSS `8`)
- `io.kokuwa.maven:helm-maven-plugin` (for services packaged/deployed with Helm)

### Upgrade checklist

- Update root parent to Spring Boot `4.0.5` and set `java.version` to `25`.
- Keep/import Spring Cloud BOM `2025.1.1` in `dependencyManagement`.
- Remove redundant version pins for Spring/Spring Cloud dependencies covered by BOMs.
- Run full verification: `./mvnw clean verify` in project root.

## Build / Test commands

```bash
./mvnw clean install            # full build, no tests skipped
./mvnw test                     # unit tests
./mvnw -pl <module> test        # single module
./mvnw verify                   # tests + integration tests (failsafe)
./mvnw spotless:apply           # format
./mvnw dependency:tree          # debug dep conflicts
./mvnw versions:display-dependency-updates   # check for upgrades
```

Run integration tests via the **Failsafe** plugin (`*IT.java` naming), unit tests via Surefire (`*Test.java`). Don't put slow/integration tests in Surefire.

## Security

- Never hardcode secrets. Use `@Value("${...}")` / env vars.
- Validate input at REST boundaries. Use Bean Validation (`@Valid`, `@NotNull`).
- Parameterized queries only. No string-concatenated SQL.

## Common pitfalls

- Mutable static state — avoid.
- Catching `Exception` / `Throwable` broadly — narrow it.
- `equals`/`hashCode` mismatch — use IDE generation or Lombok / records.
- Time: prefer `java.time` (`Instant`, `LocalDate`). Avoid `Date`/`Calendar`.
