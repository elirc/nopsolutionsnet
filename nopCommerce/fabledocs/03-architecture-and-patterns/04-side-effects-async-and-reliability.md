# Side Effects, Async Work, and Reliability

A **side effect** is anything a flow changes beyond its return value — rows, emails, caches, external APIs. Reliability engineering is the discipline of asking, for each one: *what if it fails? what if it runs twice? how would we know?* (Those three questions, verbatim, are a complete interview framework.)

## Map of side-effect machinery

| Mechanism | Sync? | Durable? | Retries? | Anchor |
| --- | --- | --- | --- | --- |
| Entity events → consumers | sync, in-process | no | no | [EventPublisher.cs#L20-L52](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52) |
| Domain events (`OrderPlacedEvent` etc.) | sync, in-process | no | no | [publish at OrderProcessingService.cs#L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1644) |
| Queued email (outbox) | async via scheduler | **yes** (DB rows) | bounded (3) | [QueuedMessagesSendTask.cs#L41-L90](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L41-L90) |
| Schedule tasks | async, in-process timer → HTTP POST to self | schedule in DB | per-run | [TaskScheduler.cs#L188-L231](../../src/Libraries/Nop.Services/ScheduleTasks/TaskScheduler.cs#L188-L231), [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| External APIs (payment, shipping, tax) | sync in request | n/a | plugin-specific | [ProcessPaymentAsync call, L1424](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1424) |
| Cache eviction | sync via events | no | no | [CacheEventConsumer.cs#L44-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) |
| Distributed cache writes | **fire-and-forget** | Redis/SQL | no | [DistributedCacheManager.cs#L182-L194](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L182-L194) |

Read the table as a decision record: **durable + retried** was spent only on email. Payment gets synchronous handling with idempotency hints; cache eviction gets best-effort. Matching *reliability budget* to *consequence of loss* is the senior skill — not making everything durable.

## The event publisher, precisely

Everything "event-driven" here goes through twenty lines ([EventPublisher.cs#L20-L52](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52)). Its exact semantics:

1. **Resolve all consumers at publish time** from the static engine ([L23](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L23)) — consumers were registered per closed `IConsumer<T>` interface by reflection ([NopStartup.cs#L307-L314](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L307-L314)). Order between consumers is registration order, which depends on assembly scanning — treat it as unspecified.
2. **Await each consumer in turn** ([L25-L30](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L25-L30)). The publisher's caller waits for *all* consumers. No parallelism, no queue.
3. **Stop early only if asked** — an event implementing `IStopProcessingEvent` with `StopProcessing = true` ends the loop (L32-L33).
4. **Swallow and log exceptions per consumer** ([L35-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L35-L50)), then continue with the next one — *except* that if resolving the logger returns `null`, the method `return`s ([L41-L42](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L41-L42)), silently skipping every remaining consumer. In practice the logger always resolves, so this is a latent edge rather than a live bug — but it's a good example of an early `return` hiding inside an error handler.

Consequence: a publish *never fails* from the caller's point of view. That's convenient for checkout and terrible for anything that must happen.

## Feature trace: what runs inside `PlaceOrderAsync` after payment succeeds

| Step | Line | Side effect | Failure mode |
| --- | --- | --- | --- |
| payment | [L1610-L1612](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1610-L1612) | external gateway call (plugin) | exception → caught at L1661 |
| save order | [L1616-L1618](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1616-L1618) | DB insert | after this point, a failure leaves a paid, persisted order with partial follow-ups |
| move cart → order items, discount & gift-card history, recurring payment | [L1621-L1631](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1621-L1631) | DB writes | separate statements, no transaction |
| notes + emails | [L1634](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1634) → [SendNotificationsAndSaveNotesAsync#L824-L850](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L824-L850) | order notes, **enqueue** emails ([WorkflowMessageService.cs#L2894](../../src/Libraries/Nop.Services/Messages/WorkflowMessageService.cs#L2894)), optionally render a PDF invoice to disk (L836-L837) | file I/O and template rendering on the checkout thread |
| reset checkout, activity log | [L1637-L1641](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1637-L1641) | DB writes | — |
| `OrderPlacedEvent` | [L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1644) | every consumer, synchronously | swallowed + logged |
| status checks, paid processing | [L1647-L1650](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1647-L1650) | more writes, possibly more events | — |

Six consumers of `OrderPlacedEvent` ship in this repo's plugins (found with `grep -rl "IConsumer<OrderPlacedEvent>" src`): Brevo, Omnisend, Payments.Manual, Tax.Avalara, Widgets.FacebookPixel, Widgets.GoogleAnalytics. Look at the Avalara one: on `OrderPlacedEvent` it calls `_avalaraTaxManager.CreateOrderTaxTransactionAsync(order)` ([Avalara EventConsumer.cs#L212-L224](../../src/Plugins/Nop.Plugin.Tax.Avalara/Services/EventConsumer.cs#L212-L224)) — an **external HTTP call to a tax service, inside the customer's checkout request**, whose failure the publisher will swallow. The customer gets a confirmation page; the tax transaction may simply not exist.

## The core concepts, in this repo's terms

**Idempotency** — safe to run twice. The order's `OrderGuid` must be set before `PlaceOrderAsync` runs ([L1598-L1599](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598-L1599) throws if it is empty) so payment processing has a stable key to dedupe on; the email send is *not* idempotent (crash between SMTP success and persisting `SentOnUtc` ⇒ duplicate email — trace it in [QueuedMessagesSendTask.cs#L60-L87](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L60-L87)). Duplicate email: acceptable. Duplicate charge: not. Same pattern, different budgets.

**Retries and their contract** — retry only what is idempotent or where duplicates are tolerable; always bound them (`maxTries = 3`, [L43](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43); the query filters `SentTries < maxSendTries` and respects `DontSendBeforeDateUtc`, [QueuedEmailService.cs#L162-L175](../../src/Libraries/Nop.Services/Messages/QueuedEmailService.cs#L162-L175)); know what happens *after* the last retry (here: the row sits silently — the observability gap flagged in [ticket 6](../06-contribution-practice/01-good-first-tickets.md)).

**Outbox** — persist intent in the same database as the business change, deliver asynchronously. The queued-email table is a textbook outbox *shape*. The caveat is now verified rather than hypothetical: the enqueue (`InsertQueuedEmailAsync`) and the order insert are separate statements with no shared transaction (see the trace above and [02-data-model](02-data-model-and-persistence.md)), so a crash between them can lose the intent. It's an outbox for *delivery* durability, not for *atomicity with the business write*.

**Compensation** — undoing an irreversible step by a counter-action, not rollback. Payment refund is compensation; `PlaceOrderAsync`'s catch block ([L1661-L1665](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1661-L1665)) logs and reports but does **not** auto-refund a captured payment — a human + support tooling is the compensation mechanism. Saying "sometimes the correct compensation is a human" is a legitimate senior answer.

**Backpressure** — bounding work intake: per-account email batch cap ([MaxNumberOfEmails, L51](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L51)); run-once-per-period guard on tasks ([ScheduleTaskRunner.cs#L122-L132](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L122-L132)). But note the email *query* is not bounded: it loads up to `int.MaxValue` rows and groups them in memory before the per-account `Take` ([L44-L51](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L44-L51)).

**Failure visibility** — every mechanism above fails *quietly* by design (logged, not alerted): event consumers swallow into the log; exhausted emails just stop; distributed-cache writes aren't awaited. Quiet failure is a choice with a cost curve: fine at 10 orders/day, ruinous at 10k. "What's the alert story?" is the question to ask of any async design.

## How background work actually runs

1. At startup `TaskScheduler.InitializeAsync` loads all tasks and builds a URL from the **current store's URL** + `scheduletask/runtask` ([TaskScheduler.cs#L59-L67](../../src/Libraries/Nop.Services/ScheduleTasks/TaskScheduler.cs#L59-L67)), computing each task's first delay from `LastStartUtc` (L80-L108).
2. A `PeriodicTimer` per task fires and **POSTs `taskType` to that URL over HTTP** ([TaskScheduler.cs#L200-L206](../../src/Libraries/Nop.Services/ScheduleTasks/TaskScheduler.cs#L200-L206)). Every node in a farm runs its own timers.
3. The receiving controller is public, inherits plain `Controller`, and carries `[IgnoreAntiforgeryToken]` ([ScheduleTaskController.cs#L21-L33](../../src/Presentation/Nop.Web/Controllers/ScheduleTaskController.cs#L21-L33)); the work context treats that path as the background-task system user ([WebWorkContext.cs#L217-L223](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L217-L223)).
4. `ScheduleTaskRunner.ExecuteAsync` applies the run-once-per-period guard, then runs the task inside `_locker.PerformActionWithLockAsync(scheduleTask.Type, expiration, …)` with `expiration = min(Seconds, 300) − 1` seconds ([ScheduleTaskRunner.cs#L134-L141](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L134-L141)).
5. On failure: the task is disabled if `StopOnError`, `LastEndUtc` is written, and an error is logged ([L143-L159](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L143-L159)).

Because step 3 is an unauthenticated endpoint, anyone can POST a task type. The guard in step 4 limits this to "run a task that is already due", so the exposure is small — but it is worth knowing that the *scheduler's* only authentication is time.

## Exercise ladder

1. **Goal:** enumerate the synchronous work in checkout.
   **Check:** `grep -rl "IConsumer<OrderPlacedEvent>" src` → 6 files. For each, open `HandleEventAsync(OrderPlacedEvent …)` and classify: local DB write, external HTTP call, or no-op unless configured.
2. **Goal:** prove the publisher never throws to its caller.
   **Check:** point to the `try/catch` at [EventPublisher.cs#L27-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L27-L50) and the nested `catch { }` at L46-L49.
3. **Goal:** compute a task's lock lifetime.
   **Check:** for a task with `Seconds = 60`, the lock expires after 59 s; for `Seconds = 3600`, after 299 s ([ScheduleTaskRunner.cs#L137](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L137)). Say what happens if the hourly task takes 10 minutes (see F1).
4. **Goal (mid):** classify each side effect of `PlaceOrderAsync` by the three questions.
   **Check:** a table with one row per step in the trace above, columns *fails? / runs twice? / how would we know?*, each answer citing a line.
5. **Goal (mid) — run locally:** observe a swallowed consumer failure.
   **Check (run locally; expected outcome derived from code):** in a dev build, make any `IConsumer<OrderPlacedEvent>` throw; place an order. Expected: the order succeeds and the confirmation page renders; the only trace is an error entry in the system log written at [EventPublisher.cs#L44](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L44).

## Senior findings (verified in source)

**F1 — The distributed task lock is not a lock.** `DistributedCacheLocker.PerformActionWithLockAsync` checks for the key with `GetStringAsync`, returns if present, otherwise `SetStringAsync`s it with an absolute expiration, runs the action, and in `finally` removes the key ([DistributedCacheLocker.cs#L33-L55](../../src/Libraries/Nop.Core/Caching/DistributedCacheLocker.cs#L33-L55)). Three problems, each read directly from those lines: (1) check-then-set is not atomic — two nodes can both see "absent" and both run the task; (2) the key expires after at most 299 seconds ([ScheduleTaskRunner.cs#L137](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L137)) even if the task is still running, after which another node can start it; (3) the `finally` deletes the key unconditionally, so a slow first runner can delete the *second* runner's lock. Combined with every node running its own timers (step 2 above), long tasks in a multi-node farm can overlap. The standard fix is an atomic set-if-absent with a per-holder token, checked on release. Confidence: high on mechanism; whether overlaps occur depends on task duration and node count.

**F2 — Distributed cache writes are fire-and-forget.** In `DistributedCacheManager.GetAsync`, the `SetStringAsync` to Redis/SQL Server is started but never awaited; only a continuation that removes the in-flight entry is attached ([DistributedCacheManager.cs#L182-L194](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L182-L194)). An exception from the cache server becomes an unobserved task exception: the request succeeds, nothing is cached remotely, and nothing is logged by this code. Symptom in production: "Redis hit rate is mysteriously low" with no errors anywhere.

**F3 — Cache invalidation inherits the publisher's swallow-and-continue.** Cache eviction runs as an event consumer ([CacheEventConsumer.cs#L13-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L13-L53)). If eviction throws (e.g. a transient Redis error during `RemoveByPrefixAsync`), the publisher logs and moves on, so the write is committed but the stale entry stays until it expires (60 minutes by default). Stale reads after a successful write, with only a log line as evidence, is the classic symptom.

## Side effects in risky places (flagged, not condemned)

1. **Synchronous event consumers inside checkout** — any plugin's `IConsumer<OrderPlacedEvent>` adds latency and (logged-only) failure modes to the purchase moment; Avalara's is an external call.
2. **Payment capture inside the request, before order persistence** — the gap analyzed in [key flow 4](../01-codebase-cartography/05-key-flows.md).
3. **Writes on GET** (recently-viewed, activity log — [ProductController.cs#L181-L195](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L181-L195)) and guest-row creation for cookieless visitors ([WebWorkContext.cs#L270-L274](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L270-L274)) — break GET-safety assumptions of caches/prefetchers.

## Debugging scenario: "Avalara shows no transaction for some orders"

1. **Confirm the order exists and is paid** — the order insert happens before the event ([L1616](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1616) vs [L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1644)), so "order exists, transaction missing" is exactly the shape a swallowed consumer failure produces.
2. **Search the system log** around the order's timestamp for an error whose message came from the Avalara client — the publisher logs `exception.Message` ([EventPublisher.cs#L44](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L44)).
3. **Check the consumer's own gate**: it returns early if the Avalara provider is not active for that customer ([Avalara EventConsumer.cs#L218-L220](../../src/Plugins/Nop.Plugin.Tax.Avalara/Services/EventConsumer.cs#L218-L220)) — a per-customer/per-store configuration difference produces "some orders" without any error.
4. **Remediate:** re-create the transactions for the affected orders (operational), then propose moving the call off the request path (an outbox row + scheduled task, mirroring the queued-email pattern) so failures become retryable and visible.

Drill: choose any `IConsumer<OrderPlacedEvent>` implementation. Answer the three questions — fails? runs twice? how do we know? — with line evidence. Basic: answers for the happy path. Solid: precise failure semantics from `EventPublisher`. Strong: proposes which of the three side-effect tiers (sync event / outbox / external) this consumer *should* use and what the migration costs.

## Verification

2026-10-06: every anchor opened and checked. The outbox caveat ("check whether enqueue and business write share a transaction") was resolved by reading `SendNotificationsAndSaveNotesAsync` and `WorkflowMessageService` L2894: they do not. Consumer count from grep (6 files). F1–F3 are read from source; overlapping task runs (F1) and unobserved exceptions (F2) were not reproduced.
