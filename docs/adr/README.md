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
| [0008](0008-no-audit-trail-or-soft-delete.md) | No audit trail and no soft delete — the ingredients ship, the policy does not | Accepted |
| [0009](0009-idempotency-keys-for-unsafe-requests.md) | `Idempotency-Key` is opt-in, POST-only, and replays a buffered response from a unique-indexed table | Accepted |
| [0011](0011-scripted-one-way-derivation-for-the-three-repositories.md) | Derivation of the single-stack repositories is a committed one-way script | Accepted |

**The gaps at 0003, 0004, 0007, and 0010 are intentional — nothing is missing.** Those four
decisions are about the client, so they ship in
[`aj-boilerplate-fe`](https://github.com/Emadkhanqai/aj-boilerplate-fe) and in the full-stack
repository, and there is nothing here for them to describe. The numbers are deliberately not
reused, so a reference to "ADR-0004" means the same document in every one of the three
repositories. See [ADR-0006](0006-three-repository-split.md).

These record the decisions taken when this boilerplate was built. Keep them as history and start
your own series at `0012`, or delete them and start at `0001` — but pick one and be consistent.

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
