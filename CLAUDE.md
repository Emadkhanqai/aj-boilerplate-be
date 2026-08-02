# CLAUDE.md — Agentic Backend Boilerplate

Project context for Claude Code. Read this first, then whichever files in `.claude/standards/`
cover what you are about to touch.

> **HARD RULE — secrets.** Never put a secret, password, API key, token, connection string,
> certificate, or any credential in this file, in a prompt, in a commit message, in an ADR, in
> a spec, or in any other context file. Ever. Configuration values that *look* like secrets go
> in the secrets provider (Secret Manager / Key Vault) and are referenced by name only. If you
> find one committed, treat it as an incident: rotate it, then remove it.

---

## What this is

A runnable starting point for a new backend service: a .NET 10 layered Clean Architecture API,
an OpenAPI contract consumers generate their clients from, a quality gate, IaC for two cloud
providers, and a committed `.claude/` harness.

It contains **no business domain**. The single sample entity, `Item`, exists to prove the whole
path end to end and is designed to be deleted on day one.

The frontend counterpart is [`aj-boilerplate-fe`](https://github.com/Emadkhanqai/aj-boilerplate-fe);
both stacks together are [`aj-boilerplate-fs`](https://github.com/Emadkhanqai/aj-boilerplate-fs).
See [ADR-0006](docs/adr/0006-three-repository-split.md).

## Making it yours (do this first)

1. **Namespaces and assemblies** — rename `AjBoilerplate` everywhere: the five project folders
   under `src/`, the three under `tests/`, every `namespace` and `using`, and `RootNamespace` /
   `AssemblyName` in the `.csproj` files if you set them explicitly.
2. **Solution name** — rename `AjBoilerplate.slnx` at the repository root and update the eight
   `<Project Path=…>` entries inside it to the new folder names.
3. **Sample entity** — delete or rename `Item` end to end: `src/AjBoilerplate.Domain/Items/`,
   `src/AjBoilerplate.Application/Items/`, `src/AjBoilerplate.Contracts/Items/`,
   `src/AjBoilerplate.Infrastructure/Persistence/ItemRepository.cs` and
   `src/AjBoilerplate.Infrastructure/Persistence/Configurations/ItemConfiguration.cs`,
   `src/AjBoilerplate.Api/Controllers/ItemsController.cs`, the `InitialCreate` migration, and
   the two `DependencyInjection.cs` registrations. Delete its tests with it; **keep the architecture
   tests** — `ControllerConventionTests` asserts at least one controller exists, so it will tell
   you if you deleted the last one.
4. **SonarQube project key** — set `SONAR_PROJECT_KEY`, or add `sonar-project.properties`.
5. **Docs** — replace `README.md` and start your own ADR series (keep ours as `0001`–`0006`
   history, or delete them and start at `0001`).
6. **Infra** — pick your provider, set the project/subscription variables, configure a remote
   state backend, and delete the tree you are not using.

## Stack

| Layer | Technology |
|---|---|
| API | .NET 10, ASP.NET Core, EF Core 10 |
| Database | Microsoft SQL Server (migration-based, never `EnsureCreated`) |
| Cache | Redis (Memorystore on GCP, Cache for Redis on Azure — protocol-identical) |
| AuthN | Google Cloud Identity (`gcp`) or Microsoft Entra ID (`azure`) |
| AuthZ | Keycloak — provider-independent, roles and policies live there |
| Tests | xUnit; integration suite on a Testcontainers SQL Server |
| Quality | SonarQube Community Build (free, self-hosted), Gitleaks, CodeQL, `dotnet format` |

## Architecture

```
.
├── AjBoilerplate.slnx
├── src/
│   ├── AjBoilerplate.Domain/          entities, domain exceptions — zero dependencies
│   ├── AjBoilerplate.Application/     use cases, ports (abstractions), validation
│   ├── AjBoilerplate.Contracts/       DTOs, ApiResponse<T>, PagedResponse<T>
│   ├── AjBoilerplate.Infrastructure/  EF Core, repositories, cache, secrets, messaging
│   └── AjBoilerplate.Api/             controllers, middleware, DI, auth, observability
└── tests/
    ├── AjBoilerplate.UnitTests/
    ├── AjBoilerplate.IntegrationTests/
    └── AjBoilerplate.ArchitectureTests/   ← enforces the rule below, in CI
```

**The layer dependency rule.** Dependencies point inward, one direction only:

```
Api → Infrastructure → Application → Domain
Api → Contracts
```

- `Domain` references nothing. No EF Core, no ASP.NET, no third-party framework.
- `Application` may reference `Domain` — and **must NOT reference `Contracts`. The architecture
  test `Application_does_not_depend_on_the_wire_contracts` enforces this.** The Application
  layer owns its own models; `Api` maps them to the wire DTOs. Otherwise a breaking API change
  would force a change to the use cases themselves. It declares ports (interfaces); it never
  references `Infrastructure` or `Api`.
- `Contracts` references nothing else. It is the wire format, shared with API consumers.
- `Infrastructure` implements those ports. It is the only project that knows about EF Core,
  Redis, HTTP clients, or a cloud SDK.
- **`Api` must not reference `Domain`** — also asserted, by
  `Api_does_not_reference_domain_directly`. It works through Application models and Contracts
  DTOs, which is what keeps entities out of request and response bodies.
- Business rules live in `Domain` and `Application`. Never in a controller, never in a
  repository.

`AjBoilerplate.ArchitectureTests` inspects **compiled assembly references**, so a violation
cannot be hidden behind a fully qualified name. If it fails, the fix is to move the type that
leaked — never to relax the test.

Full tour, with the reasoning for each boundary: [docs/architecture.md](docs/architecture.md).

## Commands

Run from the repository root.

```bash
dotnet restore
dotnet build  -warnaserror                            # warnings are errors
dotnet format --verify-no-changes                     # CI-equivalent style gate
dotnet test                                           # all three suites; needs Docker
dotnet test tests/AjBoilerplate.UnitTests             # fast, no dependencies
dotnet test tests/AjBoilerplate.ArchitectureTests     # fast, no dependencies
dotnet test tests/AjBoilerplate.IntegrationTests      # Testcontainers SQL Server
dotnet run --project src/AjBoilerplate.Api            # http://localhost:5080, Swagger at /swagger
dotnet list package --vulnerable --include-transitive
```

`dotnet test` requires Docker. `SqlServerFixture` starts one throwaway SQL Server container,
applies the migrations to it, and tears it down. There is no in-memory provider anywhere, on
purpose: a suite that cannot see a unique index, a `rowversion`, or the SQL EF Core actually
emits is not testing the integration.

Local dependencies via Docker Compose:

```bash
cp .env.example .env          # then fill in MSSQL_SA_PASSWORD
docker compose up -d db redis
docker compose --profile tools run --rm migrate     # apply migrations
```

### Migrations

```bash
dotnet ef migrations add <IntentRevealingName> \
  --project        src/AjBoilerplate.Infrastructure \
  --startup-project src/AjBoilerplate.Api \
  --output-dir     Persistence/Migrations
dotnet ef database update \
  --project        src/AjBoilerplate.Infrastructure \
  --startup-project src/AjBoilerplate.Api
```

Prefer the `/new-migration` command — it reviews the generated SQL before anything is applied.
The design-time factory reads `APP_DB_CONNECTION`, **not** `ConnectionStrings__Default` (which
only the running app uses).

### The gate

`/qa` runs the full local gate (build, format, tests, dependency audit, secret scan,
SonarQube). `/pre-push` runs it and reports readiness. Neither pushes.

## The `CLOUD_PROVIDER` switch

`CLOUD_PROVIDER` (env var, bound to `Cloud:Provider` in configuration) accepts `gcp` or
`azure`. `CloudOptions.Resolve()` throws on anything else — a typo fails loudly at startup
rather than silently reading secrets from the wrong cloud.

| Concern | `gcp` | `azure` |
|---|---|---|
| Secrets | Secret Manager | Key Vault + Managed Identity |
| Identity (authN) | Google Cloud Identity | Microsoft Entra ID |
| Authorization | Keycloak | Keycloak |
| Cache | Memorystore | Cache for Redis |
| Database | SQL Server | SQL Server |
| IaC | `infra/gcp/` (Terraform) | `infra/azure/` (Bicep) |

Only secrets and identity branch in code, behind `ISecretsProvider` and `AuthenticationSetup`.
Everything else is provider-agnostic. If you add a third provider, add it in those two places —
not scattered through feature code.

## Conventions

**API envelope.** Every response, success or failure, is `ApiResponse<T>`:
`{ success, data, message, errors, statusCode, code, timestamp, traceId }`. Collections use
`PagedResponse<T>`. `EnvelopeResultFilter` applies the envelope to controller results and
`EnvelopeStatusCodePages` applies it to replies that never reach a controller; controllers
return the payload.

**Error codes.** `code` is a stable `SCREAMING_SNAKE_CASE` slug a client may branch on;
`message` is human-readable and may change. The complete set is
`AjBoilerplate.Contracts.Common.EnvelopeCodes`:

`VALIDATION_ERROR` · `CONFLICT` · `INTERNAL_ERROR` · `UNAUTHORIZED` · `FORBIDDEN` ·
`NOT_FOUND` · `METHOD_NOT_ALLOWED` · `UNSUPPORTED_MEDIA_TYPE` · `REQUEST_FAILED` ·
`SERVICE_UNAVAILABLE` · `TOO_MANY_REQUESTS`

Add a constant there before you use a new one. Never construct an error response by hand in a
controller — exceptions map to codes in the `IExceptionHandler` chain
(Validation → Conflict → Forbidden → Unhandled).

**Status codes.** What this API actually produces, and the `code` that comes with it:

| Status | `code` | When |
|---|---|---|
| `200` | — | read, or an update that returns a body |
| `201` | — | created, with a `Location` header |
| `204` | — | delete |
| `400` | `VALIDATION_ERROR` | FluentValidation failure; `errors[]` lists the field failures |
| `401` | `UNAUTHORIZED` | no or invalid token |
| `403` | `FORBIDDEN` | policy failure, or `ForbiddenException` after the record is loaded |
| `404` | `NOT_FOUND` | missing, or not visible to this caller |
| `405` | `METHOD_NOT_ALLOWED` | wrong verb on a matched route |
| `409` | `CONFLICT` | `ConflictException`, including optimistic-concurrency failure |
| `415` | `UNSUPPORTED_MEDIA_TYPE` | wrong content type |
| `429` | `TOO_MANY_REQUESTS` | rate limited |
| `500` | `INTERNAL_ERROR` | unhandled — never leaks a stack trace or internal message |
| `503` | `SERVICE_UNAVAILABLE` | a dependency is down |

Never `200` with `success: false`.

**Routing.** `/api/v{version}/{resource}` — plural, kebab-case, versioned from day one.
`ControllerConventionTests` fails the build if a route is unversioned, or if a controller
carries neither `[Authorize]` nor an explicit `[AllowAnonymous]`.

**Migrations live in `src/AjBoilerplate.Infrastructure/Persistence/Migrations/`.** Every schema
change is an EF Core migration, reviewed as SQL before it is applied. Never edit a migration
that has been applied anywhere but your own machine; add a new one. Never `EnsureCreated`,
never out-of-band DDL, never auto-apply on startup.

**Time is `IClock`, never `DateTime.UtcNow`.** `SystemClock` is the only place in the codebase
that reads the machine's wall clock. `FixedClock` in the unit tests is what makes timestamp
assertions exact instead of approximate — and an approximate assertion about time is a flaky
test waiting for a slow CI runner.

**OpenAPI is the published contract.** Consumers generate their clients from it, so a missing
`[ProducesResponseType]` is a bug, not a documentation gap. See [docs/api/](docs/api/).

**Tests.** Failing test first, then the code. Unit tests for domain and use-case logic,
integration tests for anything crossing a boundary.

## Non-negotiable rules

1. **No secrets in context.** See the rule at the top of this file.
2. **Never `git push` without explicit human approval**, on any branch, to any remote, every
   time. Committing is fine; pushing is a human decision.
3. **The quality gate runs before any push is proposed.** Zero new Blocker, Critical, or Major
   SonarQube findings; ≥80% coverage on new code. Minor and Info may be triaged. The gate
   targets **SonarQube Community Build** (free, self-hosted): one project, one branch, no
   branch analysis and no pull-request decoration — never pass `sonar.branch.name`,
   `sonar.pullrequest.*`, or a `branch`/`pullRequest` MCP argument. See
   `.claude/standards/sonarqube.md`.
4. **Build with warnings as errors.** A warning is a failure.
5. **Respect the layer dependency rule**, including `Application ↛ Contracts` and
   `Api ↛ Domain`. If a change needs to break it, the design is wrong.
6. **Migration-based schema changes only.**
7. **No EF Core entity on the wire**, in either direction. DTOs at the boundary, always.
8. **One task per session, fresh context per task.** No unattended multi-hour runs.
9. **Human review is mandatory** and is never waived because an agent wrote the code. The
   developer who prompted it owns it. Keep PRs to roughly 400 changed lines.
10. **Update the docs with the change.** If a convention changed, `CLAUDE.md` changes in the
    same PR. If a decision was made, an ADR lands with it. If a contract changed, the OpenAPI
    document and `docs/api/` change with it.
11. **Classify the task and state the model tier before the first tool call.** Frontier tier
    for architecture, security review, complex debugging, high-risk refactors, and the final
    pre-push review; workhorse tier for everything else. Say the recommendation out loud in
    the first reply — and if this session is on a costlier model than the work needs, **stop
    and say so** rather than spending it. The `model-routing` hook injects this on every
    prompt; the policy is `.claude/model-routing.md`.

## Where to look next

| Topic | Path |
|---|---|
| Deeper standards (one file per topic) | `.claude/standards/` |
| Layering rules in detail | `.claude/standards/clean-architecture.md` |
| Slash commands | `.claude/commands/` |
| Hooks and their triggers | `.claude/hooks/` · `.claude/README.md` |
| Model routing (enforced every prompt) | `.claude/model-routing.md` |
| The SonarQube gate, Community Build setup | `.claude/standards/sonarqube.md` |
| Every layer, and why each boundary exists | [docs/architecture.md](docs/architecture.md) |
| Five-stage workflow and guardrails | [docs/workflow.md](docs/workflow.md) |
| Definition of Done | [docs/definition-of-done.md](docs/definition-of-done.md) |
| Day-1 checklist | [docs/onboarding.md](docs/onboarding.md) |
| Spec template | [docs/specs/TEMPLATE.md](docs/specs/TEMPLATE.md) |
| Architecture decisions | [docs/adr/](docs/adr/) |
| API contract workflow | [docs/api/README.md](docs/api/README.md) |
| Session handoffs | [docs/handoff/](docs/handoff/) |
| Infrastructure | [infra/gcp/README.md](infra/gcp/README.md) · [infra/azure/README.md](infra/azure/README.md) |
