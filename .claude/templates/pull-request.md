# PR: <concise title>

## Summary

What changed and why. Link the spec in `docs/specs/` and any ADR in `docs/adr/`.

## Changes

- **Layers touched:** <Domain / Application / Contracts / Infrastructure / Api>
- **Schema:** <migrations added, or "none">
- **Docs / infra:** <OpenAPI snapshot, ADRs, IaC>

## Quality gate (green before requesting merge)

- [ ] `dotnet build --no-incremental` clean (warnings are errors)
- [ ] `dotnet format --verify-no-changes` clean
- [ ] `dotnet test` green (unit + integration + architecture)
- [ ] `dotnet list package --vulnerable --include-transitive` clean
- [ ] SonarQube — **0** Blocker / Critical / Major, coverage on new code ≥80%
- [ ] Gitleaks clean
- [ ] Schema change shipped as a reviewed migration (if applicable)

## Architecture & standards

- [ ] Clean Architecture boundaries respected (`Application` still does not reference
      `Contracts`; `Api` still does not reference `Domain`)
- [ ] No EF Core entity on the wire, in either direction
- [ ] `IClock` rather than `DateTime.UtcNow`
- [ ] No `EnsureCreated`; no manual DDL

## API contract

- [ ] Versioned route; `ApiResponse<T>` envelope with `traceId`
- [ ] Correct status codes, including the hide-as-404 / 409-concurrency / 410-vs-404 calls
- [ ] Every status code annotated; `docs/api/` snapshot refreshed

## Security

- [ ] Deny-by-default policy on every endpoint
- [ ] Object ownership validated after loading the resource
- [ ] Restricted fields removed by DTO projection, with a test proving absence from the payload
- [ ] No secrets, real hostnames, project ids, or credentials

## Notes / risks

<remaining risks, follow-ups, anything a reviewer should look at hardest>

> Push requires explicit approval, every time. The SonarQube gate must pass first.
