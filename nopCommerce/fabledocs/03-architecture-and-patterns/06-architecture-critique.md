# Architecture Critique

The opinionated file. Everything here is labeled: **confirmed** (evidence read) vs **hypothesis** (would need runtime/production data). This doubles as system-design interview material — cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices

1. **The uniform write spine** (confirmed). Every default write → entity event → cache eviction ([EntityRepository](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L455) + [CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53)). Cache correctness as a *systemic property* rather than per-feature discipline is what lets a 40-subdomain codebase cache aggressively without constant staleness bugs.
2. **Capability interfaces** (confirmed). Localization/ACL/store-mapping/soft-delete as per-entity opt-ins ([Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13)) kept 20 years of feature growth out of inheritance hell.
3. **The factory layer** (confirmed). Entities never reach views; the DB shape never becomes a UI contract. Boring, disciplined, and the reason the storefront could be re-skinned/themed safely.
4. **Warnings-as-data validation** (confirmed). Business failures flow as renderable values ([AddToCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582)), keeping services presentation-free.
5. **Operator-first pragmatism** (confirmed). In-app scheduler, DB-as-queue, install-wizard config: everything runs on one cheap Windows host with zero extra infrastructure. Most "improvements" people propose break this constraint — recognize the constraint before critiquing (this sentence is a senior interview move by itself).

## Risks and tradeoffs (prioritized)

| # | Risk | Evidence | Impact | Likelihood | Label |
| --- | --- | --- | --- | --- | --- |
| 1 | Payment captured before order persisted; crash ⇒ money without order | [PlaceOrderAsync L1610-L1621](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1610-L1621) | high (money, trust) | low per-order, certain at scale | confirmed gap, mitigated by guid+logs |
| 2 | Double-order guard is single-node (named Mutex) + sync-over-async | [L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718) | med-high in LB deployments | med | confirmed limitation (documented in-code) |
| 3 | Sync in-process event consumers on hot flows; failures logged, not surfaced | [EventPublisher L25-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L25-L50) | med | med | confirmed behavior; impact = hypothesis |
| 4 | No cross-write transactions in multi-step flows; read-check-write races (cart max, lockout counter) | [ShoppingCartService L1637](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1637), [CustomerRegistrationService L160](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160) | low-med each, broad surface | med | confirmed pattern |
| 5 | Fail-open defaults: unmapped entity visible everywhere; unattributed admin action open inside area gate | [StoreMappingService L204](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204), [pattern 8](05-pattern-catalog.md) | high if hit | low, review-dependent | confirmed defaults |
| 6 | Exhausted queued emails fail silently | [QueuedMessagesSendTask L43](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43) | low-med | high (eventually) | confirmed |
| 7 | `attributesXml` stringly-typed contract through cart/order | [AddToCartAsync L1550](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1550) | med (defect class) | ongoing | confirmed design debt |
| 8 | Everything-public-virtual freezes internals as API | [pattern 16](05-pattern-catalog.md) | med (velocity) | certain | confirmed, *intentional* |

(Fuller table with suggested tests/fixes: [09-reference/risk-register.md](../09-reference/risk-register.md).)

## "Owning this for 3 months" — what I'd actually change

In order, each with migration path and test strategy; note none of them is "rewrite it":

1. **Idempotency by constraint** (attacks risks 1+2). Add a unique index on the order's payment-request guid; on violation, return the existing order. Migration: additive index (expand-only); code change localized to `PlaceOrderAsync`. Test: two concurrent calls with one guid ⇒ one order — finally *testable*, unlike the mutex. Then the mutex block becomes removable dead weight behind its setting.
2. **Alert on dead letters** (risk 6). Admin dashboard warning + optional notification when `SentTries >= max && SentOnUtc == null`. Additive, tiny blast radius; test at service level. (Junior-ticket sized — see [ticket 6](../06-contribution-practice/01-good-first-tickets.md).)
3. **Reflection guard for fail-open authz** (risk 5). The CI test from [ticket 15](../06-contribution-practice/01-good-first-tickets.md): every admin mutating action must carry `[CheckPermission]` or be explicitly excluded. Converts a review habit into an invariant.
4. **Async event dispatch tier** (risk 3). Introduce an opt-in durable consumer queue for slow/critical consumers (table-backed, like email). Migration: new interface `IDurableConsumer<T>`, publisher routes accordingly; existing consumers untouched. Test: kill-process-mid-fanout scenarios move from "lost" to "replayed."
5. **Not**: replacing `attributesXml` (risk 7) — the blast radius (every plugin, import/export, saved carts in production DBs) exceeds three months. Instead: wrap it behind a typed facade in new code paths and stop the bleeding. Explicitly declining a fix, with reasons, is the most senior line in this file.

## Hypotheses needing production data

Factory fan-out cost on large catalogs (profile before believing [scenario 4](../05-quality-engineering/03-systematic-debugging.md)); real-world cache hit rates justifying prefix-eviction coarseness; whether any deployment actually runs multi-node with memory cache (making risks 2/BB worse than they look).

Drill: pick risk 4. Write the one-page RFC (template in [07-career/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)) for fixing the cart max-items race — constraint vs transaction vs "accept it, add a log." Strong answer includes the traffic level at which "accept it" stops being right.
