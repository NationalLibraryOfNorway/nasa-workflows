# Security Reference

Rules for writing secure code, managing dependencies, and handling vulnerabilities. Apply on top of [`../AGENTS.md`](../AGENTS.md). Combine with the language ref you're working in ([`java.md`](java.md), [`kotlin.md`](kotlin.md), [`python.md`](python.md), [`typescript.md`](typescript.md)) and [`backend-api.md`](backend-api.md) for HTTP-specific rules.

## Core rules

- **Never hardcode** secrets, passwords, API keys, tokens, connection strings. Env vars only.
- **Validate at the boundary** — every external input (HTTP, queue, file, env). Reject early.
- **Parameterize queries** — SQL, NoSQL, LDAP, shell. No string concatenation with user input.
- **Encode at the sink** — HTML escape on render, URL-encode in URLs, JSON-encode in JSON.
- **Fail secure** — deny by default; an error in auth/authz means deny, not allow.
- **Least privilege** — services, DB users, cloud roles get the minimum they need.
- **Don't log secrets, tokens, or full PII**. Mask or redact. See logging in your language ref.
- Spot a hardcoded secret in existing code? Flag it immediately. Don't commit on top of it — rotate first.

## OWASP Top 10 (2021)

Web app baseline. Hit each one in code review:

1. **A01 Broken Access Control** — enforce authorization on every protected route, not only at the gateway. BOLA (broken object-level auth) is the most common API bug.
2. **A02 Cryptographic Failures** — TLS in transit; encrypt at rest where required; strong algorithms only (AES-GCM, Argon2/bcrypt for passwords). No MD5/SHA1 for security.
3. **A03 Injection** — SQL, NoSQL, LDAP, OS command, template. Parameterize, never concatenate.
4. **A04 Insecure Design** — threat-model new flows. Rate limits, lockouts, replay protection where it matters.
5. **A05 Security Misconfiguration** — no defaults in prod (default creds, debug endpoints, verbose errors). Set HSTS, CSP, `X-Content-Type-Options`.
6. **A06 Vulnerable & Outdated Components** — see [Dependency / supply chain](#dependency--supply-chain) below.
7. **A07 Identification & Auth Failures** — strong password rules, MFA where appropriate, session expiry, secure cookie flags.
8. **A08 Software & Data Integrity Failures** — sign artifacts, verify checksums, lock dependency versions, beware deserialization of untrusted data.
9. **A09 Logging & Monitoring Failures** — log auth events, access denied, validation failures. Without logs you can't detect a breach.
10. **A10 SSRF** — validate outbound URLs from user input. Allowlist hosts/schemes when possible.

## OWASP API Security Top 10 (2023)

For HTTP/JSON APIs, also check:

- **API1 BOLA** — authorize per-object, not per-endpoint.
- **API2 Broken Authentication** — token validation: signature, expiry, audience, issuer.
- **API3 Broken Object Property Level Auth** — don't blindly bind request bodies to entities (mass assignment). Use DTOs.
- **API4 Unrestricted Resource Consumption** — pagination caps, rate limits, body-size limits, query depth/complexity.
- **API5 Broken Function Level Auth** — admin endpoints need admin checks, not just authenticated checks.
- **API8 Security Misconfiguration** — see A05.
- **API10 Unsafe Consumption of APIs** — treat upstream API responses as untrusted input.

## Dependency / supply chain

Vulnerable third-party code is the most common production CVE source. Two complementary defenses:

- **Scan** dependencies on every build. Block on known-exploitable CVEs.
- **Inventory** what you ship via SBOM (Software Bill of Materials). Without an SBOM you can't answer *"are we affected?"* when a new CVE drops.

**CVSS threshold: fail the build at CVSS 8.0+** (matches the NB Spring baseline in [`java.md`](java.md)). Lower scores get tracked but don't block.

Suppress false positives explicitly with a justification + issue reference. Never blanket-ignore. Re-evaluate suppressions on every dep upgrade — the underlying issue may now apply.

## SBOM with CycloneDX (Java)

[CycloneDX](https://cyclonedx.org/) is the SBOM format. The `cyclonedx-maven-plugin` generates `target/bom.json` (and `bom.xml`) during the build.

```xml
<plugin>
  <groupId>org.cyclonedx</groupId>
  <artifactId>cyclonedx-maven-plugin</artifactId>
  <executions>
    <execution>
      <phase>package</phase>
      <goals><goal>makeAggregateBom</goal></goals>
    </execution>
  </executions>
  <configuration>
    <outputFormat>json</outputFormat>
    <includeBomSerialNumber>true</includeBomSerialNumber>
    <projectType>application</projectType>
  </configuration>
</plugin>
```

Run `./mvnw package` → verify `target/bom.json` exists and lists your dependencies. Multi-module: use `makeAggregateBom` at the root so the SBOM covers all submodules.

## Dependency Track

[Dependency Track](https://dependencytrack.org/) is the central portal that ingests SBOMs, correlates against NVD / OSS Index / GitHub advisories, and tracks risk per project over time.

CI uploads the SBOM after a successful build:

```bash
curl -X POST "$DTRACK_URL/api/v1/bom" \
  -H "X-Api-Key: $DTRACK_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F "project=$DTRACK_PROJECT_UUID" \
  -F "bom=@target/bom.json"
```

Use the team's actual env var names from the CI pipeline templates. Pipeline policy in Dependency Track decides what blocks a release — don't silence a finding by disabling the upload; fix the dep or file a suppression with a reason.

## dependency-check-maven (OWASP)

Local/CI scan against the NVD CVE database. Fails the build above the configured CVSS threshold.

```xml
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <configuration>
    <failBuildOnCVSS>8</failBuildOnCVSS>
    <nvdApiKeyEnvironmentVariable>NVD_API_KEY</nvdApiKeyEnvironmentVariable>
    <suppressionFiles>
      <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
    </suppressionFiles>
  </configuration>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
    </execution>
  </executions>
</plugin>
```

Run: `./mvnw dependency-check:check`. Set `NVD_API_KEY` to avoid rate-limit throttling (free key from nvd.nist.gov).

### Suppressions

Each suppression must have a reason and an issue reference. Don't blanket-suppress.

```xml
<suppress>
  <notes>SU-1234 — false positive: CVE matches package name only, our usage path is unaffected.</notes>
  <packageUrl regex="true">^pkg:maven/com\.example/lib@.*$</packageUrl>
  <cve>CVE-2024-12345</cve>
</suppress>
```

Suppression files live at the repo root (`dependency-check-suppressions.xml`) and are checked into git so reviewers see them.

## Local CVE check (grype + CycloneDX)

Quick pre-flight on the dev machine before push. Builds a CycloneDX SBOM and lets [grype](https://github.com/anchore/grype) match it against Anchore's DB (NVD + GHSA + distro feeds). Complements — does not replace — the CI gate on `dependency-check-maven`.

Requires local install: `brew install grype` (or `curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh`).

```bash
mvn -q --batch-mode --no-transfer-progress \
    -Dmaven.test.skip=true -Ddependency-check.skip=true \
    clean org.cyclonedx:cyclonedx-maven-plugin:2.9.1:makeAggregateBom
grype sbom:target/bom.json --sort-by severity --scope all-layers --by-cve
```

What the flags do:

- `-Dmaven.test.skip=true -Ddependency-check.skip=true` — skip tests and the OWASP scan; only resolve dependencies and build the SBOM
- `makeAggregateBom` — one SBOM covering all modules (multi-module projects)
- `--sort-by severity` — critical first
- `--scope all-layers` — include transitive dependencies
- `--by-cve` — group by CVE ID (one CVE may affect multiple packages)

Want a stricter local gate than CI? Add `--fail-on high` (blocks from CVSS 7.0; the CI gate stays at 8.0):

```bash
grype sbom:target/bom.json --fail-on high --sort-by severity --by-cve
```

Suppress locally via `~/.grype.yaml` or a repo-level `.grype.yaml` — but keep `dependency-check-suppressions.xml` as the source of truth in CI; don't duplicate without reason.

## Other stacks

- **Python** — `pip-audit` (or `uv pip audit`) against the PyPI advisory DB. CycloneDX SBOM via `cyclonedx-py`. See [`python.md`](python.md).
- **TypeScript / Bun** — `bun audit` for advisories. CycloneDX SBOM via `@cyclonedx/cyclonedx-npm` (works with Bun-resolved lockfiles). See [`typescript.md`](typescript.md).
- **All stacks** — upload SBOMs to Dependency Track via the same endpoint; only the generator changes.

## Pre-commit & repo hygiene

- **gitleaks** runs as a pre-commit hook. Don't bypass with `--no-verify` to push a "quick fix" containing a leaked key — rotate the key first, then commit clean.
- `.gitignore` must exclude `target/`, build outputs, `*.env`, IDE/editor caches, and key material (`*.pem`, `*.key`, `*.pfx`, `*.p12`).
- Never commit `application-local.yml`-style files with real credentials. Use `application-local.yml.example` with placeholders.
- Don't commit generated SBOMs (`target/bom.json` is ignored via `target/`); the CI pipeline produces and publishes them.

## Reporting & response

- Found a real vulnerability in a dependency we use? Open an issue with the CVE ID and the affected modules. Don't quietly bump the version in an unrelated commit — make the fix traceable.
- Found a vulnerability in our own code? Don't describe the exploit in a public commit message. File the issue privately, fix, then reference the ticket.

## Common pitfalls

- Shipping without an SBOM — when a CVE drops you can't tell if you're affected.
- Snoozing CVE findings without a ticket or expiry — the suppression list rots and hides real issues.
- Trusting transitive dependencies because the direct dep "looks fine" — scanners read the full graph; so should you.
- Mass assignment: binding `@RequestBody` straight to a JPA entity and exposing fields the user shouldn't set (`isAdmin`, `id`).
- Catching `Exception` and returning a generic 500 — losing the real error and silently bypassing security checks above.
- Checking auth at the gateway only — internal services must re-verify tokens; never trust upstream headers blindly.
- Using `Math.random()` / `Random` for security tokens — use `SecureRandom` (or the language equivalent).
- Logging request bodies "for debugging" in prod — leaks tokens, PII, payment details.
