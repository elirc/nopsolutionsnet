# System Design From This Repo

The whiteboard exercise: **"Design an e-commerce platform."** You have an unfair advantage — you've read a real one. This walkthrough runs the standard interview arc (requirements → API → data model → deep dives → scaling) and at every step shows what nopCommerce actually chose (with anchors), why, and what a stronger or simpler alternative looks like. Rehearse it aloud; 35–40 minutes is the target.

---

## Step 1 — Requirements (5 min)

Say functional requirements first, crisply: browse catalog, search, cart, checkout with payment, order history, admin management. Then the ones that make it interesting: **multi-store** from one deployment, **multi-vendor**, **multi-language/currency**, role-based admin, extensibility (plugins).

Non-functional: read-heavy (catalog reads ≫ writes, often 100:1), checkout must be correct over fast, catalog must be fast over fresh.

- **Junior answer:** lists features.
- **Mid answer:** separates read-heavy catalog from correctness-critical checkout and states different strategies for each.
- **Senior answer:** also asks who operates it — nopCommerce's real constraint is *thousands of independent operators on commodity hosting*, which explains almost every choice below (in-app scheduler, DB-as-queue, plugin system).

## Step 2 — High-level shape (5 min)

```
Browser ──> ASP.NET Core MVC (storefront + /Admin area)
              │  controllers → model factories → services
              ▼
        Service layer (all business logic)
              │  IRepository<T>
              ▼
        SQL (SQL Server / MySQL / Postgres)   Memory/Redis cache
              ▲
        In-app task scheduler ──> queued email outbox, cache clears
```

What the repo chose: a **modular monolith**, three libraries under one web host — [Nop.Core](../../src/Libraries/Nop.Core) (domain), [Nop.Data](../../src/Libraries/Nop.Data) (persistence), [Nop.Services](../../src/Libraries/Nop.Services) (logic), with [Nop.Web](../../src/Presentation/Nop.Web) on top. Composition happens in [Program.cs](../../src/Presentation/Nop.Web/Program.cs) + [NopStartup.cs#L67](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67).

Say this in the interview: "I'd start with a modular monolith — e-commerce's transactional core (cart→order→payment) hates network partitions, and a monolith keeps checkout in one transaction domain. I'd keep module boundaries library-shaped so extraction is possible later." That sentence alone clears the mid-level bar; naming *what would force extraction* (team scale, independent deploy cadence, search/reporting workloads) clears more.

Stronger/simpler alternatives: microservices (only justified at large org scale); serverless storefront over a commerce API (modern headless approach — say it exists, don't build it).

## Step 3 — API sketch (5 min)

The repo is server-rendered MVC, not REST — routes like `/product/{slug}` resolve to [ProductController.ProductDetails](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129), and mutations are form POSTs returning JSON fragments (see the `{success, message, redirect}` shapes in [ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)). An actual REST API ships as a plugin ([Nop.Plugin.Misc.WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend)).

For the whiteboard, sketch REST and keep moving:

```
GET  /products?category=&page=        GET  /products/{id}
POST /cart/items {productId, qty}     PATCH /cart/items/{id}
POST /checkout/order  (Idempotency-Key header)
GET  /orders/{id}                     admin: CRUD under /admin, permission-gated
```

Design points to say out loud: pagination is mandatory on collections (the repo bakes it into the data layer as [IPagedList / GetAllPagedAsync](../../src/Libraries/Nop.Data/EntityRepository.cs#L302)); server computes all prices (never trust client amounts — the repo re-derives totals in [PlaceOrderAsync step 1](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1602)); order creation takes an idempotency key (the repo's `OrderGuid` at [L1598](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598) is exactly this).

## Step 4 — Data model (7 min)

Core entities: `Customer`, `Product`, `Category`, `ShoppingCartItem`, `Order`, `OrderItem`, plus `Store` and mapping tables.

What the repo chose, worth quoting:

- **Integer PKs, `BaseEntity.Id`** everywhere ([BaseEntity.cs](../../src/Libraries/Nop.Core/BaseEntity.cs)). Simple, join-friendly; guessable IDs make object-level authorization non-optional (IDOR — see Flow 6 in [key flows](../01-codebase-cartography/05-key-flows.md)).
- **Order snapshots, not references:** `OrderItem` copies unit price at purchase time — order history must not change when the catalog does. This is the #1 data-modeling trap in e-commerce interviews; say "orders are immutable snapshots" explicitly.
- **Soft delete** via `ISoftDeletedEntity` ([Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13)) — orders reference products forever, so products are flagged, not dropped.
- **Cross-cutting via mapping tables:** ACL and store visibility are generic mapping tables keyed by entity name + id, checked by [StoreMappingService.AuthorizeAsync#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221). Generic → flexible; cost: no FK integrity on the polymorphic reference.
- **Product attributes as XML strings** (`attributesXml` through the whole cart flow, [ShoppingCartService.cs#L1549-L1553](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1553)). Legacy EAV-ish choice; today you'd use JSON columns. Naming this as *technical debt with a migration path* is a great senior move.
- **Multi-tenancy = shared schema + StoreId columns/mappings**, not schema-per-tenant. Right for thousands of small stores; wrong for isolation-sensitive tenants.

Migrations: forward-only, timestamp-versioned FluentMigrator ([PriceListMigration.cs#L7](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7)) — bring this up when asked "how do you evolve the schema?"

## Step 5 — Deep dive: checkout (8 min)

This is where interviews are won. Walk the real sequence from [PlaceOrderAsync](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594-L1719): validate & compute totals server-side → call payment gateway → persist order + items → discount/gift-card histories → publish `OrderPlacedEvent` → queue emails (outbox, [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs)) → status checks.

Then volunteer the two hard problems before the interviewer asks:

1. **Payment/persistence gap.** Gateway capture succeeds, then the DB write fails → money without an order. The repo mitigates with an idempotent `OrderGuid`, error collection, logging ([L1661-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1661-L1674)). Stronger design: authorize → persist order in `Pending` → capture → mark paid, with a reconciliation job against the gateway. Saying "authorize/capture split + reconciliation" is the senior answer.
2. **Double submission.** The repo's guard is a named `Mutex` per customer + cache flag ([L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718)) — works on one node only, and uses sync-over-async because a `Mutex` can't cross `await` (their comment, L1685-L1688). Stronger: idempotency key enforced by a unique DB constraint — the database becomes the lock. Contrast the two and you've demonstrated distributed-systems judgment with a real citation.

## Step 6 — Reads, caching, scaling (7 min)

Catalog reads: cache-aside everywhere — [MemoryCacheManager.GetAsync#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) with composed keys; invalidation is *event-driven*: every repository write publishes an entity event and [CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) evicts by prefix. Coarse but correct. Scale-out story: swap memory cache for Redis ([DistributedCacheManager](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs) exists), CDN static assets, read replicas for reporting, search offloaded to Lucene/Elastic (plugin exists: [Nop.Plugin.Search.Lucene](../../src/Plugins/Nop.Plugin.Search.Lucene)).

Background work: in-app scheduler ([ScheduleTaskRunner#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115)) + DB-as-queue. At 10x, move to a real broker + worker processes; the outbox table survives as the bridge.

## Variation prompts (practice each for 5 minutes)

1. **"Now make it multi-tenant SaaS."** The repo already is (StoreId scoping) — discuss shared-schema vs schema-per-tenant vs DB-per-tenant, and where its ACL/store-mapping checks would need hardening.
2. **"Add real-time inventory across nodes."** Memory cache breaks; discuss Redis, stock reservations with TTL, and the oversell/undersell tradeoff.
3. **"Handle 10x Black Friday traffic."** Cache stampede on hot products, queue depth on checkout, DB connection limits; degrade gracefully (cached prices, async order confirmation).
4. **"Split payments into a service."** What contract? Idempotent capture API, outbox events both directions, saga for refunds — and what you lose (single-transaction simplicity you saw in Step 5).
5. **"Add a mobile REST API."** The repo's answer is a plugin ([WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend)); discuss versioning and not coupling API models to view models.

## Grading yourself

- **Basic:** covers steps 1–4 coherently.
- **Solid (mid-level bar):** volunteers the checkout deep dive with the payment-gap and double-submit problems, and can cite how a real system handled both.
- **Strong:** runs any variation prompt without notes, names what would force architecture changes, and keeps tying claims back to concrete evidence ("in nopCommerce this is a named mutex because...").

Cross-links: architecture judgment behind these answers → [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md); flow details → [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md).
