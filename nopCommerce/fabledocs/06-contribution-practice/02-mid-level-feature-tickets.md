# Mid-Level Feature Tickets

Ten cross-layer tickets. The mid-level bar: **write the design note before code** — data model, layers touched, risk, rollback — and get it "reviewed" (rubber duck counts). Each ticket lists the layers so you can't skip one. All include: Risk, Rollback, ISP (interview story potential).

## M1: "Low stock" email digest for store owners
Layers: service + schedule task + message template + admin setting.
Story: daily digest of products under their low-stock threshold.
Design note must answer: new `IScheduleTask` ([pattern 7](../03-architecture-and-patterns/05-pattern-catalog.md)) or extend an existing task? Where does the threshold live (product-level fields exist — investigate `Product.MinStockQuantity`)? Template tokens?
Anchors: [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) as the task blueprint; message templates in `Nop.Services/Messages`.
Risk: noisy emails (digest, not per-product); double-send on multi-node. Rollback: disable the task row — zero code rollback needed (design for that!).
ISP: "I built an async feature and designed its failure/duplicate story before coding."

## M2: Per-role max order total
Layers: setting + service validation + admin UI + tests.
Story: block orders above a per-customer-role total.
Design note: enforce where? (order-placement validation, *not* the cart — justify); interacts with which existing check set ([PreparePlaceOrderDetailsAsync](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1602))?
Risk: blocking legitimate checkouts (fail-closed money path) — needs clear customer messaging via the errors-list pattern. Rollback: setting off by default.
ISP: "I added a rule to a money path with explicit fail-open/fail-closed reasoning."

## M3: Wishlist "notify me when back in stock" upgrade
Layers: entity/schema (subscription exists — `BackInStockSubscription`! investigate first) + service + UI.
Story: surface existing back-in-stock machinery on the wishlist page.
Design note: this is an *integration* ticket — inventory of what [BackInStockSubscriptionBuilder](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/BackInStockSubscriptionBuilder.cs) already supports before writing anything.
Risk: duplicating an existing feature badly. Rollback: UI-only revert.
ISP: "I discovered half the feature already existed and shipped the integration instead of a rewrite."

## M4: Admin warning banner for dead queued emails
Layers: service query + admin factory + dashboard view + test. (Big sibling of [junior ticket 6](01-good-first-tickets.md).)
Design note: query cost on the dashboard (every admin page load?) → cache with short TTL; threshold config.
Anchors: exhaustion semantics [QueuedMessagesSendTask#L43](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43).
Risk: slow dashboard. Rollback: widget removal.
ISP: "I added failure visibility to an async pipeline including the cost of the monitoring itself."

## M5: Structured pagination for a heavy admin grid
Layers: repository query + search model + factory + view.
Story: pick one admin list that loads eagerly and convert it to proper `GetAllPagedAsync` ([EntityRepository#L302](../../src/Libraries/Nop.Data/EntityRepository.cs#L302)) usage. (Finding the candidate is part of the ticket — audit admin factories for `.ToList()`-then-page shapes.)
Risk: subtle ordering changes user-visible. Rollback: revert; no schema.
ISP: "I found and fixed an unbounded query with before/after measurements."

## M6: Customer-facing order cancellation (window-limited)
Layers: setting + service (order status transition) + storefront UI + email + tests.
Design note: which statuses are cancellable; who refunds (link to `IPaymentService` capabilities — some gateways can't); idempotent cancel (double-click!).
Anchors: status machinery in [OrderProcessingService](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs); `CheckOrderStatusAsync` (L1647).
Risk: money + state machine = the riskiest ticket here; write the state diagram first. Rollback: feature setting off.
ISP: the strongest story in this file — state machines, money, idempotency in one.

## M7: Product comparison limit + persistence upgrade
Layers: cookie/session storage → DB-backed for logged-in customers.
Design note: storage choice tradeoffs (cookie size limits vs table growth); merge-on-login semantics (the cart already solves this — find and mirror its approach).
Risk: silent data loss on merge. Rollback: revert to cookie path.
ISP: "I designed client-vs-server state placement and a merge policy."

## M8: Rate-limit password recovery requests
Layers: service + generic attributes or new table + tests.
Design note: per-account and per-IP? Where does the counter live (compare the lockout counter design, [CustomerRegistrationService#L160-L168](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160-L168))? What does the user see (no enumeration!)?
Risk: locking out real users; enumeration via response differences. Rollback: setting.
ISP: "I built abuse protection with explicit anti-enumeration reasoning."

## M9: Export queued-email failures to CSV
Layers: service + ExportImport + admin action + permission.
Anchors: [Nop.Services/ExportImport](../../src/Libraries/Nop.Services/ExportImport) has the manager pattern; permission split rules from [key flow 6](../01-codebase-cartography/05-key-flows.md).
Design note: which permission gates the export (it contains customer emails — PII!).
Risk: PII in a downloadable file. Rollback: remove action.
ISP: "I treated an export feature as a data-governance decision, not a CSV writer."

## M10: Multi-node-safe run-once guard for schedule tasks
Layers: infrastructure + task runner + tests.
Story: ensure a task runs once per period across nodes.
Design note: investigate the current guard in [ScheduleTaskRunner#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) (`ensureRunOncePerPeriod`) — is it DB-backed or memory? Then: DB row claim with expiry (compare [DistributedCacheLocker](../../src/Libraries/Nop.Core/Caching/DistributedCacheLocker.cs)).
Risk: a broken guard silently *stops all background work* — the fail-closed direction is worse than double-runs for some tasks; classify tasks by idempotency first ([pattern 7 drill](../03-architecture-and-patterns/05-pattern-catalog.md)).
Rollback: guard behind a setting defaulting to current behavior.
ISP: "I hardened distributed execution and — more importantly — classified which jobs even needed it."

---

Rubric per ticket — Basic: it works locally. Solid: design note written first and honored, all listed layers touched, tests included. Strong: risk section proved out (you tested the failure path, not just the happy path) and the rollback lever actually works.
