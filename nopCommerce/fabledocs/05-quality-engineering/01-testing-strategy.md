# Testing Strategy

What the suite believes, how it is built, what it actually covers (counted, not estimated), where it quietly overstates coverage, and how to choose the right layer for a new test.

## What this suite believes

One test project ([src/Tests/Nop.Tests](../../src/Tests/Nop.Tests)) mirroring the libraries (`Nop.Core.Tests`, `Nop.Data.Tests`, `Nop.Services.Tests`, `Nop.Web.Tests`), NUnit, run wholesale by CI ([dotnet.yml](../../.github/workflows/dotnet.yml)). The defining choice lives in [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98): tests boot a **real DI container** with a **real SQLite database**, initialized by the **actual installation service** with sample data ([L109-L135](../../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L135)), then resolve real services via `GetService<T>()` ([L572-L575](../../src/Tests/Nop.Tests/BaseNopTest.cs#L572-L575)). SMTP is faked ([TestSmtpBuilder](../../src/Tests/Nop.Tests/TestSmtpBuilder.cs)), the data provider is swapped ([SqLiteNopDataProvider](../../src/Tests/Nop.Tests/SqLiteNopDataProvider.cs)), HTTP state is mocked (`IHttpContextAccessor` returning a `DefaultHttpContext`, [L234-L241](../../src/Tests/Nop.Tests/BaseNopTest.cs#L234-L241)) — nearly everything else is production wiring.

So the strategy is **sociable integration tests at the service layer**, not isolated unit tests. Know the tradeoff cold, because "unit vs integration" is a guaranteed interview topic:

| | Mock-isolated units | This repo's sociable tests |
| --- | --- | --- |
| Catches | logic bugs in one class | wiring, SQL, cross-service, event/cache bugs |
| Misses | integration seams | fine-grained branch logic buried in helpers |
| Refactor cost | high (mocks restate implementation) | low (tests care about behavior) |
| Speed | ms | slow start (full install), then OK |
| Failure diagnosis | precise | broader — which layer broke? |

The mid-level answer isn't picking a side; it's matching strategy to codebase: heavy cross-service orchestration + one team + stable interfaces → sociable tests win here. A parsing library would choose the opposite.

## How the test world is built (verified mechanics)

- **Once per process.** The container is built in a *static constructor* ([BaseNopTest.cs#L162-L570](../../src/Tests/Nop.Tests/BaseNopTest.cs#L162-L570)) and stored in a `static readonly ServiceProvider` (L100). Every fixture in the run shares one container and one database.
- **The static engine is replaced** so production code that uses `EngineContext.Current` (event publisher, cache consumers) resolves from the test container: `EngineContext.Replace(new NopTestEngine(_serviceProvider))` ([L567](../../src/Tests/Nop.Tests/BaseNopTest.cs#L567)).
- **The database is in-memory SQLite with a shared cache**: `Data Source=nopCommerceTest.sqlite;Mode=Memory;Cache=Shared` ([NopTestConfiguration.cs#L26](../../src/Tests/Nop.Tests/NopTestConfiguration.cs#L26)).
- **Install runs for real**, including sample data, permissions and themes ([L113-L135](../../src/Tests/Nop.Tests/BaseNopTest.cs#L113-L135)) — note the `.Wait()` calls: sync-over-async is acceptable in a static constructor that can't be async.
- **Plugins are faked by descriptor.** `ServiceTest` registers test plugins such as a fixed-rate tax provider ([ServiceTest.cs#L30-L49](../../src/Tests/Nop.Tests/Nop.Services.Tests/ServiceTest.cs#L30-L49)), so tax/shipping/payment code paths have something to call.

## What exists, counted

Counted on 2026-10-06 with `find … -name "*Tests.cs" | wc -l` and `grep -rE "^\s*\[Test\]" | wc -l` per folder:

| Folder | Test files | `[Test]` attributes |
| --- | --- | --- |
| `Nop.Core.Tests` | 15 | 87 |
| `Nop.Data.Tests` | 2 | 26 |
| `Nop.Services.Tests` | 62 | 388 |
| `Nop.Web.Tests` | 46 | 431 |
| **total** | **125** | **932** |

Plus 200 `[TestCase(...)]` lines (parameterized cases). `Nop.Web.Tests` is mostly validators and model factories (e.g. 11 files under `Admin/Validators/Catalog`, 13 under `Public/Factories`) — not controllers.

Confirmed gaps (absence verified by listing): no `ShoppingCartServiceTests` file in `Nop.Services.Tests/Orders`; `Nop.Services.Tests/Security` holds only `EncryptionServiceTests` (no `PermissionService`/`AclService` tests); `CustomerRegistrationServiceTests` has no lockout test (`grep -ci lock` → 0). Correction to an earlier claim: schedule tasks are *not* untested — [ScheduleTaskRunnerTests.cs](../../src/Tests/Nop.Tests/Nop.Services.Tests/ScheduleTasks/ScheduleTaskRunnerTests.cs) has 5 tests covering enabled/disabled/force-run/throw/run-once-per-period; what is untested is the individual task classes such as `QueuedMessagesSendTask`.

## What belongs at each layer (here)

- **Pure logic, no I/O** → plain NUnit against the class: see the availability tests in [ProductServiceTests.cs#L63-L156](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63-L156) — no DB needed for date math.
- **Business rules touching data** → service test on the SQLite world (most of `Nop.Services.Tests`).
- **Repository/provider semantics** → `Nop.Data.Tests`. These double as executable documentation: [EntityRepositoryTests.cs#L49-L74](../../src/Tests/Nop.Tests/Nop.Data.Tests/EntityRepositoryTests.cs#L49-L74) asserts that `GetByIdAsync` returns a soft-deleted product by default and returns `null` only with `includeDeleted: false`.
- **Validators and model factories** → `Nop.Web.Tests` (e.g. [LoginValidatorTests.cs](../../src/Tests/Nop.Tests/Nop.Web.Tests/Public/Validators/Customer/LoginValidatorTests.cs)).
- **What not to test:** controllers thin enough to be orchestration-only; generated/seeded localization content; third-party plugin internals.

## Feature trace: one service test, end to end

[CustomerRegistrationServiceTests.EnsureOnlyRegisteredCustomersCanLogin](../../src/Tests/Nop.Tests/Nop.Services.Tests/Customers/CustomerRegistrationServiceTests.cs#L69-L81):

1. `[OneTimeSetUp]` resolves real services from the shared container ([L16-L22](../../src/Tests/Nop.Tests/Nop.Services.Tests/Customers/CustomerRegistrationServiceTests.cs#L16-L22)).
2. Asserts the seeded admin can log in (L72-L73) — relying on install-time seed data.
3. Creates a customer *without* the Registered role through the real `ICustomerService` (L75; helper at L24-L59).
4. Calls the real `ValidateCustomerAsync` → which hits the repository, the role mapping, the password table and the permission service ([CustomerRegistrationService.cs#L138-L152](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L152)).
5. **Cleans up before asserting** (L78 before L80) — so a failed assertion doesn't leak the test customer into the shared database. That ordering is a deliberate hygiene pattern worth copying; the residual risk is an *exception* in step 4, which would skip cleanup because there is no `try/finally`.

## Fixtures, state, and flake prevention

The SQLite world is **shared across all fixtures in the process** — mutations leak. The suite's convention: mutate → restore in teardown (see the `[OneTimeSetUp]`/`[OneTimeTearDown]` pair in [ProductServiceTests.cs#L19-L47](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L19-L47)). When you write tests here: prefer read-only assertions against seed data; if you mutate, you own the restore; never depend on test execution order.

Time and randomness: code under test calls `DateTime.UtcNow` directly (e.g. [lockout logic](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154)) — there is no clock abstraction, so time-sensitive tests must construct state relative to now (as the availability tests do) rather than freezing time. Cite this as a known limitation and what an `ITimeProvider`/.NET `TimeProvider` refactor would buy — good RFC-kata material.

## Exercise ladder

1. **Goal:** reproduce the inventory.
   **Check:** run the two counting commands per folder and match the table above (125 files / 932 `[Test]`).
2. **Goal:** explain why a test can see seed data.
   **Check:** cite `InstallSampleData = true` in [BaseNopTest.cs#L127](../../src/Tests/Nop.Tests/BaseNopTest.cs#L127) and the static constructor that runs it once.
3. **Goal:** find a test that documents a surprising production default.
   **Check:** read [EntityRepositoryTests.cs#L59-L66](../../src/Tests/Nop.Tests/Nop.Data.Tests/EntityRepositoryTests.cs#L59-L66) and state the default it pins down (`includeDeleted = true`).
4. **Goal (mid):** write the missing lockout test (plan only, or run locally).
   **Check:** plan must (a) load `CustomerSettings`, set `FailedPasswordAllowedAttempts = 2` and save, (b) create a registered customer with the existing helper, (c) fail twice, (d) assert `LockedOut` on the third attempt even with the correct password, (e) restore settings and delete the customer in `finally`.
5. **Goal (mid):** decide which layer a `ShoppingCartService` max-items test belongs in.
   **Check:** argue for a service test (needs a real cart, settings and the repository) and identify the setting (`ShoppingCartSettings.MaximumShoppingCartItems`) and the line under test ([ShoppingCartService.cs#L1637](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1637)).

## Senior finding: multi-database tests pass vacuously in CI

`Nop.Data.Tests` parameterizes its tests over four providers — `[TestCase(DataProviderType.Unknown)]`, `SqlServer`, `MySql`, `PostgreSQL` (e.g. [EntityRepositoryTests.cs#L44-L49](../../src/Tests/Nop.Tests/Nop.Data.Tests/EntityRepositoryTests.cs#L44-L49), [NopDataProviderTests.cs#L32-L37](../../src/Tests/Nop.Tests/Nop.Data.Tests/NopDataProviderTests.cs#L32-L37)). Each test begins `if (!SetDataProviderType(type)) return;` ([EntityRepositoryTests.cs#L51-L52](../../src/Tests/Nop.Tests/Nop.Data.Tests/EntityRepositoryTests.cs#L51-L52), [NopDataProviderTests.cs#L39-L40](../../src/Tests/Nop.Tests/Nop.Data.Tests/NopDataProviderTests.cs#L39-L40)), and `SetDataProviderType` returns `false` when the connection string is empty ([BaseNopTest.cs#L606](../../src/Tests/Nop.Tests/BaseNopTest.cs#L606)). The checked-in connection strings for SQL Server, MySQL and PostgreSQL **are empty** ([NopTestConfiguration.cs#L11-L21](../../src/Tests/Nop.Tests/NopTestConfiguration.cs#L11-L21)). Result: in CI, three of every four cases *return without asserting anything* and are reported as **passed**, not skipped or inconclusive. The production databases are therefore never exercised, while the test report shows green rows for them. Confidence: high (read directly). Low-cost fix: replace `return` with `Assert.Ignore("connection string not configured")` (NUnit), so the report says what actually happened; higher-value fix: a CI matrix with containerized databases.

## Debugging scenario: "tests pass alone, fail in the full run"

*Symptom:* `MyNewServiceTests` passes when run by itself, but fails in CI with an unexpected count or a uniqueness error.

1. **Remember the world is shared.** One static container and one in-memory database for the whole process (static constructor + `Cache=Shared`). Another fixture that ran earlier may have inserted rows or changed settings and not restored them.
2. **Look for missing restores.** Grep the suspect fixtures for `[OneTimeTearDown]`/`[TearDown]` and for settings saved without being reverted.
3. **Look for cache bleed.** The static cache is part of the shared container; a test that saved a setting or entity through a path that skips events (e.g. direct `INopDataProvider` writes) can leave stale cache entries for the next fixture.
4. **Make your test order-independent:** create the data you need with unique values (e.g. a GUID in the email), assert relative to what you created, and clean up in `finally`. Reproduce locally by running the full `Nop.Services.Tests` namespace rather than the single class (*inferred* command: `dotnet test src --filter "FullyQualifiedName~Nop.Tests.Nop.Services.Tests"`).

Interview angle: "How do you test a legacy system?" — describe this exact architecture: stand up a real-but-fast world (SQLite), test at the service seam, reserve unit isolation for pure logic, and burn down flake sources (shared state, real time) by convention. Then add the twist from the finding: *make skipped coverage visible* — a green test that didn't run is worse than a yellow one.

## Verification

2026-10-06: counts produced with `find`/`grep` as stated; corrected the earlier statement that schedule tasks lack tests (`ScheduleTaskRunnerTests` has 5). All anchors opened and checked. No tests were run; the vacuous-pass finding is derived from the code paths cited, not from a test report.
