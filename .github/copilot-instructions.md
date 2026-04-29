# Copilot Instructions

> **General rules** (planning, surgical changes, security, commits, workflow) are defined in
> [`AGENTS.md`](../AGENTS.md). Read that first — this file contains only project-specific context.

## Prosjektkontekst

Delt bibliotek av **GitHub Actions reusable workflows** og **composite actions** for alle NASA-prosjekter (Nasjonalbiblioteket). Brukes som `workflow_call`-avhengigheter fra andre repos. Ingen applikasjonskode — kun YAML.

Flat struktur:
- `.github/action/<navn>/action.yml` — composite actions (gjenbrukbare byggeklosser)
- `.github/workflows/*.yaml` — reusable workflows (kallbare via `workflow_call`)
- `resources/` — støttefiler (f.eks. `toolchains.xml`)

## Teknologi

- **Språk:** YAML (GitHub Actions)
- **Java-toolchain:** Temurin JDK 21 (styrt via inputs i actions/workflows, referanse i `resources/toolchains.xml`)
- **Maven:** 3.9.x (satt opp via composite action `setup-java-maven-and-cache`)
- **CI/CD-plattform:** GitHub Actions (self-hosted runners: `self-hosted-linux-test`)
- **Artefakt-repo:** Artifactory (`artifactory-host` input)
- **Cache:** S3-kompatibel cache via `tespkg/actions-cache`
- **Secrets:** HashiCorp Vault (approle-autentisering)
- **Container-registry:** Harbor (`harbor.nb.no`)
- **Deploy:** Helm + Kubernetes (composite actions `helm-deploy`, `kubernetes-deploy`, `manifest-update`)
- **Sikkerhetsskanning:** Gitleaks (secret scanning), CodeQL

## Arkitekturmønstre

To typer byggeklosser:

**Composite actions** (`.github/action/<navn>/action.yml`):
- Innkapsler ett logisk steg (f.eks. Java-oppsett, Helm-deploy)
- Kalles med `uses: NationalLibraryOfNorway/nasa-workflows/.github/action/<navn>@<ref>`
- Inputs er eksplisitt dokumentert i `action.yml`

**Reusable workflows** (`.github/workflows/*.yaml`):
- Komplette CI/CD-pipelines kallbare via `workflow_call`
- Versjoneres med suffikser (`-v3`, `-v4`) for bakoverkompatibilitet
- Consumers peker på en spesifikk versjon/tag

Nye funksjoner legges i **ny versjon** av workflow (f.eks. `build-mvn-spring-app-v5.yaml`) — eksisterende versjoner skal ikke bryte consumers.

## Prosjektspesifikke regler

- **Aldri endre eksisterende versjonerte workflows** på en måte som bryter consumers — lag ny versjon i stedet.
- **Action-pinning:** Alle `uses:`-referanser skal pinnes til commit SHA med versjon i kommentar, f.eks. `uses: actions/setup-java@abc123 # v4.2.0`.
- **Ingen hardkodede hemmeligheter** — alle credentials via secrets/inputs eller Vault.
- **Inputs skal ha `description`** og fornuftige defaults der det er mulig.
- **YAML-formatering:** 2 mellomrom innrykk, ingen tabs.

## Commit-veiledning

- Jira-referanse (`SU-XXXX`) er påkrevd — spør alltid brukeren om Jira-nummer hvis det ikke er oppgitt.
- Øvrige regler i [`AGENTS.md` → `ref/commit.md`](../AGENTS.md).

## Skills

- Skills (`.github/skills/`) skal alltid skrives på **engelsk**.
- Bruk keyword-rik description som forklarer HVA skillen gjør og NÅR den trigges.

## Copilot i pull requests

- **Automatisk review**: Workflowen `copilot-review.yml` ber `@copilot` om review på alle nye PRer automatisk.
- Mention `@copilot` i en PR-kommentar for å be Copilot gjøre endringer direkte i PR-en.
