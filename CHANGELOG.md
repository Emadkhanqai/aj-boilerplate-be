# Changelog

Notable changes to this boilerplate. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/spec/v2.0.0.html) as adapted below.

**This file is written for the person upgrading, not for the person who made the change.**
A changelog entry that reads like a commit message has failed at its only job. Every entry
should answer: what changed, whether I have to do anything about it, and what happens if I
ignore it. If an entry needs a migration step, the step is in the entry — not in a linked
issue, not in a pull request comment.

For how to actually pull these changes into a project that cloned this boilerplate, see
[docs/upgrading.md](docs/upgrading.md).

---

## Versioning, for a boilerplate

Semantic Versioning is defined in terms of an API. A boilerplate has no API — it has a
shape that consumers have already copied. So the meanings are pinned explicitly:

| Bump | Means | Examples |
|---|---|---|
| **MAJOR** | A consumer who cloned an earlier version has to change their own code to take this. | A layer moves or is renamed · the `ApiResponse<T>` envelope changes shape · a hook's contract or opt-out variable changes · a configuration key is renamed or removed · a `.claude/` standard reverses a rule people wrote code against. |
| **MINOR** | Something new that an existing consumer can take or leave. | A new hook · a new workflow · a new ADR · a new optional configuration key with a safe default · a new document. |
| **PATCH** | A fix or a clarification with no shape change. | A hook bug · a broken CI step · a wrong path in a document · a dependency bump with no behaviour change. |

Two consequences worth stating, because both surprise people:

- **A documentation change can be MAJOR.** If a standard in `.claude/standards/` reverses a
  rule, consumers who followed it are now non-compliant. That is a breaking change even
  though no code moved.
- **A dependency bump is usually PATCH here, even when the dependency's own bump is
  MAJOR.** What matters is the effect on a consumer's tree, and a consumer who cloned this
  boilerplate owns their own `.csproj` files.

## Releasing

1. Everything intended for the release is merged to `main` and the gate is green.
2. Move the `[Unreleased]` entries into a new version section with today's date. Rewrite
   them for the upgrader if they still read like commit subjects.
3. Choose the bump using the table above. The Conventional Commits types in the range are
   an input, not the answer: `feat` suggests minor and `!`/`BREAKING CHANGE` suggests
   major, but a `docs:` commit that reverses a standard is still major.
4. Tag it. **Annotated tags only** — a lightweight tag carries no author, date, or message,
   and the tag is the thing consumers pin to.

   ```bash
   git tag -a v1.2.0 -m "v1.2.0 — container image scanning, SBOM, Dependabot"
   git push origin v1.2.0
   ```

5. **The three repositories share a version number.** They are one artefact published in
   three shapes (ADR-0006), and a consumer comparing `aj-boilerplate-be` v1.2.0 against
   `aj-boilerplate-fs` v1.2.0 must be comparing the same content. This repository is
   regenerated from `aj-boilerplate-fs` by the derivation script recorded in ADR-0011 and
   is tagged with the same version in the same release, which is why a release is cut
   there and never here.
6. Create the GitHub release from the tag, with this file's section as the body.

Tag format is `vMAJOR.MINOR.PATCH`. Pre-releases, if ever needed, are `v1.2.0-rc.1` and are
never derived to the single-stack repositories — those track releases only.

---

## [Unreleased]

Nothing yet.

---

## [1.0.0] — 2026-11-24

**A major release, because sign-in was replaced and configuration keys changed.** Derived from
`aj-boilerplate-fs` v1.0.0 (`1e96e51`). If you cloned 0.1.x of this repository and kept its
authentication, you cannot take this release by merging; read
[Upgrade steps from 0.1.x](#upgrade-steps-from-01x) first.

The short version: **the API only ever sees Keycloak JWTs, and sign-in is a server-side flow run by a new
Auth service.** The reasoning is in [ADR-0012](docs/adr/0012-gateway-session-auth.md) and the model is
in [docs/auth/](docs/auth/README.md).

### Breaking changes

- **API: direct cloud tokens are no longer accepted.** The `Authentication:GoogleCloudIdentity:*` and
  `Authentication:EntraId:*` schemes are removed, together with `Keycloak:Realm` and
  `Keycloak:ClientSecret`, which nothing read. The API validates only Keycloak JWTs whose `azp` equals
  `Keycloak:ClientId`. *If you ignore this:* a bearer token from Google or Entra is rejected with 401,
  and the removed keys are silently ignored.
- **API: new start-up refusals outside Development.** A plaintext `Keycloak:JwksUri` stops the API
  unless you set the new `Keycloak:AllowInsecureJwksTransport=true`; an empty `JwksUri` stops it unless
  `Keycloak:AllowDiscoveryFallback=true`; an `Auth:Mode` other than `gateway` stops it.
- **API: deny by default.** A fallback policy requires an authenticated caller on every endpoint that
  declares nothing. The three health endpoints opt out with `AllowAnonymous`. An endpoint you added
  outside a controller answers 401 until you decide its policy.
- **API: policies are "a seat OR a permission".** `ReadAccess`, `WriteAccess` and `AdminAccess` are
  satisfied by the role hierarchy or by `items:read`, `items:write` or `admin:manage`. A role key the
  application does not recognise is dropped and logged once, never granted.
- **API: the rate limiter runs after authentication** and partitions on the user's subject. Before,
  every caller behind a gateway shared one bucket keyed by the proxy's address.
- **`CLOUD_PROVIDER` no longer selects an identity issuer.** It selects the secrets provider and the
  `infra/` tree. [ADR-0002](docs/adr/0002-dual-cloud-provider-behind-one-switch.md)'s identity half is
  superseded by ADR-0012 (a note sits at its top; its body is unchanged).
- **New database objects.** The app database gets an `AuthenticationEvents` table (migration
  `AddAuthenticationEvents`, then `AddSessionEndAndDisplayName`, which adds the nullable end columns), and the
  Auth service needs its own database with schema `auth`, applied by its own migrations (the last,
  `AddHandoffAndSessionEndedOutbox`, is additive). None runs at start-up.
- **`.env.example` was rewritten.** The old `OIDC_*` and `Authentication__*` entries are gone; the file
  now carries the API's Keycloak keys and the Auth service's keys, and `Keycloak__*` is no longer
  missing. Compare it with your `.env`.
- **`docs/` is the full-stack documentation, identical to `aj-boilerplate-fs`.** The backend-only
  rewrites of `architecture.md`, `workflow.md`, `onboarding.md` and the other shared documents from 0.1.x
  are gone; links into frontend files point at `aj-boilerplate-fs` on GitHub. The `.claude/` harness is
  the same, minus the frontend agents, standards, templates and commands. ADR numbers are unchanged:
  this repository already used the full-stack numbering, and ADR-0012 is new.

### Added

- **`AjBoilerplate.Auth`**, a separate ASP.NET Core host: Authorization Code + PKCE sign-in against Entra
  ID, RFC 8693 token exchange at Keycloak, encrypted sessions in SQL Server with Redis in front, 8 h idle
  and 12 h absolute lifetimes, single-flight refresh, `/auth/validate`, a roles-version endpoint, a
  revoked marker, and an admin force-refresh endpoint. `AjBoilerplate.Auth.Tests` and an architecture
  test that keeps it independent of the layered API.
- **User administration and sign-in history.** The Auth service gains `/internal/admin/*` (users, groups, members,
  per-user seats, disable and enable, refresh, over a Keycloak Admin REST client restricted to
  `Auth:Admin:ManagedGroupPaths`, with a deny-by-default e-mail domain allow-list), `/auth/no-access`,
  `POST /auth/events` (off until configured), `/auth/handoff` and `/auth/handoff/redeem` (needs `Auth:Portals`) and a
  transactional outbox that reports each session's end to the API
  ([contract](docs/auth/session-ended-contract.md)). The API gains `POST /api/integration/auth-backend/session-ended`,
  `GET /api/v1/admin/authentication-history`, `GET /api/v1/me/portals` and an `/api/v1/admin/*` proxy over the Auth
  service's admin routes. New keys: `Auth:Admin:*`, `Auth:Portal:SessionEvents:*`, `Auth:Portal:DisplayName`,
  `Auth:Portals:N:*`, `Auth:Events:*`, `Auth:SessionEndDelivery:*`, `Auth:ForwardedHostHeader`, `AuthService:*`,
  `Integration:AuthBackend:*` and `Portals:N:*`; all optional, and each surface is off or 404 until configured.
  `infra/gcp` has optional variables for the Auth service's admin client and session events. *If you ignore this:* nothing
  changes for you except two new migrations to apply.
- **`GET /api/v1/me`**, returning roles, capability flags and `permissions`.
- **API identity layer**: JWKS pre-warm, seats and permissions (`ApplicationRoles`, `Permissions`,
  `RoleCapabilities`, `SeatOrPermission`), and `SignInRecordingMiddleware`, which records one row per
  gateway session.
- **A bundled Keycloak realm** (`infra/keycloak/realm-aj-boilerplate.json`, dev-only secrets) with seeded
  users `admin`, `editor`, `editor2`, `viewer` and `noaccess` (seats held through the groups `/app/admins`,
  `/app/editors` and `/app/viewers`) and a dedicated `aj-boilerplate-admin` client for the Auth service's directory
  admin, for running the API and Auth service against a local Keycloak.
- **Documentation**: [`docs/auth/`](docs/auth/README.md) (overview, setup, troubleshooting, contract),
  ADR-0012, and a rewritten `README.md` and `CLAUDE.md`.

### Changed

- Auth service defaults equal the Al Jazeera USP staging Entra values (scopes
  `openid profile offline_access User.Read`, `RequestOwnClientIdScope=false`, `SubjectIssuer=oidc`).
  Azure AD B2C stays available by configuration.
- A patched `System.Security.Cryptography.Pkcs` is pinned. `Microsoft.Data.SqlClient` 6.1.1 pulls a
  version that now carries two advisories, which fail restore under warnings-as-errors.
- The comments and XML documentation that still said the cloud switch picks an identity issuer, and the
  one for `Auth:AllowInsecureUpstreams` (it applies to five upstreams, not one), now say what the code does.

### Removed

- The API's `Authentication:GoogleCloudIdentity` and `Authentication:EntraId` schemes, and
  `Keycloak:Realm` and `Keycloak:ClientSecret`.

### Not in this release

- **Nothing deploys the Auth service or Keycloak.** `infra/` and `deploy.yml` provision and roll out the
  API, its database, Redis and secrets only. Projects using their own Auth service deploy it themselves.
- **This repository's `docker-compose.yml` runs SQL Server, Redis and the API only**, with no Keycloak
  and no Auth service. The full-stack repository has the one-command sign-in stack.
- **No real Entra tenant has been exercised.** The full-stack repository's local stack replaces Entra
  with a second Keycloak client. See [docs/auth/setup.md](docs/auth/setup.md).

### Upgrade steps from 0.1.x

1. **Decide your provider.** Outside USP: you deploy the Auth service. Inside the Al Jazeera USP
   platform: the shared gateway and auth-backend do the job and **you do not deploy the Auth service**;
   ask the USP platform team to onboard your host and client
   ([setup, section 3](docs/auth/setup.md#3-onboarding-into-usp-auth_providerusp)).
2. **API configuration.** Delete `Authentication__*`, `Keycloak__Realm` and `Keycloak__ClientSecret`. Set
   `Keycloak__ValidIssuer`, `Keycloak__JwksUri` (https) and `Keycloak__ClientId`. If your Keycloak serves
   its JWKS over http, decide deliberately about `Keycloak__AllowInsecureJwksTransport`.
3. **Keycloak.** The portal client needs token exchange, client roles `admin`/`editor`/`viewer` (plus
   any permissions) and a flat `roles` claim mapper. See [setup, section 2b](docs/auth/setup.md#2b-keycloak).
4. **Migrations.** Apply the app database's migrations (`AddAuthenticationEvents`, `AddSessionEndAndDisplayName`). Create
   the Auth database and apply the Auth service's migrations.
5. **Compose and CI.** Take the new `.env.example`. Add the Auth service image build and deploy to your
   pipeline; the boilerplate's `deploy.yml` does not.
6. **ADRs.** Continue your own series at the next free number after `0012`.
7. Run the gate. Then sign in as a user with a role, as one without, and remove a role in Keycloak to
   watch the access-revoked dialog appear.

---

## [0.1.1] — 2026-09-01

A patch release, and every entry in it is the same story: a tree that was green in August
went red in September without anybody touching it. An advisory was published against a
package nobody referenced directly, and one instruction in the upgrade guide went stale
because the boilerplate grew. Nothing changes shape — but if you kept the sample
integration fixture, there is a two-line change to make.

### Fixed

- `docs/upgrading.md` told you to start your own ADR series at `0008`, and called the shipped
  set "the seven ADRs". There are eleven, so `0008`–`0011` are taken and that advice collided
  with real files. It now says to count `docs/adr/` at the version you cloned, because this
  number moves with every release. **If you already numbered your own ADRs from `0008`, you
  have a clash to reconcile** — `docs/adr/README.md` is explicit that numbers are never
  reused or renumbered, so renumber *yours*, not the boilerplate's.

- `Testcontainers.MsSql` 4.4.0 → 4.14.0. 4.4.0 pulls `SSH.NET` 2024.2.0 transitively, which
  now carries a high-severity advisory (GHSA-q939-rpr3-3284, patched in 2026.0.0). Because
  `TreatWarningsAsErrors` is on, this fails `dotnet restore` with `error NU1903` — the build
  goes red on a package nobody referenced directly, on a machine that never changed. 4.14.0
  resolves `SSH.NET` 2026.0.0. **Nothing to do but take it**, unless you had pinned
  Testcontainers yourself. One source change came with it: `MsSqlBuilder()`'s parameterless
  constructor is obsolete in 4.14.0, so the image moves into the constructor
  (`new MsSqlBuilder("mcr.microsoft.com/mssql/server:2022-latest")`) and the `.WithImage(...)`
  call is dropped. If you kept the sample fixture, apply the same two-line change.

---

## [0.1.0] — 2026-08-04

The initial extraction, published as three repositories. Everything below is the starting
state rather than a change from something; subsequent entries will read as changes.

### Added

**Backend — .NET 10**

- Layered Clean Architecture: `Domain` → `Application` → `Contracts` → `Infrastructure` →
  `Api`, with an architecture test suite that fails the build when the dependency direction
  is violated ([ADR-0001](docs/adr/0001-layered-clean-architecture.md)).
- A uniform `ApiResponse<T>` envelope with a stable machine-readable `code`, a `traceId`,
  and a documented status-code contract. Exceptions map to responses through an ordered
  handler chain rather than through try/catch in controllers
  ([ADR-0005](docs/adr/0005-apiresponse-envelope-and-status-code-contract.md)).
- EF Core 10 against SQL Server, migration-based, with three migrations in the box so the
  workflow is demonstrated rather than described.
- Transactional outbox and inbox, a health-check split into liveness and readiness,
  fixed-window rate limiting, security headers, forwarded-headers handling, opaque entity
  identifiers, Serilog with URL sanitisation, and OpenTelemetry traces and metrics.
- Two clouds behind one switch: `CLOUD_PROVIDER=gcp|azure` selects the secrets provider and
  the identity issuer at the composition root
  ([ADR-0002](docs/adr/0002-dual-cloud-provider-behind-one-switch.md)).
- Idempotency keys for unsafe requests: opt-in, POST-only, replaying a buffered response
  from a unique-indexed table
  ([ADR-0009](docs/adr/0009-idempotency-keys-for-unsafe-requests.md)).
- No audit trail and no soft delete, both absent deliberately and both explained
  ([ADR-0008](docs/adr/0008-no-audit-trail-or-soft-delete.md)).
- The OpenAPI document committed at [`docs/api/openapi.json`](docs/api/openapi.json) and
  checked against the code by `scripts/generate-openapi.sh --check`, so a contract change
  cannot land unnoticed.

**The "What's new" feature announcements**

- The server side of a feature spotlight: which announcements are live, which route
  prefixes they are bound to, and who has already dismissed which. Acknowledgement is
  stored per user rather than per browser, so it survives cleared storage and other
  devices. Shipping the next announcement is an INSERT-only migration and no code at all.
  The client that renders it is not built here. See
  [docs/whats-new.md](docs/whats-new.md).

**Infrastructure**

- Terraform for Google Cloud (Cloud Run, Cloud SQL, Memorystore, Secret Manager) and Bicep
  for Azure (Container Apps, Azure SQL, Cache for Redis, Key Vault), provisioning the same
  logical shape.
- Deployment through dev → staging → prod with GitHub Environment protection rules as the
  approval gates, authenticating by OIDC with no long-lived cloud credentials anywhere.

**The agentic harness**

- `.claude/` ships committed, not gitignored: hooks, slash commands, standards, agents, and
  templates, so every developer and every CI run gets identical guardrails.

**Quality gate**

- Build with warnings as errors, `dotnet format` verification, unit + integration +
  architecture tests, an OpenAPI contract check, SonarQube Community Build (zero new
  Blocker/Critical/Major, ≥80% coverage on new code), Gitleaks, CodeQL, and dependency
  vulnerability scanning.

**Process documentation**

- A five-stage workflow, a Definition of Done, a spec template, a Day-1 onboarding
  checklist, and an architecture guide that matches the code.

**Repository governance and toolchain** *(this release)*

- `LICENSE` — an explicit all-rights-reserved notice recording that the licence choice is
  pending. It grants and removes nothing; it replaces a guess with a statement.
- `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `CODEOWNERS`, a pull-request
  template that embeds the Definition of Done checklist, and issue forms.
- **Conventional Commits**, enforced by `.claude/hooks/commit-msg.sh` in both git-hook and
  agent-hook mode, with a `COMMIT_MSG_SKIP=1` opt-out.
- `.gitattributes` — normalises line endings so a Windows clone does not fail
  `dotnet format` and does not break every shell hook with `bad interpreter`.
- `.env.example` — every environment variable the repository reads, what it is for, and
  whether it is required.
- A **root `docker-compose.yml`** bringing up SQL Server, Redis, and the API together, with
  migrations as a discrete, profile-gated step rather than a startup side effect.
- `.vscode/extensions.json` and `settings.json`, matching `.editorconfig` rather than
  competing with it.
- SonarQube configuration as files rather than inline CI arguments, so a local scan and a
  CI scan analyse the same code under the same rules.
- `.github/dependabot.yml` covering NuGet, GitHub Actions, and Docker base images, grouped
  and weekly so it does not produce pull-request spam.
- `.github/workflows/supply-chain.yml` — builds the API container image, scans it with
  Trivy, fails on fixable HIGH/CRITICAL, and publishes CycloneDX and SPDX SBOMs. Runs
  weekly as well as on change, because an image goes stale without a commit. The allowlist
  (`.trivyignore.yaml`) requires a justification and an expiry date, and CI fails when an
  entry outlives its date.
- **Database migrations wired into deployment.** `deploy.yml` applies a migration bundle
  built by CI from the same commit, before the rollout, with the expand → migrate →
  contract rule documented in full.
- `docs/incidents/` with a template and guidance, `docs/upgrading.md` for consumers pulling
  improvements back in, and the one-way derivation that produces this repository from the
  full-stack one, recorded in
  [ADR-0011](docs/adr/0011-scripted-one-way-derivation-for-the-three-repositories.md).

### Known limitations

Stated here rather than discovered later:

- **No licence is set.** Until `LICENSE` is replaced with a real one, no reuse rights are
  granted and outside contributions cannot be accepted.
- **Placeholders must be filled before this repository is useful to anyone else**: the
  CODEOWNERS handles, the `<owner>/<repo>` URLs in `SECURITY.md` and the issue-template
  config, the Code of Conduct enforcement contact, and the SonarQube project keys.
- **No pull-request decoration from SonarQube.** Community Build analyses one branch;
  findings appear on the dashboard and through the local pre-push hook, never as PR
  comments.
- **Derivation is not automatic.** The script upstream regenerates this repository, but
  three files still need a human on every run, and nothing checks the published
  repositories for drift after the fact.

[Unreleased]: https://github.com/<your-org>/aj-boilerplate-be/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/<your-org>/aj-boilerplate-be/releases/tag/v0.1.0

### Also in this first tag

Fixes made between writing the entries above and cutting the tag.

- **The migration bundle could not be built.** It left the compile to `dotnet ef`, which
  reports only "Build failed. Use dotnet build to see the errors", so the CI log never named
  a cause. The Release build is an explicit step now and the bundle runs `--no-build` against
  verified output.
- **The container scan could not start.** The Trivy action was pinned to `0.28.0`, which does
  not exist as a tag, so the supply-chain workflow failed to resolve it on every run.
- **`generate-openapi.sh` looked outside the repository.** It counted `../..` up from the
  backend directory, which is correct in the full-stack tree and wrong here, where the
  backend IS the root — so the contract gate reported a missing document rather than a wrong
  path. It asks git for the root now.
