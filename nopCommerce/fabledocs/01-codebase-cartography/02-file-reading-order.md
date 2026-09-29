# File Reading Order

28 files in dependency-aware order. Per file: why it matters, what to look for, what to skip. Files 1–12 are the junior path; 13–22 add the mid path; 23–28 are senior territory. Don't read linearly top-to-bottom in one sitting — pair each block with the matching drill.

## Junior path (1–12): "how does a request become a page?"

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [Program.cs](../../src/Presentation/Nop.Web/Program.cs) | entire boot in 50 lines; config from `App_Data`; Autofac optional | env-name formatting details |
| 2 | [NopStartup.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67) | one-to-one interface→class registrations, all scoped; `Order => 2000` | memorizing the list |
| 3 | [BaseEntity.cs](../../src/Libraries/Nop.Core/BaseEntity.cs) | every table row is `{ int Id }` + subclass | — |
| 4 | [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) | the capability-interface stack; entities are property bags, zero behavior | the 600 lines of properties |
| 5 | [IRepository.cs](../../src/Libraries/Nop.Data/IRepository.cs) | the whole data contract; note `publishEvent` flags and cache-key delegates | — |
| 6 | [EntityRepository.cs#L122-L455](../../src/Libraries/Nop.Data/EntityRepository.cs#L122) | cached reads; writes publish events | provider plumbing |
| 7 | [ProductController.cs#L129-L203](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129-L203) | the canonical action: gates → side effects → factory → view | other actions for now |
| 8 | [ProductModelFactory.cs#L1502-L1560](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502) | what "prepare a model" means; count the service calls | all 2,024 lines |
| 9 | [LoginValidator.cs](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9-L34) | FluentValidation at the edge; conditional rules | — |
| 10 | [ShoppingCartService.cs#L1549-L1668](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549) | warnings-list validation; update-vs-insert | the other 1,900 lines |
| 11 | [CacheEventConsumer.cs](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L13-L75) | write → event → eviction spine | — |
| 12 | [ProductServiceTests.cs#L63-L156](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63) | test naming and style you'll be expected to match | — |

## Mid path (13–22): "where are the boundaries and the money?"

| # | File | Why / look for |
| --- | --- | --- |
| 13 | [CustomerController.cs#L457-L553](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L457-L553) | auth flow orchestration; filter attributes on the action |
| 14 | [CustomerRegistrationService.cs#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) | lockout, MFA branch, check ordering |
| 15 | [PermissionService.cs#L258-L331](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258-L331) | role-based authz, cached per role |
| 16 | [CheckPermissionAttribute.cs](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16) + [AuthorizeAdminAttribute.cs](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89) | declarative gates; what they *can't* express (row ownership) |
| 17 | [StoreMappingService.cs#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221) | multi-store isolation as a generic capability |
| 18 | [OrderProcessingService.cs#L1594-L1719](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594-L1719) | the money path; read it twice |
| 19 | [QueuedMessagesSendTask.cs](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) | outbox drain; retries |
| 20 | [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) | how background work is guarded |
| 21 | [EventPublisher.cs](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52) | in-process pub/sub semantics: sync, swallowing, ordered |
| 22 | [PriceListMigration.cs](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7-L20) + [ProductBuilder.cs](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L9-L17) | schema evolution: dated, forward-only |

## Senior path (23–28): "what did they commit to?"

| # | File | Why / look for |
| --- | --- | --- |
| 23 | [ApplicationBuilderExtensions.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs) | the real request pipeline; middleware order is a contract |
| 24 | [WebWorkContext.cs#L196](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) | "current customer" resolution — ambient state and its costs |
| 25 | [MemoryCacheManager.cs#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) + [DistributedCacheManager.cs](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs) | single-node vs distributed cache seam |
| 26 | [BaseNopTest.cs#L98-L140](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98) | the test-world bootstrap: real DI + SQLite + full install |
| 27 | Any plugin end-to-end, e.g. [Nop.Plugin.Payments.Manual](../../src/Plugins/Nop.Plugin.Payments.Manual) | how the extension contract is consumed |
| 28 | [OrderProcessingService.cs#L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718) | re-read the mutex block *after* everything else; now critique it |

Pause-and-predict habit: before opening each file, write one sentence guessing what's inside from its name and location. Grade your guesses — calibration is the actual skill being trained.
