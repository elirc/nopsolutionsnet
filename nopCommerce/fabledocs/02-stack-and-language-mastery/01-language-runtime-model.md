# Language & Runtime Model

Four mental models, from first principles, each pinned to code that depends on it — then a feature trace, an exercise ladder, two verified findings, and a debugging scenario that only makes sense once the models click.

## 1. Async = freeing threads, not creating them

Model: a web server owns a small pool of threads. `await` on real I/O parks the *method* (compiler-generated state machine) and returns the *thread* to the pool; the continuation resumes on any pool thread when the I/O completes. Throughput comes from threads never waiting.

Where this repo uses it well: the main request call graphs are async end-to-end — controller ([ProductController.cs#L129](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129)) → service → repository ([EntityRepository.cs#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122)) → driver.

Sharp edge: [PlaceOrderAsync's mutex block](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1689-L1716) uses `.Result`/`.Wait()` because an OS `Mutex` must be released by the acquiring thread and `await` may resume elsewhere. Deliberate, commented ([L1685-L1688](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1685-L1688)), single-node-only — and only active when the store setting `PlaceOrderWithLock` is on, which the installer sets to `false` ([InstallRequiredData.cs#L1888](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L1888)). Understand it, quote it, don't copy it.

**Correction to a common claim:** the mutex block is the *famous* sync-over-async site, not the only one. Running `grep -rnE "\)\.Result\b|\.Wait\(\)|GetAwaiter\(\)\.GetResult\(\)" --include=*.cs src/Libraries src/Presentation | grep -v /Migrations/` on 2026-10-06 returned **30 lines**. Some are blocking waits on a `SemaphoreSlim` rather than on tasks (e.g. [RedisConnectionWrapper.cs#L118](../../src/Libraries/Nop.Services/Caching/RedisConnectionWrapper.cs#L118)), but most block on tasks. The important ones to know:

| Site | Why it blocks | Runs how often |
| --- | --- | --- |
| [NopStartup.cs#L279](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L279), [#L282](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L282) | DI factories for every `ISettings` type must return synchronously, so they block on the current store and on `LoadSettingAsync` | once per settings type per request scope that resolves it |
| [RuleBuilderOptionsExtension.cs#L16](../../src/Presentation/Nop.Web.Framework/Validators/RuleBuilderOptionsExtension.cs#L16) | FluentValidation message factories are synchronous; `WithMessageAwait` blocks on the localized string | per failed validation rule |
| [NopRazorPage.cs#L27](../../src/Presentation/Nop.Web.Framework/Mvc/Razor/NopRazorPage.cs#L27) | Razor's `T("...")` localizer helper is synchronous | per localized string rendered |
| [LanguageParameterTransformer.cs#L59](../../src/Presentation/Nop.Web.Framework/Mvc/Routing/LanguageParameterTransformer.cs#L59) | routing parameter transformers are synchronous | per generated URL with a language segment |
| [SyncCodeHelper.cs#L194](../../src/Libraries/Nop.Services/Helpers/SyncCodeHelper.cs#L194) | an entire helper class ("User synchronous code helper") exists for call sites that cannot be async | caller-dependent |

The pattern behind the list: **every one of these is a framework callback with a synchronous signature** (DI factory, validator message, Razor helper, route transformer). That is the real lesson — sync-over-async usually appears where *someone else's* API is synchronous, not because the author forgot `await`. In ASP.NET Core there is no `SynchronizationContext`, so these don't deadlock the way classic ASP.NET did; the cost is a blocked pool thread for the duration of the wait, which only matters when the awaited work is slow (a cache miss that goes to the database).

Pitfall checklist: `.Result`/`.Wait()` on pool threads (starvation under load); `async void` (uncatchable exceptions); fire-and-forget tasks (lost failures — see the distributed cache below); sequential awaits in loops when work is independent ([the loop at ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918) awaits `GetProductByIdAsync` once per bundled attribute value).

Interview angle → cards [Q1, Q2, Q12](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 2. Memory: gen0 churn is free-ish, gen2 residents are rent

Model: request-scoped objects (models, warnings lists) die young — cheap. Long-lived objects (the in-memory cache) migrate to gen2 where collection is expensive; big arrays/strings go to the LOH.

Where it matters here: [MemoryCacheManager](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L13) keeps hot catalog data in process memory for `DefaultCacheTime` minutes (default 60, [CacheConfig.cs#L11](../../src/Libraries/Nop.Core/Configuration/CacheConfig.cs#L11)) — the app trades GC pressure for zero-latency reads.

Read its `GetAsync` carefully ([MemoryCacheManager.cs#L132-L147](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132-L147)) — three runtime ideas in fifteen lines:

1. `CacheTime <= 0` bypasses caching entirely (L134-L135).
2. The cached value is a `Lazy<Task<T>>` created with `isThreadSafe: true` (L142). Concurrent misses on the same key share *one* acquire call — built-in cache-stampede protection. The cache stores the *task*, so even the in-flight database query is shared.
3. A `null` result or an exception removes the entry (L150-L158), so failures are not cached.

Closures passed as acquire delegates can accidentally capture request-scoped objects into that long-lived cache — the classic .NET leak shape. The acquire lambda in `EntityRepository.GetByIdAsync` ([L127-L130](../../src/Libraries/Nop.Data/EntityRepository.cs#L127-L130)) captures `this` (the scoped repository) and `id`; it is safe only because the `Lazy` is evaluated immediately and the delegate is not kept after the value is produced — a subtle property worth being able to argue.

Pitfall checklist: capturing scoped services in cached delegates; unbounded caches; string concat in loops on hot paths; `IEnumerable` re-enumeration running queries twice.

Interview angle → card [Q14](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 3. Value vs reference, and why money is `decimal`

Model: structs copy, classes alias. `double` is binary floating point — 0.1 doesn't exist in it; `decimal` is base-10 exact for currency scales.

Where it matters here: every price and total is `decimal` (e.g. [AddToCartAsync's customerEnteredPrice, L1551](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1551)), and the schema maps `decimal` to `decimal(18,4)` by default ([FluentMigratorExtensions.cs#L34](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L34)). Entities are classes, so services mutate shared instances — note how [ValidateCustomerAsync](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160-L170) mutates the `customer` object then persists it; anyone holding that reference sees the mutation *before* the save.

**Reference semantics + an in-process cache = shared mutable state.** This is where model 2 and model 3 collide:

- With the default in-memory cache, `IStaticCacheManager` is a **singleton** `MemoryCacheManager` ([NopStartup.cs#L127-L129](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L127-L129)) that hands back the **same object instance** to every request that hits the key.
- With Redis or SQL Server distributed caching, the manager is **scoped** ([NopStartup.cs#L106-L113](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L106-L113)) and deserializes JSON on a miss in its local map ([DistributedCacheManager.cs#L111-L118](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L111-L118), [#L161-L190](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L161-L190)) — so each request scope gets its *own* copy.

Same code, two aliasing behaviours depending on configuration. Keep this in mind for the finding below.

Pitfall checklist: `double` for money; mutating cached entity instances (you just edited the cache for everyone); `==` on reference types comparing identity when you meant value.

Interview angle → card [Q11](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 4. Virtual dispatch as an extension contract

Model: `virtual` means the runtime picks the method by the object's actual type. Cheap at the machine level; expensive at the *organizational* level — every virtual public method is a promise to unknown overriders.

Where it matters here: everything is `public virtual` + `partial` ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)); the DI container is the swap point ([NopStartup.cs#L204](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L204)). This is the product's customization strategy, not a style quirk. Even `protected virtual` helpers like `GetStandardWarningsAsync` ([ShoppingCartService.cs#L330](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L330)) are override points, which is why refactoring their call order can break a plugin you have never seen.

Pitfall checklist: calling virtual methods from constructors; overriding without preserving base invariants; assuming your private helper's call order is yours to change.

Interview angle → card [Q6](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## Feature trace: the login POST, counted in awaits

[CustomerController.Login](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L463-L553) is a good place to *see* the async model because every step is I/O:

| Step | Line | Awaited I/O |
| --- | --- | --- |
| CAPTCHA message (only if invalid) | [L466-L467](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L466-L467) | localization lookup |
| Credential validation | [L485-L486](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L485-L486) → [CustomerRegistrationService.cs#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) | customer by email/username, role check, password record, possibly an update |
| Re-load customer on success | [L491-L493](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L491-L493) | a *second* lookup of the same customer |
| Sign-in | [L495-L496](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L495-L496) | cookie + activity writes |

The re-load at L491-L493 is a small, real inefficiency: `ValidateCustomerAsync` already loaded the customer but returns only an enum, so the controller fetches it again. It exists because the service's contract is "return a result code", a type-design decision (see [03-type-system-and-contracts.md](03-type-system-and-contracts.md)), not a runtime one.

## Exercise ladder

1. **Goal:** find the sync-over-async sites yourself.
   **Check:** run the grep above; you should get 30 lines. Classify each as *framework callback forced it* or *avoidable*. Expect most to be the former.
2. **Goal:** prove the in-memory cache returns shared instances.
   **Check:** read [MemoryCacheManager.cs#L137-L147](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L137-L147) and write one sentence explaining why two requests that hit the same key receive `ReferenceEquals`-identical objects (hint: the `Lazy<Task<T>>` is stored, and a completed task always returns the same result object).
3. **Goal:** explain stampede protection.
   **Check:** describe what happens when 50 requests miss `Product.byid.42` simultaneously. Your answer must mention `GetOrCreate` + `Lazy(…, true)` and conclude "one query". Then explain why the distributed manager needed its own `_ongoing` dictionary ([DistributedCacheManager.cs#L166](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L166)) to get the same property.
4. **Goal (mid):** identify a fire-and-forget task and its failure mode.
   **Check:** in [DistributedCacheManager.cs#L182-L194](../../src/Libraries/Nop.Core/Caching/DistributedCacheManager.cs#L182-L194) the write to Redis is started but not awaited; only a continuation that removes the `_ongoing` entry is attached. State what happens to an exception thrown by `SetStringAsync` (it is never observed; the item is simply not cached remotely).
5. **Goal (mid):** run locally — demonstrate thread blocking.
   **Check (run locally, outcome derived from code):** with `PlaceOrderWithLock` enabled, place two orders for the same customer concurrently; the second request's thread blocks in `mutex.WaitOne()` ([L1691](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1691)) until the first finishes, and then is rejected by the `MinimumOrderPlacementInterval` cache flag ([L1695-L1704](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1695-L1704)).

## Senior findings (verified in source)

**F1 — Admin edits mutate the shared cached instance before they are saved.** The admin product `Edit` POST loads the product with `GetProductByIdAsync` ([Admin ProductController.cs#L1192](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1192)), which uses the static cache ([ProductService.cs#L555-L558](../../src/Libraries/Nop.Services/Catalog/ProductService.cs#L555-L558)). It then maps the posted model *onto that instance* — `product = model.ToEntity(product)` ([L1231](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1231)) — and performs several awaited calls before `UpdateProductAsync` ([L1253](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1253)). Under the default in-memory cache (singleton, shared instances), storefront requests that hit the cached product during that window observe unsaved values (price, published flag, name). If anything throws before L1253, the cached instance stays modified while the database does not, until eviction or the 60-minute default expiry. Under Redis, the same code is safe across requests because each scope deserializes its own copy. Confidence: high on mechanism; the window is short in practice. The general rule this teaches: **never mutate an object you got from a shared cache; clone or re-read for write paths.**

**F2 — Blocking settings resolution sits on the request hot path.** Every settings class is materialized by a DI factory that blocks twice ([NopStartup.cs#L276-L283](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L276-L283)). On a warm cache this is cheap; on a cold start or after a cache clear, each of those `.Result` calls can wait on database I/O while holding a pool thread. Not a bug — a constraint of the container's synchronous factory API — but it is the first place to look if thread-pool starvation shows up right after deployments.

## Debugging scenario: "the storefront showed the new price before the admin saved"

*Report:* a merchandiser says customers briefly saw a price that was never saved — the admin got a validation/redirect error and the database still holds the old price.

1. **Reproduce the conditions:** is distributed caching enabled? If `DistributedCacheConfig.Enabled` is false, `IStaticCacheManager` is the singleton `MemoryCacheManager` ([NopStartup.cs#L125-L130](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L125-L130)). The bug class in F1 only exists in that mode.
2. **Find a path that mutates then exits without saving.** In the `Edit` POST, mapping happens at L1231 and the save at L1253; any exception between them leaves the cached instance changed. Also note the stock-quantity guard at [L1204-L1208](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1204-L1208) redirects *before* mapping, so it is not the culprit — good to rule out explicitly.
3. **Confirm with a probe (run locally):** set a breakpoint after L1231, load the product page in another browser, and observe the new price served from cache.
4. **Fix options, in order of blast radius:** (a) clear the product's cache key in a `catch`/`finally` on the write path; (b) load a fresh, uncached copy for edits (the repository's `GetByIdAsync` accepts `getCacheKey: null` to bypass caching — [EntityRepository.cs#L132-L133](../../src/Libraries/Nop.Data/EntityRepository.cs#L132-L133)); (c) make the memory cache return clones (large change, affects every caller and plugin).

## Verification

2026-10-06: all anchors opened and checked. The "only one sync-over-async site" framing in the previous version was corrected using the grep shown (30 matching lines). Cache lifetime facts (singleton memory manager vs scoped distributed managers) read from `NopStartup.cs` L98-L130. No code was built or executed; "run locally" rungs state outcomes derived from reading the code.
