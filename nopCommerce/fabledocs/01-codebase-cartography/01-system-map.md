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
│ logic, 40+    │          │ FluentMigrator,  │        │ caching, events, │
│ subdomains    │          │ Linq2Db providers│        │ infrastructure   │
└───────────────┘          └──────────────────┘        └──────────────────┘
        ▲
        │ registered at runtime
┌───────┴───────┐
│  src/Plugins  │  33 projects: payments, shipping, tax, auth, widgets, search…
└───────────────┘
```

Dependency direction is strictly downward: Core knows nothing; Data knows Core; Services knows Data+Core; Web knows everything. That single rule is the architecture.

## Ownership map

| Concern | Lives in | Landmark |
| --- | --- | --- |
| Domain entities (no behavior) | `Nop.Core/Domain/*` | [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) |
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

Because of the plugin ecosystem, "public" here is unusually broad: **every `public virtual` service method is effectively a published API** (see [pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)). True external surfaces: the storefront routes, the `/Admin` area, and the REST API *plugin* ([WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend)). Genuinely internal: `Nop.Data`'s provider implementations and the migration machinery.

Senior noticing: the interesting boundary is not Web↔Services (obvious) but **Services↔plugins** — plugins consume the same service interfaces the app does, which is why service signatures change so rarely and why so much lives in settings objects instead of parameters.
