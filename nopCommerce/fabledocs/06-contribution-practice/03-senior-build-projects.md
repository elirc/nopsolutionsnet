# Senior Build Projects

Six projects, 2 days–4 weeks. Each has enough plan structure to start and enough open questions to make you think. These are realistic — several align with the maintainers' own risk register (compare [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md)).

## P1: Idempotent order placement by database constraint (≈1 week)
Problem: double-order protection is single-node ([the mutex block](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718)).
Product value: correctness in load-balanced deployments; removes sync-over-async from checkout.
Architecture decisions: unique index on order idempotency key (`OrderGuid`-derived); on violation → fetch and return existing result; keep the setting-gated mutex for back-compat one release.
Likely files: migration (new folder-versioned), [OrderProcessingService.cs#L1594-L1719](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594-L1719), order service, tests.
Migration plan: additive index (expand-only); dual-guard release; mutex removal later.
Test plan: concurrent placement test with same guid → exactly one order (finally testable!); regression on normal flow.
Security/perf: none/one index write. Rollout: setting-flagged. Rollback: drop nothing; disable flag.
Open questions: what exactly seeds `OrderGuid` per retry — must retries share it? (Read `ProcessPaymentRequest` lifecycle first.)
Stretch: reconciliation report of payment-captured-but-orderless events.
ISP: distributed-systems correctness story with measurable before/after.

## P2: Durable event consumers (outbox tier for domain events) (2–4 weeks)
Problem: `OrderPlacedEvent` consumers are sync, in-process, non-durable ([EventPublisher](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52)).
Value: plugins can do slow/critical reactions without risking checkout latency or losing events on crash.
Decisions: new `IDurableConsumer<T>` marker; publisher writes envelope rows for durable subscribers (same DB transaction question as email!); drain via schedule task; at-least-once + consumer idempotency documented.
Likely files: Core events, new table + builder + migration, publisher routing, task, admin grid for the envelope table.
Test plan: crash-simulation (kill between publish and drain), duplicate-delivery consumer tests.
Rollout: opt-in per consumer; zero change to existing ones. Rollback: consumers revert to sync interface.
Open questions: ordering guarantees? per-aggregate serialization needed for reward points?
ISP: the best design-conversation generator in this list.

## P3: Observability starter kit (≈1 week)
Problem: signals exist, notifications don't ([observability map](../05-quality-engineering/06-observability-and-operations.md)).
Value: operators learn about dead emails/stale tasks/error spikes without opening grids.
Decisions: in-app "system health" service aggregating: exhausted queued emails, stale schedule tasks, error-log rate; surfaced as admin dashboard warnings + optional notification email; thresholds as settings.
Likely files: new service in Nop.Services, admin factory/widget, settings, tests.
Rollout/rollback: pure addition/simple removal.
Open questions: does the admin dashboard already have a warnings panel to extend? (Investigate `Nop.Web/Areas/Admin` common models first.)
ISP: "I turned passive logs into active signals" — ops maturity story.

## P4: Typed product-attribute facade (2–3 weeks, the debt-management project)
Problem: `attributesXml` stringly-typed contract ([critique risk 7](../03-architecture-and-patterns/06-architecture-critique.md)).
Value: stop new defects without breaking the ecosystem.
Decisions: `ProductAttributeSelection` value type wrapping parse/serialize (the parser service already centralizes grammar — wrap, don't replace); new overloads accept the type, old signatures delegate; XML remains the storage format (explicitly out of scope to change).
Likely files: `Nop.Services/Attributes` + `Catalog` parsers, `ShoppingCartService` overloads, zero schema changes.
Test plan: round-trip property tests XML↔typed; equivalence tests old vs new paths.
Rollout: additive API; deprecation notes only. Rollback: unused overloads are harmless.
Open questions: where's the 80/20 — which 5 call sites produce most attribute bugs? (Instrument first.)
ISP: "I reduced a debt's interest rate without paying off the principal — deliberately."

## P5: Storefront REST endpoints for headless product browsing (2–4 weeks)
Problem/value: modern storefront experiments need JSON; core is MVC-only (API exists only as a plugin).
Decisions: build as a **plugin** (follow [WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend)'s architecture — that choice itself follows the repo's grain); read-only v1 (catalog + search); DTOs separate from view models *and* entities; the three visibility gates ([key flow 1](../01-codebase-cartography/05-key-flows.md)) enforced in one shared query layer.
Test plan: contract tests per endpoint; authorization tests per gate (cross-store leakage!).
Security plan: rate limiting at proxy; no exact stock counts ([review kata 4](../04-code-reading-gym/04-review-katas.md)).
Rollout: plugin install/uninstall is the flag. Open questions: pagination cursor vs offset; cache headers.
ISP: API design + tenancy enforcement story, framed inside plugin architecture.

## P6: Test-suite speed and isolation overhaul (≈2 weeks)
Problem: shared SQLite world → mutation leakage risk and slow cold start ([testing strategy](../05-quality-engineering/01-testing-strategy.md)).
Value: faster CI, safer parallelism, easier onboarding.
Decisions to evaluate (measure, then pick): snapshot/restore the SQLite file per fixture; in-memory SQLite per collection; or transaction-rollback-per-test. Introduce `TimeProvider` seam for the `DateTime.UtcNow` call sites ([lockout](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154), [availability](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L78)) as a sub-project.
Test plan for the test change: run the suite 20x, compare flake and duration distributions.
Rollback: base-class change is centralized in [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98) — one file to revert.
Open questions: how much of `Init()`'s cost is the installation service vs migrations? Profile before choosing.
ISP: "I improved the *system that produces quality* rather than one feature" — a distinctly senior story.
