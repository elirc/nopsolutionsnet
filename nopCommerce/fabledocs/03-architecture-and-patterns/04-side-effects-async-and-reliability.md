# Side Effects, Async Work, and Reliability

A **side effect** is anything a flow changes beyond its return value — rows, emails, caches, external APIs. Reliability engineering is the discipline of asking, for each one: *what if it fails? what if it runs twice? how would we know?* (Those three questions, verbatim, are a complete interview framework.)

## Map of side-effect machinery

| Mechanism | Sync? | Durable? | Retries? | Anchor |
| --- | --- | --- | --- | --- |
| Entity events → consumers | sync, in-process | no | no | [EventPublisher.cs#L20-L52](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52) |
| Domain events (`OrderPlacedEvent` etc.) | sync, in-process | no | no | [publish at OrderProcessingService.cs#L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1644) |
| Queued email (outbox) | async via scheduler | **yes** (DB rows) | bounded (3) | [QueuedMessagesSendTask.cs#L41-L90](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L41-L90) |
| Schedule tasks | async, in-process | schedule in DB | per-run | [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| External APIs (payment, shipping, tax) | sync in request | n/a | plugin-specific | [ProcessPaymentAsync call, L1424](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1424) |
| Cache eviction | sync via events | no | no | [CacheEventConsumer.cs#L44-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) |

Read the table as a decision record: **durable + retried** was spent only on email. Payment gets synchronous handling with idempotency hints; cache eviction gets best-effort. Matching *reliability budget* to *consequence of loss* is the senior skill — not making everything durable.

## The core concepts, in this repo's terms

**Idempotency** — safe to run twice. The order's `OrderGuid` ([L1598](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598)) exists so payment processing can dedupe; the email send is *not* idempotent (crash between SMTP success and persisting `SentOnUtc` ⇒ duplicate email — trace it in [QueuedMessagesSendTask.cs#L60-L87](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L60-L87)). Duplicate email: acceptable. Duplicate charge: not. Same pattern, different budgets.

**Retries and their contract** — retry only what is idempotent or where duplicates are tolerable; always bound them (`maxTries = 3`); know what happens *after* the last retry (here: row sits silently — the observability gap flagged in [ticket 6](../06-contribution-practice/01-good-first-tickets.md)).

**Outbox** — persist intent in the same database as the business change, deliver asynchronously. The queued-email table is a textbook outbox. *Investigate-level caveat:* delivery is decoupled, but check whether enqueue and business write share a transaction — if not, a crash can still lose the intent (see [02-data-model](02-data-model-and-persistence.md) on transaction scope).

**Compensation** — undoing an irreversible step by a counter-action, not rollback. Payment refund is compensation; `PlaceOrderAsync`'s catch block ([L1661-L1665](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1661-L1665)) logs and reports but does **not** auto-refund a captured payment — a human + support tooling is the compensation mechanism. Saying "sometimes the correct compensation is a human" is a legitimate senior answer.

**Backpressure** — bounding work intake: per-account email batch cap ([MaxNumberOfEmails, L51](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L51)); run-once-per-period guard on tasks ([ScheduleTaskRunner#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115)).

**Failure visibility** — every mechanism above fails *quietly* by design (logged, not alerted): event consumers swallow into the log ([EventPublisher#L35-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L35-L50)); exhausted emails just stop. Quiet failure is a choice with a cost curve: fine at 10 orders/day, ruinous at 10k. "What's the alert story?" is the question to ask of any async design.

## Side effects in risky places (flagged, not condemned)

1. **Synchronous event consumers inside checkout** — any plugin's `IConsumer<OrderPlacedEvent>` adds latency and (logged-only) failure modes to the purchase moment.
2. **Payment capture inside the request, before order persistence** — the gap analyzed in [key flow 4](../01-codebase-cartography/05-key-flows.md).
3. **Writes on GET** (recently-viewed, activity log — [ProductController.cs#L181-L195](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L181-L195)) — breaks GET-safety assumptions of caches/prefetchers.

Drill: choose any `IConsumer<OrderPlacedEvent>` implementation (grep for it). Answer the three questions — fails? runs twice? how do we know? — with line evidence. Basic: answers for the happy path. Solid: precise failure semantics from `EventPublisher`. Strong: proposes which of the three side-effect tiers (sync event / outbox / external) this consumer *should* use and what the migration costs.
