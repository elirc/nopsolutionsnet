# Refactor and Design Katas

Eight paper-first exercises. Output is a design note or a small local diff, never a big rewrite. Self-grading criteria per kata; the meta-skill is *sequencing changes so every intermediate state ships safely*.

## K1: Heal a boundary leak
Take the product-match rule in [ShoppingCartController.cs#L842-L849](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L842-L849) (domain rule in a controller). Design its move into `ShoppingCartService` such that the WebApi plugin path gets it for free.
Grade — Solid: new service-side check, controller delegates, warning-list style preserved. Strong: notes that the service signature is public API ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)) and chooses an additive path.

## K2: Design the outbox upgrade
Write the one-page RFC for [project P2](03-senior-build-projects.md) (durable consumers), but *only* the contract section: delivery guarantees, ordering, idempotency requirements on consumers, failure visibility.
Grade — Solid: at-least-once stated with consumer obligations. Strong: names which current consumers must *not* migrate (fast, best-effort ones) and why a blanket migration would be worse.

## K3: Split a god class (on paper)
[OrderProcessingService](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs) is 3,561 lines. Propose a 3-way split (placement / payment-lifecycle / fulfillment?) with: what moves, what stays, and the compatibility shim plan for `protected virtual` members ([review kata 5](../04-code-reading-gym/04-review-katas.md) is the warm-up).
Grade — Solid: cohesive split with dependency direction. Strong: sequencing where each release compiles *and* preserves override points; explicit "not worth it" option considered.

## K4: Remove duplication without creating coupling
Find the near-identical warning branches in [AddToCartAsync#L1560-L1576](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560-L1576) (three permission/actor gates). Design the helper — then argue *against* it: three explicit branches may beat one clever helper.
Grade — Strong: you can argue both sides and pick one with a reason tied to change-frequency, not aesthetics.

## K5: Improve type safety incrementally
The typed-attribute facade ([project P4](03-senior-build-projects.md)) at kata scale: design just the `ProductAttributeSelection` type's surface (parse/serialize/equality) and one adopted call site.
Grade — Solid: round-trip safe. Strong: equality semantics thought through (attribute order? case? whitespace?) — that's where cart-merge bugs ([FindShoppingCartItemInTheCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1589)) would come from.

## K6: Design a migration under load
Products table, 10M rows, add `NOT NULL` column with computed default. Write the expand/backfill/flip/contract plan ([02-data-model](../03-architecture-and-patterns/02-data-model-and-persistence.md)) as concrete FluentMigrator steps in this repo's convention ([UpgradeTo500 example](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7)).
Grade — Solid: 4 phases, each survivable. Strong: batched backfill with progress persistence; names the lock behavior difference across the three supported databases (or admits which needs research — honest uncertainty scores points).

## K7: Reduce an N+1 with proof
Take [ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918). Produce: measurement harness (query counting), the batched version using [GetByIdsAsync](../../src/Libraries/Nop.Data/EntityRepository.cs#L157), and the *decision memo* — ship it or not, based on your numbers.
Grade — Strong: the memo honestly concludes from data (either conclusion can be correct).

## K8: Review a flawed design (not a diff)
A teammate proposes: "cache `AuthorizeAsync` permission results per customer for 24h to speed up the admin area." Write your review referencing how [PermissionService](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258-L331) already caches (per *role*, event-evicted) and what per-customer 24h TTL would break (role changes, permission revocation latency — a security property, not a perf knob).
Grade — Solid: identifies the revocation-latency problem. Strong: proposes the measurement that would show whether any change is needed at all, and keeps the review kind.

---

Meta-rubric: junior refactors change code; mid-level refactors preserve behavior with tests; senior refactors are *sequenced deployments* where the ecosystem never notices. Every kata answer should state its sequencing.
