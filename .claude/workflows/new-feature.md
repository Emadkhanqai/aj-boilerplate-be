# Workflow: New Feature

> **Model routing (do first):** classify the task and recommend a model — see
> [`../model-routing.md`](../model-routing.md). New-capability *design* → frontier tier; the
> *build and tests* that follow → workhorse tier. Say so if the current model is mismatched.

End-to-end flow for a new capability, from the domain out to the published contract.

## 1. Understand & plan

- **Read the spec** in `docs/specs/`. If there is no approved spec, run
  [`/spec`](../commands/spec.md) first — building without one is how scope drifts.
- Read the ADRs in `docs/adr/` that constrain the area, and the applicable
  [`../standards/`](../standards/).
- List the **invariants** the feature must hold. Each one becomes a test.
- Break the work down with [`/task`](../commands/task.md) if it is more than half a day.

## 2. Branch

`git switch -c feature/<short-desc>`. Never work on `main` directly.

## 3. Build it, inside out

1. **Domain** — entities and invariants, persistence-ignorant. Guard the rules inside the
   aggregate so an invalid state is unconstructible.
2. **Application** — commands/queries + handlers, ports, **FluentValidation on every request**
   ([`../standards/input-validation-sanitization.md`](../standards/input-validation-sanitization.md)).
   Enforce policy + scope + **object ownership after loading the resource**
   ([`../standards/owasp-security.md`](../standards/owasp-security.md)).
3. **Infrastructure** — EF Core configuration and repositories; any schema change goes through
   [`database-change.md`](database-change.md).
4. **Contracts** — DTOs and the **`ApiResponse<T>`** envelope. **Never bind EF entities.**
5. **Api** — thin **versioned** controllers (`/api/v1/...`); errors through the central handler
   chain with `traceId`; every status documented in OpenAPI. Respect the middleware order
   ([`../standards/middleware.md`](../standards/middleware.md)).
6. **Tests** — Unit + Integration + Architecture, including a negative authorization test.

## 4. Publish the contract

Complete the OpenAPI annotations — a `[ProducesResponseType]` for **every** status code the
action can return — and refresh the snapshot in `docs/api/` so the contract change shows up in
the diff. Consumers generate their clients from that document. See
[`api-change.md`](api-change.md).

## 5. Verify locally

```bash
dotnet build -warnaserror
dotnet test                # unit + architecture + integration (needs Docker)
```

## 6. Review & gate

Run [`/qa`](../commands/qa.md), then [`/review`](../commands/review.md), then
[`pre-push-quality-gate.md`](pre-push-quality-gate.md). Fix every Blocker/Critical/Major.
**Do not push without explicit approval.**
