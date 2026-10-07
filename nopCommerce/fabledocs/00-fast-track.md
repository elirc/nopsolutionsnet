# Fast Track — One Weekend

Goal: by Sunday night you can run the store, explain two end-to-end flows out loud, and have made one safe change with a passing test. That is more than most candidates can say about any large codebase.

## Saturday morning — run it

Prerequisite: .NET SDK **10.0.100+** (pinned in [global.json](../global.json)) and a database — SQL Server, MySQL, or PostgreSQL. Docker compose files exist for each: [docker-compose.yml](../docker-compose.yml), [mysql-docker-compose.yml](../mysql-docker-compose.yml), [postgresql-docker-compose.yml](../postgresql-docker-compose.yml).

```bash
# __inferred__ (mirrors .github/workflows/dotnet.yml; not executed during authoring)
dotnet restore src
dotnet build src --configuration Debug
dotnet run --project src/Presentation/Nop.Web
```

First launch redirects to an **installation page**: you enter admin credentials and a connection string, and the app creates the schema by running every migration in [src/Libraries/Nop.Data/Migrations/Installation](../src/Libraries/Nop.Data/Migrations/Installation) plus seed data. The generated config lands in `src/Presentation/Nop.Web/App_Data/appsettings.json` — loaded explicitly in [Program.cs](../src/Presentation/Nop.Web/Program.cs), not the usual project-root appsettings. Tick "install sample data" so the catalog isn't empty.

Run the tests (no database needed — they use SQLite, see below):

```bash
# __inferred__ from CI
dotnet test src
```

## Saturday afternoon — trace flow #1: product page

Open these files **in this order** and read only the cited ranges:

1. [Program.cs](../src/Presentation/Nop.Web/Program.cs) — the whole file (~50 lines). Note: config comes from `App_Data`, DI can be Autofac *or* the built-in container, and the pipeline is built by extension methods, not inline code. This tells you the app's real entry points live in `Nop.Web.Framework/Infrastructure`.
2. [NopStartup.cs](../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67) — skim the giant block of `services.AddScoped<IThing, Thing>()`. Every service you'll ever meet is registered here. One interface, one implementation, scoped-per-request. That's the whole DI story.
3. [ProductController.ProductDetails](../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129-L203) — the single best 75 lines in the repo. In order: load entity → 404 checks → **ACL check** (L139) → **store-mapping check** (L141) → admin-preview bypass (L146–L148) → activity log → hand off to a *model factory* → pick a view template. Controllers here orchestrate; they don't compute.
4. [ProductModelFactory.PrepareProductDetailsModelAsync](../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502) — skim 30 lines. This is where entity → view-model mapping lives. The factory layer is nopCommerce's signature move: controllers never build models inline.
5. [Product.cs](../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) — read just the class declaration line. Count the interfaces: `ILocalizedEntity, ISlugSupported, IAclSupported, IStoreMappingSupported, IDiscountSupported<>, ISoftDeletedEntity, IMetaTagsSupported`. Each one is a cross-cutting capability the framework layers onto entities. This single line is a map of the whole system.

**Pause and predict:** before opening file 3, write down what checks *you* would perform before showing a product page. Then compare. Most people miss store-mapping (multi-store isolation) and the admin-preview bypass.

## Saturday evening — trace flow #2: add to cart

1. [ShoppingCartController.AddProductToCart_Details](../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791-L874) — parses the raw form (attributes, quantity, rental dates), then delegates.
2. [ShoppingCartService.AddToCartAsync](../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1684) — the validation style to internalize: a **warnings list** accumulates human-readable failures and the method returns them instead of throwing. Permission checks first (L1560–L1570), then a bot check (L1572), then update-vs-insert.
3. [EntityRepository.UpdateAsync](../src/Libraries/Nop.Data/EntityRepository.cs#L393) — every write publishes an `EntityUpdatedEvent<T>`; [CacheEventConsumer](../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L13-L53) hears it and evicts caches. Writes and cache invalidation are coupled by *events*, not by discipline.

## Sunday — first 10 files, one change, one teach-back

**The 10 files** (if you only ever read ten): the five from flow #1, the three from flow #2, plus [CustomerRegistrationService.ValidateCustomerAsync](../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) (login with lockout — textbook security) and [QueuedMessagesSendTask.cs](../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) (an entire outbox-pattern background job in 93 lines).

**One safe change:** run one focused test suite, then modify a test to prove you can:

```bash
# __inferred__ — NUnit filter syntax
dotnet test src --filter "FullyQualifiedName~ProductServiceTests"
```

Open [ProductServiceTests.cs](../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63-L110), read the product-availability tests, add one new case (e.g., available-until date exactly equal to now). Note that the test base ([BaseNopTest.cs](../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L116)) spins up a **real SQLite database with real seed data** — these are integration-style tests, not mock farms. That's a deliberate strategy; module [05-quality-engineering/01-testing-strategy.md](05-quality-engineering/01-testing-strategy.md) explains the tradeoff.

**Teach-back (do this out loud, 3 minutes, phone timer):** "Walk me through what happens when a customer clicks Add to Cart on nopCommerce." Interview-grade answer touches: controller parses form → service validates via warnings list (permissions, quantity, stock) → repository write → entity event → cache eviction. If you can't do it without notes, re-trace flow #2.

## What the fast track skips

Checkout/payments (the deepest flow — [05-key-flows.md](01-codebase-cartography/05-key-flows.md) Flow 4), the plugin system, multi-store/multi-language mechanics, migrations, the admin area, and all drills. It also skips *why* behind every pattern — that's the rest of the curriculum.

**Next:** [01-codebase-cartography/01-system-map.md](01-codebase-cartography/01-system-map.md), or if interviewing soon, [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md).
