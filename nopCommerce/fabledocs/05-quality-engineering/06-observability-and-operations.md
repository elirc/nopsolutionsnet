# Observability and Operations

**Observability** = can you answer "what is the system doing and why?" from its outputs, without adding code. This repo's answer is DB-backed and admin-visible — no metrics/tracing stack — which is the right *shape* for its self-hosted audience and a great foil for discussing what bigger deployments add.

## What exists

| Signal | Mechanism | Anchor |
| --- | --- | --- |
| Errors | `ILogger` → **Log table**, browsable in admin (System → Log) | writes at [OrderProcessingService#L1663-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1663-L1674), [EventPublisher#L40-L44](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L40-L44) |
| Unhandled exceptions | a second `UseExceptionHandler` that logs, then rethrows | [ApplicationBuilderExtensions.cs#L89-L116](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L89-L116) |
| Audit trail | activity log per customer action | [`InsertActivityAsync` calls: view product](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L194), [failed login](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L540), [place order](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1639) |
| Async state | queue tables *are* dashboards: QueuedEmail rows, ScheduleTask last-run times | [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs), [ScheduleTaskRunner#L71-L77](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L71-L77) |
| Order history | order notes record each notification enqueued, with queued-email ids | [OrderProcessingService.cs#L832-L850](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L832-L850) |
| Liveness | a `KeepAliveTask` that pings the store over HTTP, plus keep-alive middleware | [KeepAliveTask.cs](../../src/Libraries/Nop.Services/Common/KeepAliveTask.cs), [ApplicationBuilderExtensions.cs#L381-L384](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L381-L384) |
| Deploy/rollback | app is a folder + DB; migrations forward-only → rollback = backup restore | [migrations](../../src/Libraries/Nop.Data/Migrations), [Dockerfile](../../Dockerfile) |

What's absent (verified by search, 2026-10-06): no ASP.NET health-check endpoints (`grep -rn "HealthCheck" --include=*.cs src/Presentation src/Libraries` returns nothing), no metrics (request rates, latencies, cache hit ratios), no distributed tracing/correlation ids, no alerting. "Logs you must remember to read" is the operative weakness — the system records almost everything and *notifies* about almost nothing.

## How a log line is actually written

[DefaultLogger](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs) is short and worth reading end to end:

1. **Level gate.** `IsEnabled` returns `false` for `Debug` and `true` for everything else ([DefaultLogger.cs#L92-L99](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L92-L99)). Debug logging is compiled-in off; there is no runtime knob.
2. **Ignore list.** `InsertLogAsync` drops the entry if *either* the short or the full message contains any word from `CommonSettings.IgnoreLogWordlist`, case-insensitively ([L47-L58](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L47-L58), [L211-L215](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L211-L215)). The full message is the exception's `ToString()` — the whole stack trace.
3. **Context capture.** Each row stores level, messages, customer id, page URL, referrer and — only if `StoreIpAddresses` is on — the IP ([L68-L81](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L68-L81)).
4. **Write.** A normal repository insert with `publishEvent: false` ([L217](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L217)) — i.e. the log is just another table in the **same database** as the business data.
5. **Retention.** `ClearLogTask` and `ClearLogAsync(olderThan)` delete old rows with the predicate delete ([DefaultLogger.cs#L128-L133](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L128-L133)).

Two operational consequences follow directly. A single overly broad ignore word (say, "timeout", or a namespace that appears in every stack trace) silently suppresses whole classes of errors — the setting is a *filter on evidence*, so changes to it deserve review. And because the log lives in the application database, the system's error visibility depends on the component most likely to be failing.

## Feature trace: an unhandled exception, from throw to admin grid

| Step | Code | Notes |
| --- | --- | --- |
| 1 | exception escapes a controller | — |
| 2 | first handler: developer page or `/Error/Error` ([L78-L87](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L78-L87)) | chosen by `DisplayFullErrorStack` or Development environment (L77) |
| 3 | second handler: resolve `IWorkContext` → current customer ([L104](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L104)) | may *insert a guest row* if the request had no cookie ([WebWorkContext.cs#L270-L274](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L270-L274)) |
| 4 | `ILogger.ErrorAsync(exception.Message, exception, customer)` ([L107](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L107)) | ignore-list filter, then DB insert |
| 5 | `finally { ExceptionDispatchInfo.Throw(exception); }` ([L110-L114](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L110-L114)) | rethrows the *original* exception so the error page renders |

## "How would I know this broke?" — per key flow

- **Product pages down/slow:** you wouldn't, from inside — no latency metrics. External uptime monitoring is mandatory ops hygiene for this stack. (Errors would appear in the Log table.)
- **Checkout failing:** error log entries with customer linkage ([#L1670-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1670-L1674)) + falling order counts in admin reports. Signal exists; *someone must look*.
- **A checkout side effect failing** (e.g. a tax or marketing plugin's `OrderPlacedEvent` consumer): one log row from the publisher ([EventPublisher.cs#L44](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L44)); the order itself looks perfect. See [03-architecture-and-patterns/04-side-effects-async-and-reliability.md](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).
- **Emails not sending:** QueuedEmail rows with `SentTries` exhausted — visible only if an admin opens that grid ([the gap behind ticket 6](../06-contribution-practice/01-good-first-tickets.md)). Order notes help here: each order records the queued-email ids it created, so you can go from an order to its exact email rows.
- **Background task dead:** last-run timestamps on the schedule-task admin page go stale (they are written before and after each run, [ScheduleTaskRunner.cs#L71-L77](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L71-L77)). A task with `StopOnError` disables itself on failure ([L149](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L149)) — and the only trace is a log row. No alert.
- **Login attacks:** `PublicStore.FailedLogin` activity entries spike ([CustomerController#L540](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L540)) — queryable evidence, unwatched by default.
- **Cache serving stale data:** *nearly silent* — at most a publisher log row if an eviction threw; nothing at all if a distributed-cache write was dropped (it is fire-and-forget, [DistributedCacheManager.cs#L182-L194](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L182-L194)). Detection is user reports; prevention is the write-spine discipline ([pattern 4](../03-architecture-and-patterns/05-pattern-catalog.md)).

## Operations model

Deploy = publish the folder; schema migrations run automatically when the process boots ([NopDbStartup.cs#L65-L71](../../src/Libraries/Nop.Data/NopDbStartup.cs#L65-L71)); per-store config lives in the DB (settings), machine config in `App_Data/appsettings.json` plus environment variables ([Program.cs#L14-L20](../../src/Presentation/Nop.Web/Program.cs#L14-L20)). **Rollback reality:** because migrations are forward-only ([ForwardOnlyMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L8)), rolling back a release that migrated schema means restoring a DB backup — so the operational discipline is: *backup before upgrade, additive migrations, feature toggles via settings*. This is a complete, honest answer to the interview question "how do you do deployments and rollbacks without Kubernetes?"

Background jobs: the repo ships 12 `IScheduleTask` implementations in `Nop.Services` plus 3 in plugins (found by grepping for `class … : IScheduleTask`), including `QueuedMessagesSendTask`, `ClearLogTask`, `DeleteGuestsTask`, `KeepAliveTask` and `ClearCacheTask`. Each node runs its own timers that POST to the store URL ([TaskScheduler.cs#L67](../../src/Libraries/Nop.Services/ScheduleTasks/TaskScheduler.cs#L67), [#L206](../../src/Libraries/Nop.Services/ScheduleTasks/TaskScheduler.cs#L206)); behind a load balancer, the node that *executes* a task is whichever one the balancer picks, so "which server ran the 03:00 cleanup?" is answered only by the log row's page URL, not by the scheduler.

## Exercise ladder

1. **Goal:** find where an unhandled exception gets logged.
   **Check:** cite [ApplicationBuilderExtensions.cs#L89-L116](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L89-L116) and explain why there are two `UseExceptionHandler` calls (one renders, one logs).
2. **Goal:** predict which log rows are dropped.
   **Check:** with `IgnoreLogWordlist = ["Nop.Services"]`, decide whether an `ErrorAsync("Payment failed", ex)` from inside `Nop.Services` is stored (answer: no — the full message is `ex.ToString()`, whose stack trace contains "Nop.Services").
3. **Goal:** list every scheduled task and what silence means for each.
   **Check:** run the grep above (12 files) and, for three tasks, write "if this stops, the first visible symptom is…".
4. **Goal (mid):** design an alert for exhausted emails without new infrastructure.
   **Check:** your design queries rows with `SentTries >= 3` and `SentOnUtc` null (the same predicate shape as [QueuedEmailService.cs#L162-L175](../../src/Libraries/Nop.Services/Messages/QueuedEmailService.cs#L162-L175)) inside a new schedule task, and sends one admin email per run when the count is non-zero — through the very queue it monitors, which you must call out as a weakness.
5. **Goal (mid):** add a health endpoint that tells the truth.
   **Check:** your plan registers ASP.NET Core health checks with one DB check and one cache check, maps the endpoint *without* the storefront filter stack (compare how [ScheduleTaskController](../../src/Presentation/Nop.Web/Controllers/ScheduleTaskController.cs#L6-L9) avoids `BasePublicController`), and explains why it must not create a guest customer.

## Senior finding: during a database outage, error logging goes dark

Read from [ApplicationBuilderExtensions.cs#L98-L114](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L98-L114) and [DefaultLogger.cs#L211-L218](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L211-L218). The logging handler (a) resolves the current customer, which itself queries — and may insert into — the database, then (b) writes the log row to the same database. If the database is the thing that's failing, step (a) or (b) throws; the `finally` then rethrows the *original* exception, so the logging failure is discarded and **nothing** is recorded anywhere the admin UI can show. The one incident class where you most need evidence is the one that leaves none. Confidence: high on mechanism; it is also typical of DB-backed logging generally. Mitigation: add an out-of-band sink (the built-in ASP.NET Core console/file logging providers, or `stdout` in the container) for errors raised in the exception handler itself, and keep the DB log as the convenient, admin-visible copy.

## Debugging scenario: "customers report errors, but System → Log is empty"

1. **Is the DB healthy?** If not, see the finding: empty logs are expected. Check container/host logs instead.
2. **Is something filtering?** Inspect `CommonSettings.IgnoreLogWordlist` for broad words; remember it matches the full stack trace ([DefaultLogger.cs#L55-L57](../../src/Libraries/Nop.Services/Logging/DefaultLogger.cs#L55-L57)).
3. **Was it purged?** `ClearLogTask` may have run; check its last-run time and the `olderThan` window.
4. **Is it a handled error?** Many failures never reach the logger as exceptions — add-to-cart and checkout return warnings to the user ([ShoppingCartService.cs#L1559-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582)) and only `PlaceOrderAsync` logs its aggregated errors ([OrderProcessingService.cs#L1670-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1670-L1674)). Ask for the exact on-screen message and grep the localization resources for it.
5. **Is it a different node?** All nodes log to the same table, so this is rarely the answer here — but it *is* the answer for anything written to local disk (PDF invoices saved by `SaveOrderPdfToDiskAsync`, [OrderProcessingService.cs#L836-L837](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L836-L837)).

## Drill

For any feature you build from [06-contribution-practice](../06-contribution-practice), fill in this table before writing code: *failure mode → signal produced → who sees it → within how long*. If any row ends in "nobody / never," either add a signal or write down why it's acceptable. Basic: table filled. Solid: at least one row led you to add logging/an admin indicator. Strong: you can defend an "acceptable silence" row out loud — deliberate non-observability is a legitimate budget decision, unconscious non-observability is not.

## Verification

2026-10-06: every anchor opened and checked. Resolved the earlier "*investigate* specific health-check endpoints" note: no health-check registrations exist (grep). Logger and exception-handler behaviour read from source; the database-outage finding is derived from the code path and was not reproduced. Scheduled-task count (12 in `Nop.Services`, 3 in plugins) from grepping `class … : IScheduleTask`; `ScheduleTaskRunner : IScheduleTaskRunner` was excluded as a false positive.
