# Verification Log

Running log of what was inspected while building this curriculum. Every anchor in these docs was confirmed against the working tree on the date shown. Line numbers drift as the repo evolves — if an anchor looks wrong, search for the symbol name instead.

**Repo state at time of writing:** branch `develop`, HEAD `08720d3f05` ("#8254 Move GetDefaultCulture method from views to NopHtmlHelper"), clean working tree, 2026-07-16.

## Environment facts (verified)

| Fact | Evidence |
| --- | --- |
| Target framework `net10.0` | `src/Directory.Build.props` |
| SDK pinned to `10.0.100`, rollForward latestFeature | `global.json` |
| CI = GitHub Actions, windows-latest: `dotnet restore src` → `dotnet build --configuration Release src` → `dotnet test` | `.github/workflows/dotnet.yml` |
| Solution: `src/NopCommerce.sln`; 3 libraries (Nop.Core, Nop.Data, Nop.Services), 2 presentation projects (Nop.Web, Nop.Web.Framework), 1 test project (Nop.Tests), 33 plugin projects | directory listing of `src/` |
| Mapster replaced AutoMapper recently | commit `99997cecd6` |

## Code inspected (all line refs verified on 2026-07-16)

| Area | File | What was read/confirmed |
| --- | --- | --- |
| Startup | `src/Presentation/Nop.Web/Program.cs` | Full file. WebApplicationBuilder, optional Autofac (`UseAutofac` setting), `ValidateScopes = false`, `ConfigureApplicationServices`, `ConfigureRequestPipeline` |
| DI | `src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs` | class at L67, ~200 `AddScoped` registrations (e.g. `IShoppingCartService` at L204), `Order => 2000` at L336 |
| Repository | `src/Libraries/Nop.Data/EntityRepository.cs` | class L14; `GetByIdAsync` L122; `GetAllAsync` overloads L226–L301; `GetAllPagedAsync` L302, L324; `InsertAsync` L340/L357; `UpdateAsync` L393; `DeleteAsync` L433/L460/L497; `Table` L525 |
| Repository contract | `src/Libraries/Nop.Data/IRepository.cs` | method list incl. `publishEvent` flags, cache-key delegates |
| Product page | `src/Presentation/Nop.Web/Controllers/ProductController.cs` | `ProductDetails` L129–L203: ACL check L139, store-mapping check L141, admin-preview bypass L146–L148, activity log L194, factory call L198 |
| Product factory | `src/Presentation/Nop.Web/Factories/ProductModelFactory.cs` | `PrepareProductDetailsModelAsync` L1502; recursive call for associated products L1727; file is 2024 lines |
| Add to cart (web) | `src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs` | `AddProductToCart_Catalog` L593; `AddProductToCart_Details` L791–L874 (form parsing, attribute parsing, SaveItemAsync); `ProductDetails_AttributeChange` L879 |
| Add to cart (service) | `src/Libraries/Nop.Services/Orders/ShoppingCartService.cs` | `AddToCartAsync` L1549–L1668: permission checks L1560–L1570, bot check L1572, warning-list validation, update-vs-insert branch; `GetShoppingCartAsync` L753; `GetShoppingCartItemWarningsAsync` L1124; file is 1976 lines |
| Place order | `src/Libraries/Nop.Services/Orders/OrderProcessingService.cs` | `PlaceOrderAsync` L1594–L1719: local function `placeOrder`, `OrderPlacedEvent` publish L1644, named-Mutex + sync-over-async block L1679–L1718 (`.Result` L1698/L1707, `.Wait()` L1710); `SaveOrderDetailsAsync` L723; payment call L1424; file is 3561 lines |
| Login | `src/Presentation/Nop.Web/Controllers/CustomerController.cs` | `Login` GET L442, POST L457–L553: CAPTCHA, OTP branch, `ValidateCustomerAsync` call, result switch, failed-login activity log |
| Credential validation | `src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs` | `ValidateCustomerAsync` L138–L194: existence/deleted/active/registered/lockout checks, failed-attempt counter + lockout write L160–L172, MFA branch L175–L181; `ValidateCustomerByPhoneAsync` L205–L228 |
| Permissions | `src/Libraries/Nop.Services/Security/PermissionService.cs` | `AuthorizeAsync` overloads L258, L272, L291, L305, L331 (role-based, cached per role) |
| Admin gate | `src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs` | sealed TypeFilterAttribute L11; permission check `ACCESS_ADMIN_PANEL` L89; `OnAuthorizationAsync` L103 |
| Action permission | `src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs` | attribute L16; `AuthorizeAsync` filter body L106–L119 |
| Admin controller | `src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs` | `[CheckPermission(...)]` usage on `List` L943–L944, `BulkEdit` L952–L953, `BulkEditSave` L962–L964 (view vs create/edit/delete permission split) |
| Multi-store | `src/Libraries/Nop.Services/Stores/StoreMappingService.cs` | `AuthorizeAsync<TEntity>` L204 and L221 (`IStoreMappingSupported` constraint) |
| Events | `src/Libraries/Nop.Services/Events/EventPublisher.cs` | full file: resolve-all-consumers loop, per-consumer try/catch, `IStopProcessingEvent` support |
| Cache invalidation | `src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs` | L13–L75: consumes Insert/Update/Delete entity events, prefix-based eviction |
| Cache manager | `src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs` | class L13 implements `IStaticCacheManager`; `GetAsync(key, acquire)` L132 |
| Background task | `src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs` | full file: outbox-style queued email send, `maxTries = 3`, per-email try/catch/finally with `SentTries` increment |
| Task runner | `src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs` | `ExecuteAsync(ScheduleTask, ...)` L115; `task.ExecuteAsync()` call L74 |
| Migrations | `src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs` | `[NopSchemaMigration("2026-03-23 00:00:00", "Price lists")]` L7, `ForwardOnlyMigration`, `CreateTableIfNotExists<T>()` calls; version folders UpgradeTo440–UpgradeTo500 exist |
| Schema builders | `src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs` | `NopEntityBuilder<Product>` L9, `MapEntity` L17 |
| Domain entity | `src/Libraries/Nop.Core/Domain/Catalog/Product.cs` | `Product : BaseEntity, ILocalizedEntity, ISlugSupported, IAclSupported, IStoreMappingSupported, IDiscountSupported<>, ISoftDeletedEntity, IMetaTagsSupported` L13; 606 lines |
| Validation | `src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs` | `BaseNopValidator<LoginModel>` L9, `RuleFor` chains L16–L34 |
| Work context | `src/Presentation/Nop.Web.Framework/WebWorkContext.cs` | `GetCurrentCustomerAsync` L196 |
| Tests | `src/Tests/Nop.Tests/BaseNopTest.cs` | class L98; `Init()` L109 uses SQLite provider + real installation service; `GetService<T>` L572; 854 lines |
| Tests | `src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs` | `ProductServiceTests : ServiceTest` L9; NUnit `[Test]` methods from L63 |
| Frontend build | `src/Presentation/Nop.Web/package.json` | gulp 4 devDependencies; bootstrap 4, jQuery-era admin stack (AdminLTE 3, DataTables) |

## Commands run

| Command | Result |
| --- | --- |
| `git log --oneline -3`, `git status` | clean tree at `08720d3f05` |
| Directory listings of `src/`, `Libraries`, `Presentation`, `Tests`, `Plugins`, `Migrations` | recorded above |
| Various `grep -n` symbol searches | used to pin line numbers cited above |
| `dotnet build` / `dotnet test` | **NOT run** (build/test commands in these docs are marked *inferred* from CI config and README, not verified locally) |

## Late verifications (during writing)

- Confirmed test-file inventory: `Nop.Services.Tests/Orders` has `OrderProcessingServiceTests` etc. but **no `ShoppingCartServiceTests`**; `Nop.Services.Tests/Security` holds only `EncryptionServiceTests` (**no `PermissionService` tests**); `Nop.Services.Tests/Customers` has `CustomerRegistrationServiceTests` ✓; `Nop.Services.Tests/Messages` has `QueuedEmailServiceTests` etc. Key-flows and writing-tests docs were corrected accordingly; gaps recorded as risk R11 and contribution tickets.
- `ShoppingCartModelFactoryTests` exists at `src/Tests/Nop.Tests/Nop.Web.Tests/Public/Factories/`.
- No `PagedList` test file found (search by name) — ticket 12 stands as written ("investigate" label retained).

## Known uncertainties

- Build and test commands were **not executed** in this session; they mirror `.github/workflows/dotnet.yml` exactly and are labeled *inferred* wherever cited.
- Claims about runtime behavior (cache hit rates, request pipeline ordering beyond what code shows) are labeled as inference in the docs.
- The `Mutex`/sync-over-async block in `PlaceOrderAsync` is flagged as a **risk/discussion point**, not a confirmed bug — it is intentional per its own comments (L1685–L1688).
- Plugin projects were surveyed by name only; individual plugin internals were not read except where cited.
- `docs/` conventions: repo has no existing docs folder for learning material; `fabledocs/` was created per user request.
