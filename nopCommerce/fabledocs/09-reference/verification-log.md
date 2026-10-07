# Verification Log

Running log of what was inspected while building this curriculum. Every anchor in these docs was confirmed against the working tree on the date shown. Line numbers drift as the repo evolves — if an anchor looks wrong, search for the symbol name instead.

**Repo state at time of writing:** branch `develop`, HEAD `08720d3f05` ("#8254 Move GetDefaultCulture method from views to NopHtmlHelper"), clean working tree, 2026-07-16.

## Environment facts (verified)

| Fact | Evidence |
| --- | --- |
| Target framework `net10.0` | `src/Directory.Build.props` |
| SDK pinned to `10.0.100`, rollForward latestFeature | `global.json` |
| CI = GitHub Actions, windows-latest: `dotnet restore src` → `dotnet build --configuration Release src` → `dotnet test` | `.github/workflows/dotnet.yml` |
| Solution: `src/NopCommerce.sln`; 3 libraries (Nop.Core, Nop.Data, Nop.Services), 2 presentation projects (Nop.Web, Nop.Web.Framework), 1 test project (Nop.Tests), 32 plugin projects (corrected 2026-10-06 from 33: the `.sln` has 33 `Plugin` matches but one is the `Plugins` solution folder) | directory listing of `src/` |
| Mapster replaced AutoMapper recently | commit `99997cecd6` |

## Code inspected (all line refs verified on 2026-07-16)

| Area | File | What was read/confirmed |
| --- | --- | --- |
| Startup | `src/Presentation/Nop.Web/Program.cs` | Full file. WebApplicationBuilder, optional Autofac (`UseAutofac` setting), `ValidateScopes = false`, `ConfigureApplicationServices`, `ConfigureRequestPipeline` |
| DI | `src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs` | class at L67, 158 `services.Add*` registrations (143 `AddScoped`, 13 `AddSingleton`, 2 `AddTransient`; corrected 2026-10-06 from "~200") (e.g. `IShoppingCartService` at L204), `Order => 2000` at L336 |
| Repository | `src/Libraries/Nop.Data/EntityRepository.cs` | class L14; `GetByIdAsync` L122; `GetAllAsync` overloads L226–L301; `GetAllPagedAsync` L302, L324; `InsertAsync` L340/L357; `UpdateAsync` L393; `DeleteAsync` L433/L460/L497; `Table` L525 |
| Repository contract | `src/Libraries/Nop.Data/IRepository.cs` | method list incl. `publishEvent` flags, cache-key delegates |
| Product page | `src/Presentation/Nop.Web/Controllers/ProductController.cs` | `ProductDetails` L129–L203: ACL check L139, store-mapping check L141, admin-preview bypass L146–L148, activity log L194, factory call L198 |
| Product factory | `src/Presentation/Nop.Web/Factories/ProductModelFactory.cs` | `PrepareProductDetailsModelAsync` L1502; recursive call for associated products L1727; file is 2024 lines |
| Add to cart (web) | `src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs` | `AddProductToCart_Catalog` L593; `AddProductToCart_Details` L791–L874 (form parsing, attribute parsing, SaveItemAsync); `ProductDetails_AttributeChange` L879 |
| Add to cart (service) | `src/Libraries/Nop.Services/Orders/ShoppingCartService.cs` | `AddToCartAsync` L1549–L1728 (main body to L1684; local function `addRequiredProductsToCartAsync` L1686–L1727; anchors updated from L1668 to L1684 on 2026-10-06): permission checks L1560–L1570, bot check L1572, warning-list validation, update-vs-insert branch; `GetShoppingCartAsync` L753; `GetShoppingCartItemWarningsAsync` L1124; file is 1976 lines |
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
| Domain entity | `src/Libraries/Nop.Core/Domain/Catalog/Product.cs` | `Product : BaseEntity, ILocalizedEntity, ISlugSupported, IAclSupported, IStoreMappingSupported, IDiscountSupported<>, ISoftDeletedEntity, IMetaTagsSupported` L13; 606 lines (605 newline-terminated lines + a final line without trailing newline, so `wc -l` prints 605) |
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

## 2026-10-06 accuracy + expansion pass

**Working tree:** a shallow snapshot (`git log` shows a single backup commit, `86f5d3b`); the upstream HEAD `08720d3f05` and commit `99997cecd6` cited above could not be re-checked in this clone. Mapster is present (`Nop.Core.csproj` line 18) and no csproj references AutoMapper, which is consistent with the Mapster claim.

**Citation check.** A script extracted every relative markdown link (and any backtick `path:line`) in the tree, confirmed each target file exists and each `#Lx`/`#Lx-Ly` range lies within the file; a second heuristic compared identifiers in link labels with the cited lines. Before edits: 767 links, 0 missing/out-of-range. After edits: 1250 links, 0 missing/out-of-range; every heuristic flag was reviewed by hand (method-body anchors labelled with the method name).

**Corrections made (claims that were wrong, not just imprecise):**

| Claim | Correction | Evidence |
| --- | --- | --- |
| 33 plugin projects | 32 (the `.sln` has a `Plugins` solution folder) | `ls src/Plugins` |
| ~200 registrations in `NopStartup` | 158 `services.Add*` (143 scoped, 13 singleton, 2 transient) | grep count |
| "One interface per service, ~200 times" | 148 interfaces in `Nop.Services` | grep count |
| `Nop.Services` cannot reference MVC / mention `IActionResult` | false: `Nop.Core.csproj` references ASP.NET Core MVC packages (lines 19–20); `ICustomerRegistrationService.cs` L62 returns `IActionResult`; 28 service files mention HTTP types | csproj + grep |
| WebApi.Frontend plugin "bolts a REST API onto the same service" | in this tree it is a 4-file, 128-line configuration shell | `wc -l` |
| gulp writes to `wwwroot/lib` | writes to `wwwroot/lib_npm` (1269 tracked files) | `copyDependencies.js` L9, `git ls-files` |
| "no analyzers/StyleCop config" | a 235-line `.editorconfig` with naming rules exists; CI doesn't enforce it | `.editorconfig` |
| sync-over-async: "one famous exception" | 30 matching lines outside migrations (settings DI factories, validators, Razor helpers, …) | grep in `01-language-runtime-model.md` |
| pipeline order "lives in ApplicationBuilderExtensions.cs" | that file forwards to the engine; order = sorted `INopStartup.Order` values | `NopEngine.cs` L132–L144 |
| `BaseNopTest` fakes a work context | registers the real `WebWorkContext`, mocks `IHttpContextAccessor` | `BaseNopTest.cs` L234–L241, L473 |
| schedule tasks have no tests (R11) | `ScheduleTaskRunnerTests` has 5 tests; individual task classes are untested | test file |
| R2 double-order guard | keyed on customer id, and `PlaceOrderWithLock` is installed as `false` | `OrderProcessingService.cs` L1683; `InstallRequiredData.cs` L1888 |
| `AddToCartAsync` L1549–L1668 | method body runs to L1684 (local function to L1727); anchors widened | source |

**New findings added** (all read from source, none reproduced in a running store) are recorded as R15–R25 in [risk-register.md](risk-register.md).

**Expanded chapters (word counts by `wc -w`, before → after):** 01/01 system map 375 → 2030; 01/03 glossary 508 → 1912; 02/01 runtime 494 → 1926; 02/02 framework 513 → 1772; 02/03 types 476 → 1661; 02/04 tooling 394 → 1711; 03/01 boundaries 582 → 1678; 03/02 data 550 → 2004; 03/03 auth 559 → 1598; 03/04 side effects 611 → 1969; 05/01 testing 493 → 1458; 05/06 observability 507 → 1605. Each now has a feature trace, Goal + **Check** exercise ladder, verified senior finding(s), a debugging scenario and its own dated verification note.

**Still not done:** no `dotnet build`/`dotnet test` was run; every "run locally" rung states an outcome derived from code. Chapters 04, 06, 07, 08 and the already-dense files received accuracy fixes only.
