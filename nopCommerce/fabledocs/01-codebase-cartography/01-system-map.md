# System Map

## Shape

Single .NET solution ([src/NopCommerce.sln](../../src/NopCommerce.sln)), not a workspace monorepo: one deployable web app plus class libraries and runtime-loaded plugins. Target `net10.0` ([Directory.Build.props](../../src/Directory.Build.props)).

```
                      ┌────────────────────────────────────────┐
                      │  Nop.Web  (the only deployable)        │
                      │  Controllers / Factories / Views /     │
                      │  Areas/Admin / Validators / wwwroot    │
                      └───────────────┬────────────────────────┘
                                      │ uses
                      ┌───────────────▼────────────────────────┐
                      │  Nop.Web.Framework                     │
                      │  MVC plumbing: filters, DI startup,    │
                      │  WorkContext, themes, tag helpers      │
                      └───────────────┬────────────────────────┘
        ┌─────────────────────────────┼──────────────────────────┐
        ▼                             ▼                          ▼
┌───────────────┐          ┌──────────────────┐        ┌──────────────────┐
│  Nop.Services │────────▶ │    Nop.Data      │───────▶│     Nop.Core     │
│ ALL business  │          │ IRepository,     │        │ domain entities, │
│ logic, 40     │          │ FluentMigrator,  │        │ caching, events, │
│ subfolders    │          │ Linq2Db providers│        │ infrastructure   │
└───────────────┘          └──────────────────┘        └──────────────────┘
        ▲
        │ registered at runtime
┌───────┴───────┐
│  src/Plugins  │  32 projects: payments, shipping, tax, auth, widgets, search…
└───────────────┘
```

Dependency direction is strictly downward: Core knows nothing; Data knows Core; Services knows Data+Core; Web knows everything. That single rule is the architecture.

Counts above were taken by listing directories on 2026-10-06: `ls -d src/Libraries/Nop.Services/*/` returns 40 folders; `ls src/Plugins` returns 32 plugin projects (the `.sln` contains 33 lines matching `Plugin`, but one is the `Plugins` *solution folder*, not a project — an easy miscount).

## Ownership map

| Concern | Lives in | Landmark |
| --- | --- | --- |
| Domain entities (no behavior) | `Nop.Core/Domain/*` (31 subfolders) | [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) |
| Caching, events, DI engine | `Nop.Core/{Caching,Events,Infrastructure}` | [MemoryCacheManager.cs#L13](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L13) |
| Data access + schema | `Nop.Data` | [EntityRepository.cs#L14](../../src/Libraries/Nop.Data/EntityRepository.cs#L14), [Migrations/](../../src/Libraries/Nop.Data/Migrations) |
| Business logic | `Nop.Services/{Orders,Catalog,Customers,…}` | [OrderProcessingService.cs](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594) |
| HTTP: storefront | `Nop.Web/Controllers` + `Factories` + `Views` | [ProductController.cs#L129](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129) |
| HTTP: admin | `Nop.Web/Areas/Admin` | [Admin ProductController](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943) |
| Filters, auth attributes, startup | `Nop.Web.Framework` | [NopStartup.cs#L67](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67) |
| Background jobs | `Nop.Services/ScheduleTasks` + task classes per domain | [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| Tests | `src/Tests/Nop.Tests` (mirrors library structure) | [BaseNopTest.cs#L98](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98) |
| Client assets | `Nop.Web/wwwroot`, gulp pipeline | [package.json](../../src/Presentation/Nop.Web/package.json) |

## Public interfaces vs private internals

Because of the plugin ecosystem, "public" here is unusually broad: **every `public virtual` service method is effectively a published API** (see [pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)). True external surfaces: the storefront routes, the `/Admin` area, and the REST API *plugin* ([WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend) — in this source tree only a 128-line configuration shell; its endpoints are not in this repository). Genuinely internal: `Nop.Data`'s provider implementations and the migration machinery.

Senior noticing: the interesting boundary is not Web↔Services (obvious) but **Services↔plugins** — plugins consume the same service interfaces the app does, which is why service signatures change so rarely and why so much lives in settings objects instead of parameters.

## How the process boots (verified sequence)

A system map is incomplete until you know *in what order the boxes come alive*. Read [Program.cs](../../src/Presentation/Nop.Web/Program.cs) top to bottom — it is 52 lines:

1. **Configuration** — `appsettings` JSON, an environment-specific JSON, then environment variables ([Program.cs#L14-L20](../../src/Presentation/Nop.Web/Program.cs#L14-L20)).
2. **Container choice** — `UseAutofac` from `CommonConfig` selects Autofac or the default Microsoft container; the default container is configured with `ValidateScopes = false` and `ValidateOnBuild = true` ([Program.cs#L25-L39](../../src/Presentation/Nop.Web/Program.cs#L25-L39)).
3. **Services** — `ConfigureApplicationServices` initializes plugins into the MVC part manager, creates the engine, adds a global rate limiter, and calls `engine.ConfigureServices` ([ServiceCollectionExtensions.cs#L101-L138](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L101-L138)).
4. **Module discovery** — the engine finds every `INopStartup` in every loaded assembly (core *and* plugins), instantiates them with `Activator.CreateInstance`, sorts by `Order`, and calls `ConfigureServices` on each ([NopEngine.cs#L104-L114](../../src/Libraries/Nop.Core/Infrastructure/NopEngine.cs#L104-L114)).
5. **Schema migrations run here, at boot** — `NopDbStartup` (Order 10) registers `IRepository<>` → `EntityRepository<>` ([NopDbStartup.cs#L63](../../src/Libraries/Nop.Data/NopDbStartup.cs#L63)) and then, if the DB is installed, builds a temporary service provider and applies schema migrations from every assembly that contains migrations ([NopDbStartup.cs#L65-L71](../../src/Libraries/Nop.Data/NopDbStartup.cs#L65-L71)).
6. **Pipeline** — `app.ConfigureRequestPipeline()` asks the engine to run each startup's `Configure` in the same `Order` ([ApplicationBuilderExtensions.cs#L52-L55](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L52-L55)).
7. **AppStartedEvent** — published once through the event system ([ApplicationBuilderExtensions.cs#L62-L67](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L62-L67)), then `RunAsync`.

The `Order` values are the real "map of the pipeline". Collected by grepping `Order =>` in every class that implements `INopStartup` (`Nop.Web.Framework/Infrastructure`, `Nop.Data/NopDbStartup.cs`, `Nop.Web/Infrastructure`, plugin `Infrastructure` folders):

| Order | Startup class | Comment in source |
| --- | --- | --- |
| -1 | `NopProxyStartup` | forwarded headers first |
| 0 | `ErrorHandlerStartup` | error handlers first |
| 10 | `NopDbStartup` (in `Nop.Data`) | data layer + migrations |
| 99 | `NopStaticFilesStartup` | before routing |
| 100 | `NopCommonStartup` | after error handlers |
| 300 | `NopWebMarkupMinStartup` | before `UseRouting` |
| 400 | `NopRoutingStartup` | before authentication |
| 500 | `AuthenticationStartup` | between routing and endpoints |
| 600 | `AuthorizationStartup` | after authentication |
| 900 | `NopEndpoints` | before MVC |
| 1000 | `NopMvcStartup` | "MVC should be loaded last" |
| 2000 | `NopStartup` (core service registrations) | [NopStartup.cs#L336](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L336) |
| 2002 | `Nop.Web` `NopStartup` | [Nop.Web/Infrastructure/NopStartup.cs#L129](../../src/Presentation/Nop.Web/Infrastructure/NopStartup.cs#L129) |
| 3000 | most plugin startups, e.g. [AzureBlob](../../src/Plugins/Nop.Plugin.Misc.AzureBlob/Infrastructure/PluginNopStartup.cs#L46) | registered *after* core |

Why plugins use 3000: with the Microsoft container, the last registration for a service type wins on single resolution. Core registers `IThumbService` at [NopStartup.cs#L287](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L287); the AzureBlob plugin re-registers `IThumbService` with a settings-driven factory at [PluginNopStartup.cs#L24-L32](../../src/Plugins/Nop.Plugin.Misc.AzureBlob/Infrastructure/PluginNopStartup.cs#L24-L32), and because 3000 > 2000 the plugin's registration is the one resolved. A few plugins use `Order => 1` (e.g. [PayPalCommerce](../../src/Plugins/Nop.Plugin.Payments.PayPalCommerce/Infrastructure/NopStartup.cs#L41)) — those only *add* their own types, so ordering doesn't matter for them.

## Feature trace: one storefront product page, box by box

Follow `GET /some-product-slug` through every box in the diagram:

| Hop | Box | Code | What happens |
| --- | --- | --- | --- |
| 1 | Web.Framework | [GenericUrlRouteProvider.cs#L38-L41](../../src/Presentation/Nop.Web/Infrastructure/GenericUrlRouteProvider.cs#L38-L41) | a dynamic route hands unknown paths to `SlugRouteTransformer` |
| 2 | Web.Framework | [SlugRouteTransformer.cs#L109-L110](../../src/Presentation/Nop.Web.Framework/Mvc/Routing/SlugRouteTransformer.cs#L109-L110) | the slug's `UrlRecord` says entity `Product` → route to `Product/ProductDetails` with `productId` |
| 3 | Web | [ProductController.cs#L129-L131](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129-L131) | action loads the product through the service interface |
| 4 | Services | [ProductService.cs#L555-L558](../../src/Libraries/Nop.Services/Catalog/ProductService.cs#L555-L558) | `GetProductByIdAsync` → `_productRepository.GetByIdAsync(productId, cache => default)` — `default` means "use the default by-id cache key" |
| 5 | Data | [EntityRepository.cs#L122-L144](../../src/Libraries/Nop.Data/EntityRepository.cs#L122-L144) | builds the key, delegates to the static cache with a query lambda over `Table` |
| 6 | Core | [MemoryCacheManager.cs#L132-L147](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132-L147) | `GetOrCreate` with a `Lazy<Task<T>>` — at most one DB hit per key even under concurrent misses |
| 7 | Web → Services | [ProductController.cs#L135-L148](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L135-L148) | published / ACL / store-mapping / availability checks, with an admin-preview bypass |
| 8 | Web | [ProductController.cs#L181-L195](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L181-L195) | side-effect writes on a GET: recently-viewed list + activity log |
| 9 | Web | [ProductController.cs#L198-L202](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L198-L202) | factory builds the view model; the template view path is chosen per product |

Notice that one request touches all five projects, and that the *only* place the database is reached is through `EntityRepository` → `INopDataProvider`. That is the map made concrete.

## Exercise ladder

Each rung has a **Check** you can perform with only the source tree (no build required).

1. **Goal:** confirm the dependency direction is compiler-enforced, not just drawn.
   **Check:** open [Nop.Services.csproj](../../src/Libraries/Nop.Services/Nop.Services.csproj) and [Nop.Data.csproj](../../src/Libraries/Nop.Data/Nop.Data.csproj) and list their `ProjectReference` items. Expected: Services references both Core and Data (lines 32–33 of its csproj); Data references only Core (line 27); neither references a `Presentation` project.
2. **Goal:** reproduce the plugin count without trusting this page.
   **Check:** `ls src/Plugins | wc -l` → 32. Then `grep "Project(" src/NopCommerce.sln | grep -c Plugin` → 33, and explain the difference (one match is the `Plugins` solution folder).
3. **Goal:** predict which registration wins for a contested service.
   **Check:** `grep -rn "IThumbService" src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs src/Plugins/*/Infrastructure/*.cs`. Using the `Order` table above, state which implementation is resolved when the AzureBlob plugin is installed, and why the answer would flip if the plugin's `Order` were 1.
4. **Goal (mid):** find every place a request can reach the database *without* going through `IRepository<T>`.
   **Check:** `grep -rn "_dataProvider\." src/Libraries/Nop.Services --include=*.cs | wc -l` (123 lines on 2026-10-06; many are bulk/installation paths), then open two hits. Expected finding: services like [AclService.cs#L315](../../src/Libraries/Nop.Services/Security/AclService.cs#L315) and [StoreMappingService.cs#L269](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L269) call `INopDataProvider` directly — which skips the repository's entity events (see the senior finding below).
5. **Goal (mid):** draw the boot sequence for a node joining a load-balanced farm during an upgrade.
   **Check:** your drawing must place the migration step from [NopDbStartup.cs#L65-L71](../../src/Libraries/Nop.Data/NopDbStartup.cs#L65-L71) *before* the pipeline is built, and must state what happens if two nodes boot at the same moment (answer in the finding below).

## Senior findings (verified in source)

**F1 — Schema migrations run inside `ConfigureServices` on every node.** [NopDbStartup.cs#L65-L71](../../src/Libraries/Nop.Data/NopDbStartup.cs#L65-L71) builds a throw-away `ServiceProvider` (`services.BuildServiceProvider()`) and calls `ApplyUpSchemaMigrations` for each migration assembly, every time the process starts with an installed database. Consequences: (a) every web node is also a migration runner, so a rolling deploy of N nodes means N processes attempting the same schema change; whether that is safe depends entirely on FluentMigrator's version-table semantics on your database engine — nothing in this file coordinates them; (b) a failing migration fails *startup*, not a separate deploy step, so the symptom is "site won't boot" rather than "migration job failed"; (c) building a second provider during registration creates singleton instances that are not the ones the real app uses. Confidence: high on the mechanism (it's ten lines); impact depends on deployment topology. The standard alternative is a dedicated migration step in the release pipeline.

**F2 — Not every write goes through the repository.** The map says "data access = `EntityRepository`", but `INopDataProvider` is injected directly in services; the two cited hits above flip `SubjectToAcl` / `LimitedToStores` on an entity without publishing `EntityUpdatedEvent`, so cache consumers keyed on that entity never hear about it. This is the kind of thing a map *hides* — boxes imply discipline the code doesn't fully have. Full analysis in [03-architecture-and-patterns/02-data-model-and-persistence.md](../03-architecture-and-patterns/02-data-model-and-persistence.md).

## Debugging scenario: "my plugin's override isn't being used"

*Symptom:* you wrote `services.AddScoped<IThumbService, MyThumbService>()` in a plugin's `INopStartup`, deployed, and the core `ThumbService` still runs.

1. **Hypothesis A — order.** Check your startup's `Order`. If it is less than 2000, core's `NopStartup` registers `IThumbService` *after* you and wins. Compare with [AzureBlob's `Order => 3000`](../../src/Plugins/Nop.Plugin.Misc.AzureBlob/Infrastructure/PluginNopStartup.cs#L46).
2. **Hypothesis B — the plugin assembly isn't loaded.** Startup discovery only sees loaded assemblies ([NopEngine.cs#L104](../../src/Libraries/Nop.Core/Infrastructure/NopEngine.cs#L104)). Plugins build into `Nop.Web/Plugins/<Name>` via their csproj `OutputPath` (e.g. [Payments.Manual csproj line 12](../../src/Plugins/Nop.Plugin.Payments.Manual/Nop.Plugin.Payments.Manual.csproj)) and must be installed; an uninstalled plugin's startup never runs.
3. **Hypothesis C — someone resolved the service before your override mattered.** Because `ValidateScopes = false` ([Program.cs#L36](../../src/Presentation/Nop.Web/Program.cs#L36)), a singleton that captured the old scoped instance would keep using it. Grep for singletons that take `IThumbService` in their constructor.
4. **Confirm** by logging `GetType().FullName` of the resolved `IThumbService` in a controller (run locally) — a one-line probe that ends the guessing.

The general lesson: in a module-discovery system, "where is X registered?" is a question about *ordering across assemblies*, and the answer is always the sorted `Order` table, not the file you happen to be reading.

## Three things the box diagram does not show

1. **A static service locator cuts across the layers.** `EngineContext.Current` (in `Nop.Core/Infrastructure`) lets any code resolve any service. It is used where constructor injection is impossible — e.g. the `CacheEventConsumer` base constructor ([CacheEventConsumer.cs#L28-L32](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L28-L32)) and the event publisher's consumer lookup ([EventPublisher.cs#L23](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L23)). Arrows on a diagram assume dependencies are visible; these are not.
2. **Settings objects are DI services resolved per request.** Every `ISettings` class is registered as a scoped factory that loads it for the current store ([NopStartup.cs#L273-L283](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L273-L283)) — and that factory blocks on two async calls with `.Result`. So "configuration" is not a box at the top of the diagram; it is a database read inside the service graph.
3. **Identity is resolved by the framework layer, and it writes.** `WebWorkContext.SetCurrentCustomerAsync` walks a chain (background-task user → search-engine user → authenticated user → impersonation → guest cookie) and, if nothing matches, *inserts a guest customer row* ([WebWorkContext.cs#L212-L284](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L212-L284), insert at L273). A cookieless crawler that is not recognised as a search engine therefore creates rows. The diagram shows `Nop.Web.Framework` as "plumbing"; it is also a writer of business data.

## Verification

2026-10-06: all line anchors on this page opened and checked against the working tree; plugin count corrected from 33 to 32 (directory listing), Nop.Services subfolder count (40) and Nop.Core/Domain subfolder count (31) taken with `ls -d */`; the `Order` table was produced by grepping `Order =>` in each startup file. No build or run was performed.
