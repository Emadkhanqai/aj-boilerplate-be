# CLAUDE.md — Agentic Backend Boilerplate

Project context for Claude Code. Read this first, then [`README.be.md`](README.be.md) (the
engineering reference: migrations, idempotency, feature flags, file storage) and whichever files in
`.claude/standards/` cover what you are about to touch. For anything about sign-in, start at
[`docs/auth/README.md`](docs/auth/README.md).

> **This is `aj-boilerplate-be`, the backend half of a three-repository set** (ADR-0006), derived
> from [`aj-boilerplate-fs`](https://github.com/Emadkhanqai/aj-boilerplate-fs). The Angular app and
> the **nginx gateway** live in [`aj-boilerplate-fe`](https://github.com/Emadkhanqai/aj-boilerplate-fe).
> Change the source in `fs` and re-derive; do not hand-edit shared files here.

> **HARD RULE — secrets.** Never put a secret, password, API key, token, connection string,
> certificate, or any credential in this file, in a prompt, in a commit message, in an ADR, in
> a spec, or in any other context file. Ever. Configuration values that *look* like secrets go
> in the secrets provider (Secret Manager / Key Vault) and are referenced by name only. If you
> find one committed, treat it as an incident: rotate it, then remove it.

---

## What this is

A runnable starting point for a new backend service: a .NET layered Clean Architecture API, an
OpenAPI contract consumers generate clients from, an Auth service (Entra ID sign-in, sessions,
Keycloak token exchange, user admin through Keycloak's Admin REST API, a session-ended outbox to the API)
that keeps tokens out of the browser, a quality gate, IaC for two cloud
providers, and a committed `.claude/` harness. The gateway nginx that sits in front of both is in the
frontend repository.

It contains **no business domain**. Two modules ship, and they are not the same kind of thing:

- **`Item`** — the sample slice. It exists to prove the whole path end to end and is designed
  to be deleted on day one.
- **`Features`** — the "What's new" feature spotlight: shows each user a newly shipped feature
  once, the first time they hit a URL prefix bound to it, with acknowledgement stored per user
  server-side. Domain-free, and meant to be kept. Adding an announcement is an INSERT-only
  migration — no service change, no frontend change. Full reference: [docs/whats-new.md](docs/whats-new.md).

## Making it yours (do this first)

1. **.NET identity** — rename `AjBoilerplate` everywhere: the six project folders under `src/`
   (including `AjBoilerplate.Auth`), the four under `tests/`, `AjBoilerplate.slnx`, every
   `namespace`/`using`, `RootNamespace`/`AssemblyName` in the `.csproj` files, and the
   `sonar.projectKey`.
2. **Sample entity** — delete or rename `Item` end to end: domain entity, handlers, DTOs, EF
   configuration, the `Items` table in the `InitialCreate` migration, controller and OpenAPI paths.
   Delete its tests with it; keep the architecture tests. Leave the `Features` module and
   `AddFeatureAnnouncements` alone — they carry no domain and are not part of the sample slice.
3. **Auth** — pick `AUTH_PROVIDER`: `embedded` (you deploy the Auth service; every project outside
   the Al Jazeera USP platform) or `usp` (inside USP; you do not deploy it). Replace the seats and
   permissions with your model (`ApplicationRoles`, `Permissions`, the local realm, and the
   frontend's role map in `aj-boilerplate-fe`). See [docs/auth/setup.md](docs/auth/setup.md).
4. **Docs** — replace `README.md` and continue the ADR series at `0013` (keep ours as
   `0001`–`0012` history or delete them).
5. **Infra** — pick your provider, set project/subscription variables, configure a remote
   state backend, and delete the tree you are not using. `infra/` provisions the API, database,
   Redis and secrets; deploying the web container, Auth service and Keycloak is yours.

## Stack

| Layer | Technology |
|---|---|
| API | .NET 10, ASP.NET Core, EF Core 10 |
| Database | Microsoft SQL Server (migration-based, never `EnsureCreated`) |
| Cache | Redis (Memorystore on GCP, Cache for Redis on Azure — protocol-identical) |
| AuthN | Microsoft Entra ID, through the Auth service (`AjBoilerplate.Auth`): server-side Authorization Code + PKCE. The same on both clouds. |
| AuthZ | Keycloak — roles and permissions live there; the API validates the JWT it mints for this app's client |
| Gateway | **Not in this repository.** nginx in the web container of `aj-boilerplate-fe` (`AUTH_PROVIDER=embedded`), or USP's shared gateway (`usp`) |
| Tests | xUnit + Testcontainers-based integration suite |
| Quality | SonarQube Community Build (free, self-hosted), Gitleaks, CodeQL, `dotnet format`, Prettier-free: `.editorconfig` |

## Architecture

```
.
├── AjBoilerplate.slnx
├── src/
│   ├── AjBoilerplate.Domain/          entities, domain exceptions — zero dependencies
│   ├── AjBoilerplate.Application/     handlers, ports (abstractions), validation
│   ├── AjBoilerplate.Contracts/       DTOs, ApiResponse<T>, PagedResponse<T>
│   ├── AjBoilerplate.Infrastructure/  EF Core, repositories, cache, secrets, messaging
│   ├── AjBoilerplate.Api/             controllers, middleware, DI, JWT validation (Identity/), observability
│   └── AjBoilerplate.Auth/            SEPARATE host: sign-in, sessions, /auth/validate. References none of
│                                      the five above, and none reference it
└── tests/
    ├── AjBoilerplate.UnitTests/
    ├── AjBoilerplate.IntegrationTests/
    ├── AjBoilerplate.Auth.Tests/          the Auth service (fakes, no Docker needed)
    └── AjBoilerplate.ArchitectureTests/   ← enforces the rule below and the Auth service's independence
Both shipped modules are foldered the same way at every layer — `Items/` next to `Features/` — so a
third one has an obvious shape to copy.

**The layer dependency rule.** Dependencies point inward, one direction only:

```
Api → Infrastructure → Application → Domain
Api → Contracts
Auth → (none of the above)
```

- `Domain` references nothing. No EF Core, no ASP.NET, no third-party framework.
- `Application` may reference `Domain` — and **not** `Contracts`. The Application layer owns its
  own models; `Api` maps them to the wire DTOs. Otherwise a breaking API change would force a
  change to the use cases themselves. It declares ports (interfaces); it never references
  `Infrastructure` or `Api`.
- `Contracts` references nothing else. It is the wire format, shared with API consumers.
- `Infrastructure` implements those ports. It is the only project that knows about EF Core,
  Redis, HTTP clients, or a cloud SDK.
- `Api` composes. Controllers are thin: validate, delegate to a handler, shape a response.
- Business rules live in `Domain` and `Application`. Never in a controller, never in a
  repository, never in a client.

`AjBoilerplate.ArchitectureTests` fails the build if any of this is violated, including the Auth
service referencing the API-side projects or being referenced by them. Do not weaken it.

## Commands

Run from the repository root unless noted.

```bash
dotnet restore  AjBoilerplate.slnx
dotnet build    AjBoilerplate.slnx -warnaserror
dotnet format   AjBoilerplate.slnx --verify-no-changes   # CI-equivalent check
dotnet test     tests/AjBoilerplate.UnitTests
dotnet test     tests/AjBoilerplate.ArchitectureTests
dotnet test     tests/AjBoilerplate.IntegrationTests     # needs Docker (Testcontainers starts SQL Server)
dotnet test     tests/AjBoilerplate.Auth.Tests           # the Auth service; no Docker needed
dotnet run    --project src/AjBoilerplate.Api            # http://localhost:5080
dotnet run    --project src/AjBoilerplate.Auth           # http://localhost:5090 (launchSettings; secrets via env)
dotnet list     AjBoilerplate.slnx package --vulnerable --include-transitive
```

The API answers `401` to protected endpoints until `Keycloak__ValidIssuer`, `Keycloak__JwksUri` and
`Keycloak__ClientId` point at a Keycloak. The sign-in flow end to end (Keycloak, Auth service,
gateway, seeded users) runs from the root `docker-compose.yml` of `aj-boilerplate-fs`, not from here.
The Auth service has its own migrations (schema `auth`, `AUTH_DB_CONNECTION` for the design-time
factory); see `src/AjBoilerplate.Auth/README.md`.

### Migrations

```bash
dotnet ef migrations add <Name> \
  --project        src/AjBoilerplate.Infrastructure \
  --startup-project src/AjBoilerplate.Api
dotnet ef database update \
  --project        src/AjBoilerplate.Infrastructure \
  --startup-project src/AjBoilerplate.Api
```

Prefer the `/new-migration` command — it reviews the generated SQL before anything is applied.

### Whole-repo gate

`/qa` runs the full local gate (build, format, tests, SonarQube). `/pre-push` runs it and
reports readiness. Neither pushes.

## The `CLOUD_PROVIDER` switch

`CLOUD_PROVIDER` (env var, bound to `Cloud:Provider` in configuration) accepts `gcp` or
`azure`. Set it before running the API. An unrecognised value fails at start-up.

| Concern | `gcp` | `azure` |
|---|---|---|
| Secrets | Secret Manager | Key Vault + Managed Identity |
| Cache | Memorystore | Cache for Redis |
| IaC | `infra/gcp/` (Terraform) | `infra/azure/` (Bicep) |

Only **secrets** branch in code, behind `ISecretsProvider`. **Identity does not branch:** users
sign in against Entra ID on either cloud, and the API validates only Keycloak JWTs
([ADR-0012](docs/adr/0012-gateway-session-auth.md); ADR-0002's identity half is superseded).
If you add a third provider, add it in that one place, not scattered through feature code.

## Authentication — read before touching anything auth-shaped

The model, in one line: **browser → nginx (cookie) → Auth service → Keycloak JWT → API.** Full
picture and diagrams: [docs/auth/README.md](docs/auth/README.md). The wire contract all four parts
share (nginx, Auth service, API, SPA) is [docs/auth/gateway-contract.md](docs/auth/gateway-contract.md);
**change one part and you change all four in the same PR.** In this repository you own two of the
four, the API and the Auth service; nginx and the SPA are in `aj-boilerplate-fe`, so a contract change
here is a coordinated change there. Work on the contract itself in `aj-boilerplate-fs`.

Two switches: `AUTH_PROVIDER=embedded|usp` (who runs the gateway) and `AUTH_MODE=gateway|dev` (`dev`
is a role picker for UI work; it cannot call a gateway-mode API).

**Auth non-negotiables** (these are the contract's invariants, restated):

1. **The browser holds no token.** No `localStorage`/`sessionStorage` token, no `Authorization`
   header from the SPA, no browser OIDC client. Only the Auth service sets or reads the session cookie.
2. **Only three things end a session:** an Entra `invalid_grant` (or its siblings), a definitive
   Keycloak "no role", and the 12 h absolute lifetime. **An infrastructure fault never signs anyone
   out**: it serves the current JWT while it has more than 5 s left, otherwise answers 503.
3. **Only a `401` may become a login redirect.** A 403 or 503 never does. `X-Usp-Access: revoked` is
   sent only on a definitive verdict.
4. **The API trusts Keycloak JWTs only**, checks `azp` equals `Keycloak:ClientId` (fails closed),
   uses `ClockSkew = 0`, and refuses a plaintext JWKS outside Development unless
   `Keycloak:AllowInsecureJwksTransport` is set on purpose. Every endpoint declares a policy; the
   fallback policy denies.
5. **Seat OR permission.** A policy is satisfied by the role hierarchy or by a `resource:action`
   permission. A role key the app does not know is dropped, never granted. A role name may appear in
   exactly the places `docs/auth/setup.md` section 4 lists.
6. **Never log** `Authorization`, `X-Auth-Jwt`, cookies, tokens, client secrets or claim payloads.
   Log reason slugs, status codes and route templates. Do not log a raw URL: it can carry an OAuth
   `code`.
7. **Secrets arrive as environment variables** from the secrets provider. The API loads the secret
   store itself; the Auth service does not, so its platform injects them. Never in a file.
8. **The Auth service stays independent** of `Api`, `Application`, `Domain`, `Infrastructure` and
   `Contracts`.

## Conventions

**API envelope.** Every response, success or failure, is `ApiResponse<T>`:
`{ success, data, message, errors, statusCode, code, timestamp, traceId }`. Collections use
`PagedResponse<T>`. A result filter applies the envelope; controllers return the payload.

**Status codes.** `200` read, `201` created (with `Location`), `204` delete, `400` validation,
`401` unauthenticated, `403` unauthorised, `404` missing, `409` conflict or concurrency,
`422` domain-rule violation, `429` rate-limited, `500` unhandled. Never `200` with
`success: false`.

**Error codes.** `code` is a stable `SCREAMING_SNAKE_CASE` string clients may branch on
(`ITEM_NOT_FOUND`, `VALIDATION_FAILED`, `CONCURRENCY_CONFLICT`). `message` is human-readable
and may change; `code` may not. Exceptions map to codes in the `IExceptionHandler` chain —
never construct an error response by hand in a controller.

**Routing.** `/api/v{version}/{resource}` — plural, kebab-case, versioned from day one.

**Migrations.** Every schema change is an EF Core migration, reviewed as SQL before it is
applied. Never edit a migration that has been applied anywhere but your own machine; add a new
one. Never `EnsureCreated`, never out-of-band DDL.

**The API contract.** `docs/api/openapi.json` is generated from the compiled API by
`./scripts/generate-openapi.sh` and committed; CI fails when it drifts. Clients generate their types
from it. Never hand-edit it. A missing `[ProducesResponseType]` is a bug, and a contract change is
reviewed as a diff of that file.

**Identity tests.** Test the user-visible outcome of an auth change (the status, the page, the
capability flags), not only that a request was sent. `AjBoilerplate.Auth.Tests` covers the Auth
service; the API's integration tests act as a role through a test authentication handler.

**Tests.** Failing test first, then the code. Unit tests for domain and handler logic,
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
5. **Respect the layer dependency rule.** If a change needs to break it, the design is wrong.
6. **Migration-based schema changes only.**
7. **The OpenAPI document is the contract**: commit it with the change, never hand-edit it.
8. **One task per session, fresh context per task.** No unattended multi-hour runs.
9. **Human review is mandatory** and is never waived because an agent wrote the code. The
   developer who prompted it owns it. Keep PRs to roughly 400 changed lines.
10. **Update the docs with the change.** If a convention changed, `CLAUDE.md` changes in the
    same PR. If a decision was made, an ADR lands with it. If a contract changed, the OpenAPI
    document changes with it, and consumers regenerate their types.
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
| Five-stage workflow and guardrails | [docs/workflow.md](docs/workflow.md) |
| Definition of Done | [docs/definition-of-done.md](docs/definition-of-done.md) |
| Day-1 checklist | [docs/onboarding.md](docs/onboarding.md) |
| Spec template | [docs/specs/TEMPLATE.md](docs/specs/TEMPLATE.md) |
| The "What's new" module, end to end | [docs/whats-new.md](docs/whats-new.md) |
| Architecture decisions | [docs/adr/](docs/adr/) |
| API contract workflow | [docs/api/README.md](docs/api/README.md) |
| Session handoffs | [docs/handoff/](docs/handoff/) |
| Sign-in: overview, setup, troubleshooting, contract | [docs/auth/](docs/auth/README.md) |
| Infrastructure | [infra/gcp/README.md](infra/gcp/README.md) · [infra/azure/README.md](infra/azure/README.md) |
