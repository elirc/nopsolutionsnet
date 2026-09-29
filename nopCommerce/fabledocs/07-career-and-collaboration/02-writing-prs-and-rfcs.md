# Writing PRs and RFCs

## PR descriptions

Five sections, always, even for small PRs (small PRs get small sections):

```
What: one sentence of outcome, not mechanism.
Why: the problem/issue link. (nopCommerce convention: reference the GitHub issue —
     note how commits do it: "#8254 Move GetDefaultCulture method..." )
How tested: exact commands + what you verified manually. "dotnet test --filter ..." beats "tests pass".
Risks: what could this break, and why you believe it doesn't. Name the blast radius explicitly —
       in THIS repo that includes plugin/override compatibility for any public-virtual change.
Follow-ups: what you deliberately did NOT do.
```

The "Risks" section is where mid-level shows. Compare: "low risk" (says nothing) vs "touches `AddToCartAsync`'s signature — additive overload only, old signature delegates, so existing plugins compile unchanged; the new path is covered by tests X and Y."

## Commit messages

Repo convention (read `git log`): `#<issue> <imperative summary>`. Keep the first line under ~70 chars; body explains *why* when non-obvious. One logical change per commit; migrations in their own commit when practical (they're the non-revertible part — reviewers want them isolated).

## When to write an RFC instead

Write the design doc first when any of these is true: schema changes beyond additive; any `public virtual` signature change; new infrastructure (queues, caches, locks); security-model changes; anything on the checkout path. (Notice these are exactly the [risk-register](../09-reference/risk-register.md) categories — RFC-worthiness *is* risk.)

## RFC template tailored to this repo

```markdown
# RFC: <title>            Status: draft | reviewed | accepted | rejected
## Problem
What breaks/costs today. Anchor to code+line and, if possible, an issue or measurement.
## Constraints (this repo's non-negotiables)
- Runs on commodity single-host deployments (no new required infra)
- Plugin/override ecosystem compatibility (public-virtual surface)
- Three databases (SQL Server, MySQL, Postgres) — features must work on all
- Forward-only migrations; upgrade path from prior versions
## Proposal
Design + the contract it commits us to. Include the DO-NOT list (explicit non-goals).
## Alternatives considered
At least two, each with the reason it loses. "Do nothing" is always one of them.
## Migration & rollout
Sequenced releases; every intermediate state shippable. Settings/flags used.
## Test strategy
What pins the new invariants; what regression coverage protects the old ones.
## Risks & rollback
Blast radius, failure signals ("how would we know"), and the rollback lever per phase.
```

Practice: write the P1 RFC (idempotent orders, [senior projects](../06-contribution-practice/03-senior-build-projects.md)) with this template. Grade — Solid: constraints section reflects the repo's reality, not generic best practice. Strong: your "alternatives" section can steelman the *current* mutex design using the authors' own comment ([L1685-L1688](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1685-L1688)).

Interview angle: "walk me through a technical decision you drove" is an RFC narrated aloud: problem → constraints → alternatives → decision → rollout → result. Practicing the written form *is* practicing the answer.
