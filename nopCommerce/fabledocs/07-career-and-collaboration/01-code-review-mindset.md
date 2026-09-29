# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path, obvious breakage.
2. **Is it correct?** — edge cases, concurrency, failure paths. (*Most review value lives here and reviewers skip it because it requires thinking, not scanning.*)
3. **Will it stay correct?** — tests pinning the behavior; invariants made explicit.
4. **Does it fit?** — follows the codebase's patterns or diverges with a reason.
5. **Is it kind to future maintainers?** — names, size, docs where non-obvious.

Order matters: nitpicking names (layer 5) on a PR with a race condition (layer 2) is the classic junior-reviewer failure. State the layer when you comment: "correctness:" / "fit:" / "nit:" — it calibrates severity for free.

## This repo's specific checklist

Derived from the patterns and risks documented in modules 03–05 — this is what "does it fit" means *here*:

- [ ] Writes go through the repository (event/cache spine intact — [pattern 4](../03-architecture-and-patterns/05-pattern-catalog.md)); predicate deletes and raw `Table` writes flagged consciously
- [ ] New admin action: `[CheckPermission]` present, read/write split respected, row-scope (vendor/store) handled imperatively ([key flow 6](../01-codebase-cartography/05-key-flows.md))
- [ ] New entity: capability interfaces *decided* (ACL? store mapping? soft delete?) and cache consumer added
- [ ] Schema change: builder + **both** installation and `UpgradeTo*` migration ([review kata 7](../04-code-reading-gym/04-review-katas.md)); forward-only survivable
- [ ] Business failures returned as warnings/results, not exceptions ([pattern 2](../03-architecture-and-patterns/05-pattern-catalog.md))
- [ ] Signature changes to `public virtual` members: additive-with-shim or flagged as breaking ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md))
- [ ] User-facing strings localized ([ticket 1's lesson](../06-contribution-practice/01-good-first-tickets.md))
- [ ] Entity never reaches a view; factories own model prep
- [ ] Security checklist bottom-of-file pass ([05-security-checklist](../05-quality-engineering/05-security-checklist.md))

## Comment language that works

Formula: **observation → consequence → question/suggestion.** Specific beats vague; questions beat commands; and always distinguish blocking from preference.

> **correctness (blocking):** `GetShoppingCartItemWarningsAsync` runs against the *new* total quantity on the update branch but this diff validates the increment only — two adds of 3 could pass where one add of 6 fails. Could we validate `existing + quantity` like [L1596-L1606](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1596-L1606) does?

> **fit:** this builds the model in the controller; everywhere else uses a factory (`Prepare*ModelAsync`). Moving it keeps the controller consistent with e.g. ProductController — happy to pair on it.

> **nit (non-blocking):** `GetData2Async` → maybe `GetProductsByVendorAsync`? Take it or leave it.

Anti-patterns to unlearn: "why didn't you just…" (hostile), approving with unaddressed blocking comments (spineless), rewriting the PR in comments (do it in a suggestion or offer to pair), and reviewing the *author* instead of the code.

Interview angle: review rounds grade exactly two things — do you find the layer-2 issue, and are you decent to work with while saying it. Practice both together on the [review katas](../04-code-reading-gym/04-review-katas.md); the kata answers include model comments.
