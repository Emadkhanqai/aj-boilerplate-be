# Architecture Decision Records

An ADR records **why** a decision was made, at the time it was made. It is not documentation of
how the system works today — that is what `CLAUDE.md` and the standards are for. An ADR stays
true even after the decision is reversed, because it describes a moment.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-layered-clean-architecture.md) | Layered Clean Architecture, not vertical slices | Accepted |
| [0002](0002-dual-cloud-provider-behind-one-switch.md) | Two cloud providers behind a single `CLOUD_PROVIDER` switch | Accepted |
| [0005](0005-apiresponse-envelope-and-status-code-contract.md) | Uniform `ApiResponse<T>` envelope and status-code contract | Accepted |
| [0006](0006-three-repository-split.md) | Publish as three repositories derived from one tree | Accepted |

**The gap at 0003 and 0004 is intentional.** Those two decisions — PrimeNG as the sole component
library, and generating the client's API types from OpenAPI — are frontend decisions, so they ship
in [`aj-boilerplate-fe`](https://github.com/Emadkhanqai/aj-boilerplate-fe) and in the full-stack
repository instead. The numbers are not reused, so a reference to "ADR-0004" means the same
document everywhere. See [ADR-0006](0006-three-repository-split.md).

These record the decisions taken when this boilerplate was built. Keep them as history and start
your own series at `0007`, or delete them and start at `0001` — but pick one and be consistent.

## Writing one

1. Copy [TEMPLATE.md](TEMPLATE.md) to `NNNN-short-slug.md`, using the next free number.
2. Fill in context, decision, consequences (including the negative ones), and the alternatives
   you actually considered.
3. Open it as its own pull request, or alongside the change it governs.
4. Set the status to `Accepted` when it merges.

**Never edit an accepted ADR to reflect a new decision.** Write a new one, mark it as superseding
the old, and set the old one's status to `Superseded by ADR-NNNN`.

## When to write one

Write an ADR when the decision is expensive to reverse, crosses team or layer boundaries,
constrains future work, or will provoke "why is it like this?" from someone who was not there.

Do not write one for a choice a single pull request can undo.

Reading the most recent five is part of [Day-1 onboarding](../onboarding.md).
