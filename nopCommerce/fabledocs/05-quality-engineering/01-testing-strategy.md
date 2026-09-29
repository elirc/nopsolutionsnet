# Testing Strategy

## What this suite believes

One test project ([src/Tests/Nop.Tests](../../src/Tests/Nop.Tests)) mirroring the libraries (`Nop.Core.Tests`, `Nop.Data.Tests`, `Nop.Services.Tests`, `Nop.Web.Tests`), NUnit, run wholesale by CI ([dotnet.yml](../../.github/workflows/dotnet.yml)). The defining choice lives in [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98): tests boot a **real DI container** with a **real SQLite database**, initialized by the **actual installation service** with seed data (L109-L116), then resolve real services via `GetService<T>()` (L572). SMTP is faked ([TestSmtpBuilder](../../src/Tests/Nop.Tests/TestSmtpBuilder.cs)), the data provider is swapped ([SqLiteNopDataProvider](../../src/Tests/Nop.Tests/SqLiteNopDataProvider.cs)) — nearly everything else is production wiring.

So the strategy is **sociable integration tests at the service layer**, not isolated unit tests. Know the tradeoff cold, because "unit vs integration" is a guaranteed interview topic:

| | Mock-isolated units | This repo's sociable tests |
| --- | --- | --- |
| Catches | logic bugs in one class | wiring, SQL, cross-service, event/cache bugs |
| Misses | integration seams | fine-grained branch logic buried in helpers |
| Refactor cost | high (mocks restate implementation) | low (tests care about behavior) |
| Speed | ms | slow start (full install), then OK |
| Failure diagnosis | precise | broader — which layer broke? |

The mid-level answer isn't picking a side; it's matching strategy to codebase: heavy cross-service orchestration + one team + stable interfaces → sociable tests win here. A parsing library would choose the opposite.

## What belongs at each layer (here)

- **Pure logic, no I/O** → plain NUnit against the class: see the availability tests in [ProductServiceTests.cs#L63-L156](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63-L156) — no DB needed for date math.
- **Business rules touching data** → service test on the SQLite world (most of `Nop.Services.Tests`).
- **Schema/migrations** → `Nop.Data.Tests`.
- **What not to test:** controllers thin enough to be orchestration-only (their value is covered by service tests + the fact that factories/services are exercised); generated/seeded localization content; third-party plugin internals.

## Fixtures, state, and flake prevention

The SQLite world is **shared across tests** — mutations leak. The suite's convention: mutate → restore in teardown (see the SetUp/TearDown pair in [ProductServiceTests.cs#L20-L47](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L20)). When you write tests here: prefer read-only assertions against seed data; if you mutate, you own the restore; never depend on test execution order.

Time and randomness: code under test calls `DateTime.UtcNow` directly (e.g. [lockout logic](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154)) — there is no clock abstraction, so time-sensitive tests must construct state relative to now (as the availability tests do) rather than freezing time. Cite this as a known limitation and what an `ITimeProvider`/.NET `TimeProvider` refactor would buy — good RFC-kata material.

## Gaps to know (candidate contributions)

Direct coverage is thin for: admin controllers, the schedule-task classes ([QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) — *investigate*), and concurrency behaviors (the mutex block is untestable in-process by design). Each gap is a ticket in [06-contribution-practice](../06-contribution-practice/01-good-first-tickets.md).

Interview angle: "How do you test a legacy system?" — describe this exact architecture: stand up a real-but-fast world (SQLite), test at the service seam, reserve unit isolation for pure logic, and burn down flake sources (shared state, real time) by convention. That's a production-grade answer.
