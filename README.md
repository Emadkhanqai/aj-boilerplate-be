<div align="center">

<img src="docs/assets/banner.png" alt="Al Jazeera engineering banner: the words &quot;Clone. Spec. Ship.&quot; set beside a terminal window showing a spec-driven agent run in progress and a quality gate reporting a pass" width="900">

# Agentic Backend Boilerplate

**A .NET 10 API starting point that ships with its own engineering guardrails.**

[![.NET](https://img.shields.io/badge/.NET-10-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![SQL Server](https://img.shields.io/badge/SQL_Server-EF_Core_10-CC2927?logo=microsoftsqlserver&logoColor=white)](https://learn.microsoft.com/ef/core/)
[![Keycloak](https://img.shields.io/badge/AuthZ-Keycloak-4D4D4D)](https://www.keycloak.org/)
[![Licence](https://img.shields.io/badge/licence-not%20set-lightgrey)](#licence)

</div>

---

## What this is

Most "starter templates" give you a folder structure and leave the hard parts — layering that
actually holds, an API contract consumers can trust, a quality gate that blocks bad merges,
infrastructure you can read — as an exercise. This one ships those parts working, with a single
sample entity proving the whole path end to end.

It is also built to be driven by [Claude Code](https://claude.com/claude-code). A `.claude/`
harness ships **committed, not gitignored**: hooks that format on save, block dangerous shell
commands, protect sensitive files, run the affected tests, scan for secrets, and gate a push on
the quality gate — plus slash commands for the recurring work.

It contains **no business domain**. The one sample entity, `Item`, is designed to be deleted on
your first day. The other module that ships — the "what's new" feature spotlight — is not a
sample: it is a small piece of product plumbing most applications end up needing, and it is meant
to be kept.

## Why it exists

Every new service re-litigates the same decisions: where do business rules live, what shape is
an error response, who owns the API contract, what blocks a merge, how do we get to two clouds
without forking the codebase. Those decisions are made here, written down as
[ADRs](docs/adr/), and enforced by tests and CI rather than by convention.

## What you get

- **Layered Clean Architecture** — `Domain` → `Application` → `Contracts` → `Infrastructure` →
  `Api`, with an architecture test suite that fails the build when the dependency direction is
  violated. It checks compiled assembly references, so the rule cannot be talked around.
- **A consistent API envelope** — every response is `ApiResponse<T>` with a stable
  machine-readable `code`, a `traceId`, and a documented status-code contract. Exceptions map
  to responses through an `IExceptionHandler` chain, not through try/catch in controllers.
- **OpenAPI as the published contract** — the document is generated from the controllers and
  the contract types, and consumers generate their clients from it. A missing
  `[ProducesResponseType]` is a bug. See [docs/api/](docs/api/).
- **EF Core migration workflow** — MSSQL, migration-based, with two migrations in the box
  (`InitialCreate` and `AddFeatureAnnouncements`) so the workflow is demonstrated rather than
  described.
- **A "what's new" feature spotlight** — server-side announcements that surface to each user
  exactly once, on the routes you bind them to, with the dismissal recorded per user so it
  survives cleared browser storage and a second device. Two endpoints, two tables, no seeded
  rows: each announcement ships as its own INSERT-only migration, and no service code changes to
  add one. See [docs/whats-new.md](docs/whats-new.md).
- **Real integration tests** — the suite starts its own SQL Server through Testcontainers,
  applies the migrations, and boots the real pipeline. There is no in-memory provider anywhere,
  on purpose. 245 tests ship green: 185 unit, 9 architecture, 51 integration.
- **Optimistic concurrency, outbox and inbox** — `rowversion` on every audited entity, and the
  messaging tables wired end to end rather than left as a TODO.
- **Two clouds, one switch** — `CLOUD_PROVIDER=gcp|azure` selects the secrets provider and the
  identity issuer at the composition root, and selects which `infra/` tree deploys. Terraform
  for GCP, Bicep for Azure, provisioning the same logical shape.
- **A real quality gate** — build with warnings as errors, `dotnet format` verification, unit +
  integration + architecture tests, SonarQube Community Build (zero new
  Blocker/Critical/Major, ≥80% coverage on new code), Gitleaks, CodeQL, dependency
  vulnerability scanning.
- **The agentic harness** — 8 hooks and the `/spec`, `/task`, `/qa`, `/review`, `/implement`,
  `/pre-push`, `/quality-gate`, `/new-migration` commands, all committed.
- **A written process** — a [five-stage workflow](docs/workflow.md), a
  [Definition of Done](docs/definition-of-done.md), a [spec template](docs/specs/TEMPLATE.md),
  and a [Day-1 onboarding checklist](docs/onboarding.md).
- **An architecture guide that matches the code** — [docs/architecture.md](docs/architecture.md)
  walks every layer: what may and may not live there, what it depends on, the rule the
  architecture tests actually enforce, a real example from the sample slice, and the mistake
  newcomers make with it.

What it is **not**: a platform, a CMS, an auth server, or a deployment you can apply as-is.
`infra/` ships as reviewed IaC with no state and no real project identifiers — you configure a
state backend and your own values before the first `apply`.

## Quickstart (about 5 minutes)

**Prerequisites:** .NET SDK 10, Docker (for the local database and the integration suite), and
a Keycloak realm if you want authorization enforced locally.

```bash
# 1 — clone
git clone https://github.com/Emadkhanqai/aj-boilerplate-be.git
cd aj-boilerplate-be

# 2 — configure. Export what the API needs, or put it in a local .env — either way
#     nothing here is committed; .gitignore already excludes .env and .env.*
cp .env.example .env               # then fill in MSSQL_SA_PASSWORD
export CLOUD_PROVIDER=gcp          # or: azure
export ConnectionStrings__Default='Server=localhost,1433;Database=AjBoilerplate;User Id=sa;Password=<your-local-password>;TrustServerCertificate=True;'
export ConnectionStrings__Redis='localhost:6379'
export APP_DB_CONNECTION="$ConnectionStrings__Default"   # what the design-time factory reads

# 3 — local dependencies
docker network create app-net      # once per host
docker compose up -d db redis

# 4 — database. Apply both migrations: InitialCreate and AddFeatureAnnouncements.
dotnet tool install --global dotnet-ef
dotnet ef database update \
  --project        src/AjBoilerplate.Infrastructure \
  --startup-project src/AjBoilerplate.Api

# 5 — run the API  →  http://localhost:5080
dotnet run --project src/AjBoilerplate.Api
```

Open <http://localhost:5080/swagger> and exercise **Items** — create, read, update, and delete a
row. That round trip runs the real middleware order, the envelope filter, the controller, the
use-case service, the repository, and the migration you just applied. **Features** is there too;
it answers an empty array until you ship your first announcement, because the migration seeds no
rows. (Endpoints are
`[Authorize]`d, so you will get a `401` until you point the API at your identity provider and
Keycloak realm — which is the correct answer, not a broken one.)

Then verify the gate is green before you change anything:

```bash
dotnet build  -warnaserror
dotnet format --verify-no-changes
dotnet test                        # all three suites; the integration one needs Docker
```

Full command reference: [CLAUDE.md](CLAUDE.md).

## Repository map

```
.
├── CLAUDE.md              project context for Claude Code — read this first
├── AjBoilerplate.slnx     the solution: 5 source projects, 3 test projects
├── .claude/               committed agentic harness: hooks, commands, standards, agents
├── .mcp.json              MCP server configuration
├── src/
│   ├── AjBoilerplate.Domain/          entities and invariants — references nothing internal
│   ├── AjBoilerplate.Application/     use cases, ports, validation
│   ├── AjBoilerplate.Contracts/       wire DTOs and the response envelope
│   ├── AjBoilerplate.Infrastructure/  EF Core, cache, transports, cloud secrets
│   └── AjBoilerplate.Api/             controllers, middleware, auth, composition root
├── tests/
│   ├── AjBoilerplate.UnitTests/
│   ├── AjBoilerplate.IntegrationTests/    Testcontainers SQL Server
│   └── AjBoilerplate.ArchitectureTests/   the dependency rule, as compiled-reference assertions
├── docs/
│   ├── adr/               architecture decision records (+ template)
│   ├── specs/             feature specs (+ template)
│   ├── api/               how the OpenAPI contract is produced and consumed
│   ├── handoff/           session handoffs written by the Stop hook
│   ├── architecture.md    every layer, and why each boundary exists
│   ├── whats-new.md       the feature-spotlight module, end to end
│   ├── onboarding.md      Day-1 checklist
│   ├── workflow.md        Spec → Plan → Execute → Verify → Review, with diagrams
│   └── definition-of-done.md
├── .github/
│   ├── workflows/         backend-ci · deploy (+ its reusable per-environment job)
│   └── gitleaks.toml      secret-scanning config; extends the default ruleset
├── infra/
│   ├── gcp/               Terraform: Cloud Run, Cloud SQL, Memorystore, Secret Manager
│   └── azure/             Bicep: Container Apps, Azure SQL, Cache for Redis, Key Vault
├── Dockerfile
└── docker-compose.yml     local SQL Server, Redis, the API, and a migrate profile
```

## How work flows here

Every change — a bug fix, a feature, a refactor — moves through the same five stages, and the same
gates. The solid path is what you do; the shaded gates fire whether or not anyone remembers them.

```mermaid
flowchart LR
  S1["1 · Spec<br/><i>/spec</i>"] --> H1{{"human<br/>approves"}}
  H1 --> S2["2 · Plan<br/><i>/task</i>"]
  S2 --> S3["3 · Execute<br/><i>/implement</i><br/>test first, one task"]
  S3 --> S4["4 · Verify<br/><i>/qa · /pre-push</i>"]
  S4 --> S5["5 · Review<br/><i>/review</i>, then a human"]
  S5 --> PR(["Pull request → CI → merge"])

  S4 -- "gate red" --> S3
  S5 -- "blocker" --> S3
  S5 -- "spec was wrong" --> S1

  HOOKS["hooks · every edit and every shell command<br/>protect-files · block-dangerous · secret-scan<br/>auto-format · run-affected-tests · sonar-pre-push"]
  GATES["CI · every push and pull request<br/>build with warnings as errors · format<br/>unit · integration · architecture<br/>SonarQube · Gitleaks · CodeQL · dependency audit"]

  HOOKS -.-> S3
  HOOKS -.-> S4
  GATES -.-> PR

  classDef det fill:#f2f2f2,stroke:#888,color:#333
  class HOOKS,GATES det
```

The distinction matters more than the stages do. The standards, commands, and agents in `.claude/`
are **prose** — an agent reads them, usually follows them, and occasionally does not. The hooks,
the permission policy, the architecture tests, and CI are **deterministic**: they fire every time,
identically, and a `PreToolUse` hook can refuse a tool call before it happens. That is why a rule
worth enforcing lives in a hook rather than only in a document.

The full picture — every stage, every hook, which agent does what, and one small feature followed
through all five stages with real commands — is in **[docs/workflow.md](docs/workflow.md)**.

## Related repositories

The same boilerplate is published in three shapes. Pick the one that matches your project — the
single-stack repos are derived from the full-stack one, not forks that drift. See
[ADR-0006](docs/adr/0006-three-repository-split.md).

| Repository | Contents |
|---|---|
| [`aj-boilerplate-fs`](https://github.com/Emadkhanqai/aj-boilerplate-fs) | The source of truth — backend + frontend + infra |
| [`aj-boilerplate-be`](https://github.com/Emadkhanqai/aj-boilerplate-be) | This repo — backend only, promoted to the repo root |
| [`aj-boilerplate-fe`](https://github.com/Emadkhanqai/aj-boilerplate-fe) | Frontend only, promoted to the repo root |

All three share `.claude/`, the stack-neutral pages of `docs/`, `.gitignore`, and
`.editorconfig`.

## CI configuration

The workflows in `.github/workflows/` need the following repository settings. They are **not**
included and CI will not pass until you provide them. Cloud authentication uses GitHub OIDC —
there are no long-lived cloud credentials in any workflow, and none should ever be added.

**Repository variables** (*Settings → Secrets and variables → Actions → Variables*)

| Variable | Used by | Purpose |
|---|---|---|
| `CLOUD_PROVIDER` | deploy | `gcp` or `azure` — selects which IaC runs |
| `SONAR_HOST_URL` | backend CI | Your SonarQube server URL. **The quality-gate job skips itself while this is unset** — see the comment in `backend-ci.yml` and remove the guard once you have a server. |
| `SONAR_PROJECT_KEY` | backend CI | The project key on that server |
| `API_IMAGE` | deploy | Container image for the API, tag or digest |
| `NAME_PREFIX` | deploy (gcp) | Short resource-name prefix, 12 characters or fewer |
| `GCP_PROJECT_ID` · `GCP_REGION` | deploy (gcp) | Target project and region |
| `TF_STATE_BUCKET` | deploy (gcp) | Existing GCS bucket holding Terraform state |
| `AZURE_LOCATION` · `AZURE_NAME_PREFIX` | deploy (azure) | Target region and resource-name prefix |
| `AZURE_SQL_ADMIN_OBJECT_ID` · `AZURE_SQL_ADMIN_LOGIN` | deploy (azure) | Entra principal (use a group) that administers Azure SQL |

**Repository secrets**

| Secret | Used by | Purpose |
|---|---|---|
| `SONAR_TOKEN` | backend CI | SonarQube analysis token |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | deploy (gcp) | Full workload identity provider resource name |
| `GCP_SERVICE_ACCOUNT` | deploy (gcp) | Service account CI impersonates |
| `AZURE_CLIENT_ID` · `AZURE_TENANT_ID` · `AZURE_SUBSCRIPTION_ID` | deploy (azure) | Federated identity credential for OIDC |
| `GITLEAKS_LICENSE` | backend CI | Optional. Only needed for organisation-owned **private** repositories; Gitleaks is free on public ones. |

The SonarQube side targets **Community Build**, the free self-hosted edition: one project, one
branch, no pull-request decoration. The quality-gate job is deliberately guarded to `push` on
`main` for that reason — see [`.claude/standards/sonarqube.md`](.claude/standards/sonarqube.md).

Both cloud paths bootstrap themselves: the identity CI uses is created by the same IaC, so the
first deployment is run locally and the resulting values become the secrets above. Both
`infra/*/README.md` files walk through it.

**Environments** — create `dev`, `staging`, and `prod` under *Settings → Environments*. Add
required reviewers to `staging` and `prod`. Those protection rules **are** the approval gates in
`deploy.yml`; without them it promotes straight to production unreviewed.

## Contributing

Read [docs/workflow.md](docs/workflow.md) and [docs/definition-of-done.md](docs/definition-of-done.md)
before opening a pull request, and [docs/architecture.md](docs/architecture.md) before your first
change. In short: spec first, failing test first, keep the diff small, green gate, and a human
reviews every change — including the ones an agent wrote.

## Licence

**No licence is set yet.** Until a `LICENSE` file is added, default copyright applies and no
reuse rights are granted. The repository owner should choose a licence and commit the
corresponding `LICENSE` file, carrying the appropriate copyright line.
