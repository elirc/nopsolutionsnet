# Observability and Operations

**Observability** = can you answer "what is the system doing and why?" from its outputs, without adding code. This repo's answer is DB-backed and admin-visible — no metrics/tracing stack — which is the right *shape* for its self-hosted audience and a great foil for discussing what bigger deployments add.

## What exists

| Signal | Mechanism | Anchor |
| --- | --- | --- |
| Errors | `ILogger` → **Log table**, browsable in admin (System → Log) | writes at [OrderProcessingService#L1663-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1663-L1674), [EventPublisher#L40-L44](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L40-L44) |
| Audit trail | activity log per customer action | [`InsertActivityAsync` calls: view product](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L194), [failed login](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L540), [place order](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1639) |
| Async state | queue tables *are* dashboards: QueuedEmail rows, ScheduleTask last-run times | [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs), [ScheduleTaskRunner#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| Deploy/rollback | app is a folder + DB; migrations forward-only → rollback = backup restore | [migrations](../../src/Libraries/Nop.Data/Migrations), [Dockerfile](../../Dockerfile) |
| Health | ASP.NET hosting basics; warnings surfaced in admin dashboard | *investigate* specific health-check endpoints |

What's absent (and would be first additions at scale): metrics (request rates, latencies, cache hit ratios), distributed tracing/correlation ids, alerting. "Logs you must remember to read" is the operative weakness — the system records almost everything and *notifies* about almost nothing.

## "How would I know this broke?" — per key flow

- **Product pages down/slow:** you wouldn't, from inside — no latency metrics. External uptime monitoring is mandatory ops hygiene for this stack. (Errors would appear in the Log table.)
- **Checkout failing:** error log entries with customer linkage ([#L1670-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1670-L1674)) + falling order counts in admin reports. Signal exists; *someone must look*.
- **Emails not sending:** QueuedEmail rows with `SentTries` exhausted — visible only if an admin opens that grid ([the gap behind ticket 6](../06-contribution-practice/01-good-first-tickets.md)).
- **Background task dead:** last-run timestamps on the schedule-task admin page go stale. No alert.
- **Login attacks:** `PublicStore.FailedLogin` activity entries spike ([CustomerController#L540](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L540)) — queryable evidence, unwatched by default.
- **Cache serving stale data:** *silent by nature* — the one failure class with no signal at all. Detection is user reports; prevention is the write-spine discipline ([pattern 4](../03-architecture-and-patterns/05-pattern-catalog.md)).

## Operations model

Deploy = publish the folder, run migrations on first start; per-store config lives in the DB (settings), machine config in `App_Data/appsettings.json`. **Rollback reality:** because migrations are forward-only ([ForwardOnlyMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L8)), rolling back a release that migrated schema means restoring a DB backup — so the operational discipline is: *backup before upgrade, additive migrations, feature toggles via settings*. This is a complete, honest answer to the interview question "how do you do deployments and rollbacks without Kubernetes?"

## Drill

For any feature you build from [06-contribution-practice](../06-contribution-practice), fill in this table before writing code: *failure mode → signal produced → who sees it → within how long*. If any row ends in "nobody / never," either add a signal or write down why it's acceptable. Basic: table filled. Solid: at least one row led you to add logging/an admin indicator. Strong: you can defend an "acceptable silence" row out loud — deliberate non-observability is a legitimate budget decision, unconscious non-observability is not.
