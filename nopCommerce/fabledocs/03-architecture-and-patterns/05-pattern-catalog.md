# Pattern Catalog

Sixteen patterns this codebase actually uses, as recognition training. For each: what problem it solves, where it lives here, and when it fails. The goal is not to name-drop patterns in interviews — it's to *recognize* them in any repo in under a minute and discuss their tradeoffs.

---

## Pattern 1: Generic Repository

Problem it solves: one consistent, testable seam for all data access; entities never leak ORM specifics upward.
General shape: `IRepository<TEntity>` with typed CRUD + an `IQueryable` escape hatch; services compose queries, the repository owns execution, events, and caching.
Real example: [EntityRepository.cs#L14-L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L14-L122) (class + cached `GetByIdAsync`)
Second example: contract in [IRepository.cs](../../src/Libraries/Nop.Data/IRepository.cs); the `Table` IQueryable property at [EntityRepository.cs#L525](../../src/Libraries/Nop.Data/EntityRepository.cs#L525)
Why this implementation works: every write funnels through `InsertAsync`/`UpdateAsync`/`DeleteAsync`, which is what makes system-wide entity events (Pattern 4) and cache invalidation *reliable* instead of aspirational.
Failure modes: the `Table` IQueryable escape hatch lets services build queries the repository can't see — bypassing caching; generic repositories also tempt people into N+1 loops of `GetByIdAsync` instead of one joined query.
Use it when: you need uniform cross-cutting behavior on all writes. Avoid it when: your ORM (EF Core) already *is* a repository/UoW and you'd just be wrapping it with a thinner vocabulary.
Interview angle: "Do you wrap EF Core in a repository?" — mid-level answer: "Depends what the wrapper buys. nopCommerce's buys entity events + caching on every write; a wrapper that buys nothing is ceremony."
Drill: find the one write path in `EntityRepository` that does **not** publish an event by default (hint: [DeleteAsync by predicate, L497](../../src/Libraries/Nop.Data/EntityRepository.cs#L497)) and explain what cache consequence that has.

## Pattern 2: Warnings-List Validation (Result Object)

Problem it solves: expected business failures ("out of stock", "wishlist disabled") must reach the UI as friendly messages, not stack traces.
General shape: methods return `IList<string>` warnings or a result object with `Errors`; empty = success; exceptions reserved for the truly unexpected.
Real example: [ShoppingCartService.AddToCartAsync#L1549-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1582)
Second example: [PlaceOrderResult in OrderProcessingService.cs#L1606-L1665](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1606-L1665); enum variant [CustomerLoginResults](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194)
Why this implementation works: callers (web page, admin grid, API plugin) each render failures their own way; the service stays presentation-agnostic; multiple failures can be reported at once.
Failure modes: callers can *ignore* the returned list (compiler won't force handling); string-typed errors are unmatchable — you can't programmatically branch on "which failure" without fragile string checks.
Use it when: failures are part of the domain. Avoid it when: the failure means a broken invariant — throw instead; also avoid stringly-typed results in new code (prefer typed error enums/discriminated results).
Interview angle: "Exceptions vs Result types" — a classic. Cite this repo for the pattern *and* critique the stringly-typed weakness to sound senior.
Drill: find a caller of `AddToCartAsync` and trace how warnings become UI. Then find one place a returned warning list could be accidentally dropped.

## Pattern 3: Model Factory (View-Model Preparation Layer)

Problem it solves: controllers stay thin; the expensive, cache-aware assembly of view models is centralized and reusable.
General shape: `I{Thing}ModelFactory.Prepare{Thing}ModelAsync(entity, ...)` → view model; controllers only orchestrate.
Real example: [ProductModelFactory.PrepareProductDetailsModelAsync#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502) called from [ProductController.cs#L198](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L198)
Second example: [CustomerModelFactory.PrepareLoginModelAsync](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L551) (call site)
Why this implementation works: the same model prep serves multiple actions; factories can layer caching per-fragment; testable without HTTP.
Failure modes: factories grow monstrous (ProductModelFactory is 2,024 lines); recursive prep ([associated products, L1727](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1727)) can fan out into dozens of service calls per page.
Use it when: view models aggregate many sources. Avoid it when: the model is a 1:1 projection — a mapping expression suffices.
Interview angle: "How do you keep controllers thin?" — this is the concrete answer; contrast with MediatR-style handlers as the alternative.
Drill: count distinct injected services in `ProductModelFactory`'s constructor. What does that number tell you about the class's cohesion?

## Pattern 4: Entity Lifecycle Events → Cache Invalidation

Problem it solves: hundreds of cached queries must be evicted when any entity changes — without every service remembering to do it.
General shape: repository publishes `EntityInserted/Updated/DeletedEvent<T>` on every write; a per-entity `CacheEventConsumer<T>` translates events into cache-prefix evictions.
Real example: publish in [EntityRepository.InsertAsync#L340-L355](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L355); consume in [CacheEventConsumer.cs#L13-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L13-L53)
Second example: prefix keys in [NopEntityCacheDefaults](../../src/Libraries/Nop.Core/Caching/NopEntityCacheDefaults.cs)
Why this implementation works: invalidation is *systemic* — new cached queries only need to use the right key prefixes and eviction comes for free.
Failure modes: prefix eviction is coarse (update one product → evict all "all products" caches); the `publishEvent: false` parameter lets a caller silently skip invalidation; direct SQL or `Table`-built updates bypass it entirely.
Use it when: cache correctness matters more than hit rate. Avoid it when: you need fine-grained dependency tracking — then you want explicit cache dependencies or shorter TTLs.
Interview angle: "Two hard problems: cache invalidation..." — describe this event-driven scheme as a real production answer, then name its coarseness as the tradeoff.
Drill: pick `Category`, find its cache consumer under [Nop.Services/Caching](../../src/Libraries/Nop.Services/Caching), and list which prefixes an update evicts.

## Pattern 5: In-Process Pub/Sub (Consumer Registry)

Problem it solves: decouple "something happened" (order placed) from the N things that react (email, reward points, plugins) without the publisher knowing them.
General shape: `IEventPublisher.PublishAsync(evt)` resolves all `IConsumer<TEvent>` from DI and invokes them in a loop, each wrapped in try/catch.
Real example: [EventPublisher.cs#L20-L52](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52)
Second example: publish site [OrderPlacedEvent at OrderProcessingService.cs#L1644](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1644)
Why this implementation works: plugins subscribe by merely registering a consumer — zero changes to core code; per-consumer try/catch means one broken plugin doesn't break checkout.
Failure modes: **synchronous and in-process** — a slow consumer adds latency to the user request; no ordering guarantees between consumers; swallowed exceptions (logged only) mean silent partial failure; not durable — crash mid-fanout loses remaining consumers.
Use it when: reactions are quick, optional, same-process. Avoid it when: reactions are critical or slow — use a durable queue/outbox.
Interview angle: "Event-driven architecture in a monolith?" — this is the honest version: it's the observer pattern over DI, not Kafka. Saying that distinction cleanly is a senior signal.
Drill: find two consumers of `OrderPlacedEvent` (grep `IConsumer<OrderPlacedEvent>`), and decide for each: what happens to the order if it throws?

## Pattern 6: Transactional Outbox (Queued Email)

Problem it solves: emails must not fail or slow the flows that trigger them.
General shape: write the message to a DB table in the same flow as the business change; a background task drains the table with bounded retries.
Real example: drain side [QueuedMessagesSendTask.cs#L41-L90](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L41-L90) (`maxTries = 3`, `finally` increments `SentTries`)
Second example: enqueue side via `IWorkflowMessageService` (e.g. order notifications from [SendNotificationsAndSaveNotesAsync, OrderProcessingService.cs#L1634](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1634))
Why this implementation works: the send is decoupled, retryable, observable (rows are inspectable in admin), and throttled per account (L51).
Failure modes: no backoff (retry cadence = schedule); exhausted messages don't alert; if the business write and the enqueue aren't in one transaction, you can still lose the message (worth *investigating* here).
Use it when: any external side effect rides on a business write. Avoid it when: you have a real broker with transactional publish — then the table is redundant.
Interview angle: "How do you guarantee an email is sent when an order is placed?" — the outbox is *the* expected answer at mid-level and above.
Drill: write the SQL you'd run to find "dead" emails (tries exhausted, never sent). What column proves exhaustion?

## Pattern 7: Scheduled Tasks (In-App Cron)

Problem it solves: recurring work (send queued emails, clear cache, update exchange rates) without external infrastructure.
General shape: `IScheduleTask.ExecuteAsync()` implementations + DB-stored schedule rows + a runner that guards against overlapping runs.
Real example: runner [ScheduleTaskRunner.ExecuteAsync#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115), invoking `task.ExecuteAsync()` at [L74](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L74)
Second example: [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L9) implements `IScheduleTask`
Why this implementation works: zero external dependencies; schedules editable at runtime from admin; `ensureRunOncePerPeriod` prevents pile-ups.
Failure modes: in-process scheduling dies with the app and competes with request traffic for resources; multi-node deployments can double-run tasks unless the run-once guard is distributed-safe (*investigate*).
Use it when: modest recurring jobs, single-node or careful guards. Avoid it when: jobs are heavy/critical — use Hangfire/Quartz cluster mode or platform cron.
Interview angle: "Where would you run a nightly job in a web app?" — compare in-app scheduler vs external worker; name the multi-node double-run hazard.
Drill: list three other `IScheduleTask` implementations (grep) and classify each: idempotent if double-run, or not?

## Pattern 8: Declarative Authorization Filters

Problem it solves: authorization applied uniformly and visibly at the controller surface, not scattered through method bodies.
General shape: attributes (`[AuthorizeAdmin]`, `[CheckPermission(name)]`) wrap an `IAsyncAuthorizationFilter` that consults `IPermissionService`.
Real example: [CheckPermissionAttribute.cs#L16-L119](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16-L119); usage split read/write at [Admin ProductController L943 vs L963](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965)
Second example: area gate [AuthorizeAdminAttribute.cs#L89](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89)
Why this implementation works: reviewers can audit an endpoint's access rule by reading one line above the action; permission names are constants, greppable.
Failure modes: attributes can't express *row ownership* (IDOR) — vendor scoping is manual inside actions; forgetting the attribute on a new action fails **open** unless a controller-level default exists.
Use it when: role/permission gates. Avoid relying on it for: object-level checks.
Interview angle: "How do you prevent a user editing someone else's record?" — attribute authz + in-query scoping; be precise that the attribute alone is insufficient.
Drill: grep a random admin controller and verify every mutating action carries a write-level permission. Report any that don't (candidate finding!).

## Pattern 9: Capability Interfaces on Entities

Problem it solves: cross-cutting features (localization, ACL, multi-store, soft delete, slugs, discounts) applied per-entity without inheritance explosions.
General shape: marker/capability interfaces; generic services accept `where TEntity : BaseEntity, ICapability`.
Real example: [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) implements seven capability interfaces
Second example: [StoreMappingService.AuthorizeAsync<TEntity> where TEntity : IStoreMappingSupported, #L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221)
Why this implementation works: one `AuthorizeAsync` works for products, categories, topics...; the type system documents which entities have which behaviors.
Failure modes: capabilities are opt-in — forgetting `IStoreMappingSupported` on a new entity silently means "visible in every store"; behavior is spread across services, so the entity file doesn't show *how* the capability works.
Use it when: many entities share orthogonal behaviors. Avoid it when: only one entity needs it — plain code beats a framework of one.
Interview angle: "Composition over inheritance, concretely?" — this is a crisp production example.
Drill: pick `ISoftDeletedEntity`. Find where the framework actually applies the `Deleted` filter (hint: look in Nop.Data extensions/queries) and state the invariant it maintains.

## Pattern 10: Strongly-Typed Settings Objects

Problem it solves: hundreds of store-configurable options, editable at runtime, multi-store-scoped, injected as plain typed objects.
General shape: `class XxxSettings : ISettings` persisted as name/value rows; DI materializes and injects them; admin UI edits them.
Real example: `_shoppingCartSettings.MaximumShoppingCartItems` used at [ShoppingCartService.cs#L1637](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1637)
Second example: `_customerSettings.FailedPasswordAllowedAttempts` at [CustomerRegistrationService.cs#L161](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L161)
Why this implementation works: business rules become data — lockout thresholds, cart limits, captcha toggles change without deploys; typed access keeps refactors safe.
Failure modes: settings sprawl (every bug becomes a new toggle); code paths multiply per flag — test matrix explodes; runtime changes mean behavior can differ from what code review approved.
Use it when: genuinely operator-tunable behavior. Avoid it when: it's a developer decision wearing a settings costume.
Interview angle: "Feature flags vs configuration vs code?" — use lockout settings as the example of legitimate operator config.
Drill: find every setting consulted inside `AddToCartAsync` and its callers. How many distinct behaviors can an admin toggle on that one path?

## Pattern 11: Cache-Aside with Composed Keys and Prefixes

Problem it solves: read-heavy catalog pages need caching whose invalidation stays tractable.
General shape: `IStaticCacheManager.GetAsync(cacheKey, acquireFunc)` — try cache, else run acquire and store; keys built from typed defaults + parameters; prefixes group keys for bulk eviction.
Real example: [MemoryCacheManager.GetAsync#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132); key preparation in [PlaceOrderAsync L1695-L1698](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1695-L1698)
Second example: repository-level cached reads [EntityRepository.GetByIdAsync#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122) with `Func<ICacheKeyService, CacheKey>` parameter
Why this implementation works: the acquire-delegate shape makes "cached" a one-line decision at each call site; prefix eviction pairs with Pattern 4.
Failure modes: cache stampede on hot misses (many threads run acquire concurrently) unless locked; memory-cache per node → cross-node staleness in load-balanced setups unless the [DistributedCacheManager](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs) is configured.
Use it when: reads dominate and slight staleness is tolerable. Avoid it when: correctness-critical reads (permissions checks are cached here — worth knowing).
Interview angle: "Explain cache-aside and its failure modes" — stampede, staleness, invalidation; all three are visible in this repo.
Drill: does `MemoryCacheManager.GetAsync` protect against stampede? Read [MemoryCacheManager.cs#L132-L175](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) and answer with evidence.

## Pattern 12: FluentValidation at the MVC Boundary

Problem it solves: input-shape validation (required, length, format) declared once per model, executed automatically at model binding.
General shape: `class XxxValidator : BaseNopValidator<XxxModel>` with `RuleFor` chains; registered by convention.
Real example: [LoginValidator.cs#L9-L34](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9-L34) (conditional rules for email vs phone login)
Second example: any of the ~8 validators in [Validators/Customer](../../src/Presentation/Nop.Web/Validators/Customer)
Why this implementation works: keeps controllers free of `if (string.IsNullOrEmpty...)` noise; localized messages built in; testable in isolation.
Failure modes: it validates *shape*, not *business rules* — juniors see a validator exists and assume "validated"; the deep rules live in services (Pattern 2). Two-layer validation is the invariant: **shape at the edge, semantics in the domain**.
Use it when: any bound model. Avoid: duplicating business rules into validators (they can't see the DB consistently).
Interview angle: "Where do you put validation?" — the two-layer answer, with this repo as evidence for each layer.
Drill: find where `RegisterValidator` ends and `CustomerRegistrationService` business checks begin. List one rule that appears at each layer and one that could *only* live in the service.

## Pattern 13: Versioned, Attribute-Dated Migrations

Problem it solves: schema evolution across thousands of independently-hosted installations that upgrade at their own pace.
General shape: FluentMigrator classes tagged `[NopSchemaMigration("yyyy-MM-dd HH:mm:ss", description)]`, grouped in `UpgradeTo{version}` folders, forward-only.
Real example: [PriceListMigration.cs#L7-L20](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7-L20) — `ForwardOnlyMigration`, `CreateTableIfNotExists<T>()`
Second example: folder progression [Migrations/UpgradeTo440 … UpgradeTo500](../../src/Libraries/Nop.Data/Migrations); schema-from-code builders like [ProductBuilder.cs#L9-L17](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L9-L17)
Why this implementation works: timestamp ordering is merge-friendly (no sequential-integer collisions); `IfNotExists` makes re-runs tolerant; forward-only reflects reality (nobody down-migrates a live store).
Failure modes: forward-only means rollback = restore-from-backup — migrations must be *additive first* (add column, backfill, later drop); data migrations mixed with schema can lock big tables.
Use it when: you don't control when consumers upgrade. Avoid destructive changes in the same release that introduces the replacement.
Interview angle: "How do you do zero-downtime schema changes?" — expand/contract, and this repo shows the expand discipline.
Drill: read the UpgradeTo500 folder. Classify each migration: pure schema, pure data, or mixed. Which one would you be most nervous to run on a 10M-row table?

## Pattern 14: Convention-Based Startup Modules & Plugins

Problem it solves: 33 plugin projects must register services, routes, and pipeline steps without editing core code.
General shape: any assembly can contain `INopStartup` implementations discovered by reflection at boot, ordered by an `Order` property; plugins ship as separate projects loaded at runtime.
Real example: [NopStartup.cs#L67 + Order at L336](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67) (the core module registering ~200 services); orchestration from [Program.cs](../../src/Presentation/Nop.Web/Program.cs) via `ConfigureApplicationServices`
Second example: any plugin's startup, e.g. under [src/Plugins/Nop.Plugin.Payments.PayPalCommerce](../../src/Plugins/Nop.Plugin.Payments.PayPalCommerce)
Why this implementation works: open/closed at the deployment level — features arrive as folders, not core diffs; ordering gives deterministic pipeline assembly.
Failure modes: reflection-based discovery hides the registration graph (nothing to ctrl-click); a plugin can re-register/override core services — powerful and dangerous; boot cost scales with assembly scanning.
Use it when: third parties extend your app. Avoid it when: a single team owns everything — explicit registration is more debuggable.
Interview angle: "How would you design a plugin system?" — discovery, registration, ordering, isolation, and versioning; this repo demonstrates the first three.
Drill: find how plugins override a core service (search a plugin for `AddScoped` re-registration). What determines which registration wins?

## Pattern 15: Named Mutex + Sync-over-Async (Deliberate Tradeoff)

Problem it solves: prevent double order submission by the same customer racing through checkout twice.
General shape: `new Mutex(false, customerId)` held across order placement, with `.Result`/`.Wait()` because a `Mutex` is thread-affine and cannot cross `await`.
Real example: [OrderProcessingService.cs#L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718) — read the in-code comment at L1685-L1688; it explains *why* (mutex + await incompatibility, semaphore issues on UNIX).
Second example: the async-friendly alternative exists in-repo: [MemoryCacheLocker / DistributedCacheLocker](../../src/Libraries/Nop.Core/Caching/DistributedCacheLocker.cs)
Why this implementation works: correct on a single node, opt-in via `PlaceOrderWithLock` setting, scoped tightly to one customer's checkout.
Failure modes: **single-process only** — load-balanced deployments get no protection; sync-over-async risks thread-pool starvation under load; `.Result` wraps exceptions in `AggregateException`.
Use it when: single-node, last-resort dedup. Prefer: idempotency keys (the `OrderGuid` at [L1598](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598) is halfway there) or a distributed lock.
Interview angle: gold for "tell me about a tradeoff you disagreed with but understood." You can cite the authors' own comment as the constraint that forced it.
Drill: rewrite this block (on paper) using `SemaphoreSlim.WaitAsync` per customer id. What new problem does your version introduce? (Hint: unbounded dictionary of semaphores.)

## Pattern 16: Everything `virtual`, Everything `partial`

Problem it solves: thousands of merchants customize behavior without forking — they subclass a service, override one method, re-register it in DI.
General shape: `public partial class` + `public virtual` on essentially every service method and controller action.
Real example: every method in [ShoppingCartService.cs](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549) and every action in [ProductController.cs#L129](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129)
Second example: DI re-registration point [NopStartup.cs#L204](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L204) — replace the binding, keep the interface.
Why this implementation works: it is the product strategy — nopCommerce sells customizability; `virtual` is the extension contract.
Failure modes: every method is a public API → nothing is safe to refactor without breaking someone; overriders inherit private assumptions (call order, cache expectations) that aren't documented; "fragile base class" at ecosystem scale.
Use it when: extensibility is the product. Avoid it when: you own all callers — default to sealed/internal and open seams deliberately.
Interview angle: "What does making a method public/virtual *cost*?" — this repo is the extreme case; articulating the cost is a senior signal.
Drill: pick any protected helper in `OrderProcessingService` (e.g. `SaveOrderDetailsAsync` [L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723)). List two invariants an overrider could break that the compiler would never catch.
