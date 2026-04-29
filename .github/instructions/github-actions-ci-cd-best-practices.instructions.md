---
applyTo: '.github/workflows/*.yml,.github/workflows/*.yaml'
description: 'Guide for building secure and efficient CI/CD workflows using GitHub Actions. Covers Java/Maven builds, secret management via HashiCorp Vault, action pinning, S3 caching, Docker image builds, and deployment best practices.'
---

# GitHub Actions CI/CD Best Practices

Guidelines for GitHub Actions workflows in this project. Org defaults: Java 21, Maven via wrapper, self-hosted runners, HashiCorp Vault for secrets, S3 dependency cache.

## Existing Workflows

<!-- TODO: List your project's workflows -->

| Workflow | Trigger | Purpose |
|---|---|---|
| `{{WORKFLOW_1}}` | {{TRIGGER_1}} | {{PURPOSE_1}} |
| `{{WORKFLOW_2}}` | {{TRIGGER_2}} | {{PURPOSE_2}} |

## Action Pinning

- Pin all actions to a full-length commit SHA with a version comment
- Never use mutable tags (`@v4`, `@main`, `@latest`) — they can be silently moved
- Use Dependabot to automate SHA updates

```yaml
# Good — immutable SHA with version comment
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

# Bad — mutable tag, vulnerable to supply chain attacks
uses: actions/checkout@v4
```

## Permissions

- Set `permissions: contents: read` at workflow level as default
- Override at job level only when needed
- Never grant write permissions unless strictly required

```yaml
permissions:
  contents: read

jobs:
  test:
    runs-on: self-hosted-linux
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

## Runners

- All workflows run on `self-hosted-linux` — never use `ubuntu-latest` or other GitHub-hosted runners

## Secret Management

Secrets are fetched from **HashiCorp Vault** via `hashicorp/vault-action` — not stored as GitHub Secrets directly. Vault secrets with `*` wildcard are exported as **environment variables** (`${{ env.SECRET_NAME }}`).

- Never log or expose secrets in outputs
- Use Gitleaks as pre-commit hook locally to prevent accidental commits
- No hardcoded credentials in code or configuration
- K8s secrets in overlays use `CHANGE_ME` placeholders — real values come from Vault

```yaml
- name: Import secrets from vault
  uses: hashicorp/vault-action@v3
  id: vault
  with:
    url: ${{ secrets.NB_VAULT_URL }}
    method: approle
    roleId: ${{ secrets.NASA_VAULT_ROLE_ID }}
    secretId: ${{ secrets.NASA_VAULT_SECRET_ID }}
    secrets: ${{ secrets.NASA_VAULT_SECRET_PATH }} *
```

### GitHub Secrets (Vault access)

| Secret                   | Purpose                 |
|--------------------------|-------------------------|
| `NB_VAULT_URL`           | Vault server URL        |
| `NASA_VAULT_ROLE`        | Vault role name         |
| `NASA_VAULT_ROLE_ID`     | Vault AppRole role ID   |
| `NASA_VAULT_SECRET_ID`   | Vault AppRole secret ID |
| `NASA_VAULT_SECRET_PATH` | Vault secret path       |

### Secrets from Vault (available as environment variables)

| Secret                  | Purpose                                     |
|-------------------------|---------------------------------------------|
| `ARTIFACTORY_REPO_HOST` | Artifactory hostname for Maven dependencies |
| `ARTIFACTORY_USER`      | Artifactory username                        |
| `ARTIFACTORY_PASS`      | Artifactory password                        |
| `PROXY_NB_HOST`         | NB outbound proxy hostname                  |
| `PROXY_NB_PORT`         | NB outbound proxy port                      |
| `S3_NB_HOST`            | S3 cache endpoint                           |
| `S3_NB_ACCESS_KEY`      | S3 cache access key                         |
| `S3_NB_SECRET_KEY`      | S3 cache secret key                         |
| `S3_NB_CACHE_BUCKET`    | S3 cache bucket                             |

<!-- TODO: Add project-specific secrets below if needed -->

## Java / Maven Workflows

- Use `NationalLibraryOfNorway/nasa-workflows/.github/action/setup-java-maven-and-cache` — sets up JDK, Maven, settings.xml, and S3 cache in one step
- `java-version` defaults to `'21'`, `maven-version` defaults to `'3.9.14'`
- Use Maven wrapper (`./mvnw`) instead of `mvn` for reproducible builds
- Run `./mvnw -B verify -s settings.xml` for build + test in CI
- Upload surefire reports as artifacts for debugging failed tests

```yaml
- name: Setup Java, Maven and cache
  uses: NationalLibraryOfNorway/nasa-workflows/.github/action/setup-java-maven-and-cache@main
  with:
    java-version: '21'
    maven-version: '3.9.14'
    artifactory-host: ${{ env.ARTIFACTORY_REPO_HOST }}
    artifactory-user-nasa: ${{ env.ARTIFACTORY_USER }}
    artifactory-pass-nasa: ${{ env.ARTIFACTORY_PASS }}
    proxy-host-nb: ${{ env.PROXY_NB_HOST }}
    proxy-port-nb: ${{ env.PROXY_NB_PORT }}
    cache-endpoint: ${{ env.S3_NB_HOST }}
    cache-accessKey: ${{ env.S3_NB_ACCESS_KEY }}
    cache-secretKey: ${{ env.S3_NB_SECRET_KEY }}
    cache-bucket: ${{ env.S3_NB_CACHE_BUCKET }}

- name: Build and test
  run: ./mvnw -B verify -s settings.xml

- name: Upload test reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: surefire-reports
    path: '**/target/surefire-reports/'
    retention-days: 7
```

<!-- TODO: For non-Java projects, replace the section above with the appropriate build tool setup -->

## Docker Image Builds

When adding CI/CD for image builds:

- Tag images with `<version>-<git-sha>` for traceability
- Push to Harbor registry (`harbor.nb.no/{{HARBOR_PROJECT}}/{{IMAGE_NAME}}`)
- Never bake secrets into Docker images
- Use multi-stage builds to minimize image size

## Kubernetes Manifest Validation (optional)

<!-- TODO: Remove this section if the project does not use Kubernetes -->

If the project includes Kubernetes manifests:

- Validate all Kustomize overlays with kubeconform using `--strict` mode
- Validate standalone deployment manifests
- Trigger only on changes to `k8s/**` paths

## Caching

- Maven dependencies cached via S3 (handled by `setup-java-maven-and-cache` action)
- Cache key is based on `pom.xml` hash — invalidates when dependencies change

## Artifact Retention

- Set `retention-days` on all uploaded artifacts to manage storage
- Test reports: 7 days is sufficient
- Build artifacts: set based on deployment needs

## Concurrency Control

- Use `concurrency` to cancel outdated PR builds
- Prevent concurrent deployments to same environment

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Workflow Review Checklist

- [ ] Actions pinned to full commit SHAs with version comments
- [ ] `permissions: contents: read` set at workflow level
- [ ] Secrets fetched from Vault via `hashicorp/vault-action`
- [ ] Maven caching enabled via `setup-java-maven-and-cache`
- [ ] Test reports uploaded as artifacts with retention policy
- [ ] Concurrency control configured
- [ ] No hardcoded credentials or secrets
- [ ] `fetch-depth: 1` used for checkout (unless full history needed)
