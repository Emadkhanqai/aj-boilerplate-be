# The API contract

**The OpenAPI document is the contract.** It is the single source of truth for every request
shape, response shape, status code, and error code that crosses this service's boundary. If
something is not in the OpenAPI document, a consumer cannot rely on it.

This repository is the **producer** of that document. Client code is not written here — consumers
generate their own clients from what this API publishes.

See [ADR-0005](../adr/0005-apiresponse-envelope-and-status-code-contract.md) for the envelope
every response uses.

---

## Where the document comes from

It is produced **from the code**, not written by hand. The API generates it from the controllers
and the types in `AjBoilerplate.Contracts`, so it cannot describe an endpoint that does not
exist or a shape the server does not actually serialise.

With the API running:

| Artefact | URL |
|---|---|
| OpenAPI JSON | <http://localhost:5080/swagger/v1/swagger.json> |
| Interactive UI | <http://localhost:5080/swagger> |

The quality of the document is entirely determined by the quality of the annotations. That makes
the following non-optional on every action:

- `[ProducesResponseType(typeof(ApiResponse<XDto>), StatusCodes.Status200OK)]` — and one for
  **every** status code the action can return, including the failures.
- `[ApiVersion]` and a versioned route: `/api/v{version}/{resource}`.
- XML documentation comments on contract types and their properties — they become the
  descriptions in every generated client and in the Swagger UI.
- Accurate nullability. An optional field and a required one produce different generated code,
  and the difference is the whole point.
- Enums declared as enums, not as strings, so a generated client gets a closed type.

An action with a single `ProducesResponseType` for the happy path produces a client that has no
idea how the endpoint fails. Treat a missing failure annotation as a bug.

The conventions behind all of this are in
[`.claude/standards/swagger-openapi.md`](../../.claude/standards/swagger-openapi.md).

---

## How consumers use it

Consumers **generate** their client from this document; they do not hand-write types that mirror
these contracts. That is not this repository's job to enforce, but it is this repository's job to
make possible — which is what the annotation rules above are for.

The practical consequence for work done here: the OpenAPI document is a published artefact, and
changing it is a contract change even when no C# consumer exists yet. Sloppy annotations produce
sloppy clients somewhere you cannot see.

A useful habit is to fetch `/swagger/v1/swagger.json` after a contract change and read the diff
before opening the pull request. It is the same diff a consumer will regenerate from.

---

## Changing the contract

The order matters, and it is always this order:

1. **Spec.** Agree the endpoints, DTOs, status codes, and error codes in
   [the spec template](../specs/TEMPLATE.md) §3, before any code. New `code` values are declared
   there.
2. **Server.** Implement it, with complete annotations.
3. **Verify the document.** Run the API and read the generated OpenAPI diff.
4. **Announce.** A breaking change needs a new version and an ADR before consumers are asked to
   move.
5. **Review.** The contract diff is part of the pull request and is reviewed, not skimmed.

Never run this backwards. Writing a client type first and making the server match it is how a
contract stops describing the system.

---

## Versioning and breaking changes

Routes are versioned from day one: `/api/v1/items`. Version 1 exists before there is any reason
for a version 2, so introducing one is a routine change rather than a migration.

A change is **breaking** if it removes or renames a field, narrows a type, makes an optional
field required, changes a status code, changes an existing `code` value, or removes an enum
member. Breaking changes need a new API version and an ADR.

A change is **additive** — and safe within a version — if it adds an optional field, adds a new
endpoint, adds a new `code` value, or adds an enum member the client can treat as unknown.

The spec template's breaking-change checklist (§3) exists to force this judgement before the code
is written rather than after a consumer breaks.

Full rules: [`.claude/standards/api-versioning.md`](../../.claude/standards/api-versioning.md).

---

## What clients are told to branch on

`code`, never `message`. `code` is a stable `SCREAMING_SNAKE_CASE` slug declared in
`AjBoilerplate.Contracts.Common.EnvelopeCodes`; `message` is human-readable and may be reworded
or localised at any time.

That contract only holds if it holds here: never invent a `code` inline in a controller, and
never reword a `code` because a message read better. Add the constant, document it in the spec,
and list it in the OpenAPI document.

Every failure response also carries `traceId` — the correlation identifier that appears in this
service's logs and traces. Consumers are expected to surface it, which is only useful if the
value is genuinely the one in the logs.
