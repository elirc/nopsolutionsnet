# C# / .NET Deep-Dive Question Cards

Sixteen cards for the language/runtime round, each anchored to a place this repo actually exercises the concept. Practice pattern: read the anchor first, then answer aloud in ≤90 seconds, then check yourself against the mid/senior lines.

(The prompt template calls this file "js-ts-node"; this repo's stack is C#/ASP.NET Core, so the round is adapted accordingly. If you also interview on JS, map each concept: async/await ↔ event loop & promises, DI lifetimes ↔ module scope/singletons, IQueryable ↔ lazy generators.)

---

## Q1: What actually happens when you `await` something?

Round: C#/.NET deep-dive
What it's really testing: do you know async is about *freeing threads*, not spawning them.
Repo anchor: [`ShoppingCartService.AddToCartAsync`](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549) — a typical chain: controller awaits service awaits repository awaits driver.
Junior answer sounds like: "It waits for the task to finish without blocking."
Mid-level answer adds: the compiler rewrites the method into a state machine; at a genuinely asynchronous point the thread is *returned to the pool* and the continuation resumes later — that's why one server handles thousands of concurrent requests with few threads. ASP.NET Core has no `SynchronizationContext`, so continuations resume on any pool thread.
Senior answer includes: awaiting already-completed tasks continues synchronously (fast path); `ConfigureAwait(false)` matters in libraries targeting UI/legacy frameworks but is largely noise in ASP.NET Core; the cost model — each `await` allocates when the task is incomplete, which is why hot paths sometimes use `ValueTask`.
Likely follow-ups: "What happens if you never await a returned Task?" (exception observed by nobody, fire-and-forget hazard.)
Practice drill: trace one request through controller → service → repository and count the await hops.

## Q2: What's wrong with `.Result` / `.Wait()` — and is it ever justified?

Round: C#/.NET deep-dive
What it's really testing: deadlock/starvation understanding and judgment about real-world exceptions to the rule.
Repo anchor: [`OrderProcessingService.PlaceOrderAsync#L1689-L1716`](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1689-L1716) — `.Result` at L1698/L1707, `.Wait()` at L1710, with the authors' comment (L1685-L1688) explaining why: a named `Mutex` is thread-affine and cannot be released across an `await`.
Junior answer sounds like: "Blocking is bad, always use await."
Mid-level answer adds: blocking a pool thread while it waits for work that *needs* pool threads risks starvation; in legacy contexts with a SynchronizationContext it's a classic deadlock. Exceptions surface wrapped in `AggregateException`.
Senior answer includes: this repo's block is a *deliberate documented tradeoff* forced by OS-lock semantics — and the stronger fix is removing the need for the lock (idempotency key + unique constraint) rather than a cleverer lock. Being able to cite real production code that breaks the rule *with reasons* is exactly what this question is fishing for.
Likely follow-ups: "How would you redesign it?"
Practice drill: explain to a rubber duck why `SemaphoreSlim.WaitAsync` would fix the await problem but create a semaphore-lifetime problem.

## Q3: Explain DI lifetimes. Why is nearly everything here Scoped?

Round: C#/.NET deep-dive
What it's really testing: whether you've been burned by captive dependencies.
Repo anchor: [`NopStartup.cs#L195-L215`](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215) — a wall of `AddScoped`; note Program.cs sets `ValidateScopes = false` with a comment explaining they resolve scoped services from the root during startup ([Program.cs](../../src/Presentation/Nop.Web/Program.cs)).
Junior answer sounds like: definitions of transient/scoped/singleton.
Mid-level answer adds: scoped = one instance per HTTP request — the right default for services holding per-request state (current customer, store context like [WebWorkContext](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196)); a singleton capturing a scoped service ("captive dependency") silently freezes one request's state forever.
Senior answer includes: why `ValidateScopes=false` here is a risk trade (startup-time resolution convenience vs losing the safety net); cache managers are effectively singleton-ish state while data access is scoped — mixing them is where lifetime bugs live.
Likely follow-ups: "What breaks if IShoppingCartService were singleton?"
Practice drill: find one service whose scoped-ness is load-bearing and explain the bug if it weren't.

## Q4: IQueryable vs IEnumerable — where does the query actually run?

Round: C#/.NET deep-dive
What it's really testing: deferred execution and accidental client-side evaluation.
Repo anchor: [`EntityRepository.Table`](../../src/Libraries/Nop.Data/EntityRepository.cs#L525) exposes `IQueryable<TEntity>`; services compose LINQ against it and materialize via the repository's `GetAll*` methods.
Junior answer sounds like: "IQueryable is for databases."
Mid-level answer adds: `IQueryable` builds an expression tree translated to SQL at enumeration; calling `.AsEnumerable()`/`.ToList()` early drags the whole table into memory and runs the rest in C#. The filter you add *after* materialization doesn't reach the database.
Senior answer includes: the repository's design tension — `Table` is an escape hatch that bypasses the caching/eventing guarantees of the typed methods; also expression-tree limits (can't translate arbitrary C# methods) and how that produces runtime, not compile-time, failures.
Likely follow-ups: "How would you find accidental client-side evaluation in prod?" (query logging, slow-query review.)
Practice drill: find one service method composing on `Table` and state exactly which line triggers SQL execution.

## Q5: Generic constraints — what does `where TEntity : BaseEntity, IStoreMappingSupported` buy?

Round: C#/.NET deep-dive
What it's really testing: can you use the type system as a design tool, not just syntax.
Repo anchor: [`StoreMappingService.AuthorizeAsync<TEntity>#L204-L221`](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221); the capability-interface stack on [`Product.cs#L13`](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13).
Junior answer sounds like: "It restricts what types you can pass."
Mid-level answer adds: one method authorizes *any* entity that declares the capability — compile-time proof that you can't ask "which stores can see this?" of an entity that has no store mapping. Composition over inheritance, enforced by the compiler.
Senior answer includes: the failure mode — capabilities are opt-in, so *forgetting* the interface on a new entity doesn't fail, it silently means "no store isolation." Type systems catch wrong calls, not missing declarations; that gap is closed by review checklists, not compilers.
Likely follow-ups: "Interface vs abstract base for this?"
Practice drill: sketch the signature for a generic `SoftDeleteAsync<T>` and its constraint.

## Q6: Why would a codebase make every method `virtual`? What does it cost?

Round: C#/.NET deep-dive
What it's really testing: API-surface thinking; do you understand that accessibility is a *commitment*.
Repo anchor: any service — e.g. every method of [`ShoppingCartService`](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549) is `public virtual`, and partial classes throughout.
Junior answer sounds like: "So you can override for testing."
Mid-level answer adds: it's the extensibility contract — customers subclass and re-register in DI ([NopStartup.cs#L204](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L204)); the cost is that every method becomes public API: internal refactors become breaking changes for the plugin ecosystem.
Senior answer includes: fragile-base-class risk (overriders depend on undocumented call order/invariants); the modern alternative — sealed by default with deliberate seams (interfaces, events, options) — and why a *product* whose feature is customizability rationally chooses the opposite.
Likely follow-ups: "How would you version such an API?"
Practice drill: pattern card 16 in [the catalog](../03-architecture-and-patterns/05-pattern-catalog.md) has the drill; do it aloud.

## Q7: Exceptions vs result objects — where's the line?

Round: C#/.NET deep-dive
What it's really testing: error-handling philosophy with concrete criteria.
Repo anchor: warnings list in [`AddToCartAsync#L1559-L1582`](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582) vs `throw` for broken preconditions at [L1555-L1557](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555-L1557) (`ArgumentNullException.ThrowIfNull`).
Junior answer sounds like: "Exceptions for errors."
Mid-level answer adds: the rule this repo demonstrates in one method — *expected domain outcomes* (out of stock, cart disabled) are data the UI must render, so they're return values; *programmer errors* (null customer) are exceptions. Exceptions also cost: stack unwinding, and control flow invisible in signatures.
Senior answer includes: critique — stringly-typed warnings can't be branched on reliably; typed error enums or a Result<T, Error> improve it; and the observability angle: result-object failures need explicit logging because nothing throws.
Likely follow-ups: "How does this interact with transactions/rollback?"
Practice drill: classify five failure cases from checkout as exception vs result, and defend each.

## Q8: How do interfaces enable the testing strategy here?

Round: C#/.NET deep-dive
What it's really testing: seams and test design, not mocking trivia.
Repo anchor: [`IRepository<T>`](../../src/Libraries/Nop.Data/IRepository.cs) as the seam; but note the tests mostly *don't* mock it — [`BaseNopTest`](../../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L116) boots a real SQLite DB with real services.
Junior answer sounds like: "Interfaces let you mock dependencies."
Mid-level answer adds: this repo chose *integration-style* tests over mocks — real DI container, real SQLite, real seed data — because the value of e-commerce logic is in the interaction of services, and mock-heavy tests would just restate the implementation.
Senior answer includes: the tradeoff matrix — integration tests catch real regressions and survive refactors but are slower and share state; unit-with-mocks pin behavior precisely but ossify design. Naming *when each wins* is the senior move.
Likely follow-ups: "How would you test the mutex block in PlaceOrderAsync?" (You largely can't in-process; test the behavior it protects via idempotency assertions.)
Practice drill: read one test in [`ProductServiceTests`](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L63) and say what it would catch that a mocked version wouldn't.

## Q9: Extension methods — where do they help and where do they hide?

Round: C#/.NET deep-dive
What it's really testing: readability judgment.
Repo anchor: [`this.CreateTableIfNotExists<PriceList>()`](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L15) — migration DSL built from extensions in [Nop.Data/Extensions](../../src/Libraries/Nop.Data/Extensions); also the entire startup pipeline (`ConfigureApplicationServices`, `ConfigureRequestPipeline` in [Program.cs](../../src/Presentation/Nop.Web/Program.cs)) is extension methods.
Junior answer sounds like: "They add methods to types you don't own."
Mid-level answer adds: they build fluent DSLs and keep call sites clean, but they hide *where behavior lives* — `app.ConfigureRequestPipeline()` looks like one line; it's an entire subsystem in [ApplicationBuilderExtensions](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs). Static resolution: no polymorphism, no mocking.
Senior answer includes: discoverability cost for new team members and the convention that makes it work here (all in `Infrastructure/Extensions` folders); when a real type/service beats an extension (state, substitutability).
Likely follow-ups: "Why can't you override an extension method?"
Practice drill: open `ApplicationBuilderExtensions` and list, in order, five things the pipeline registers.

## Q10: How does this app discover and load plugins at runtime? What are the risks?

Round: C#/.NET deep-dive
What it's really testing: reflection, assembly loading, and open/closed design.
Repo anchor: `INopStartup` implementations discovered and ordered — core module [NopStartup.cs#L67 + Order L336](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L67); 32 plugin projects under [src/Plugins](../../src/Plugins).
Junior answer sounds like: "Reflection scans assemblies for a marker interface."
Mid-level answer adds: discovery + `Order` property = deterministic composition; plugins can *re-register* core services to override behavior — the DI container's "last registration wins" becomes the extension mechanism.
Senior answer includes: risks — nothing to ctrl-click (hidden control flow), version skew between plugin and core (the `virtual`-everywhere contract from Q6 is what absorbs it), boot-time cost, and trust (a plugin runs with full app privileges).
Likely follow-ups: "How would you sandbox plugins?" (Honest answer: in-process you mostly can't; process isolation or scoped APIs.)
Practice drill: find a plugin's startup class and identify one thing it adds and one thing it could accidentally break.

## Q11: Why `decimal` for money? What breaks with `double`?

Round: C#/.NET deep-dive
What it's really testing: numeric representation basics that cost real money.
Repo anchor: `decimal customerEnteredPrice` through [`AddToCartAsync#L1551`](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1551); all totals in `OrderProcessingService`.
Junior answer sounds like: "decimal is more precise."
Mid-level answer adds: `double` is base-2 — 0.1 isn't representable, sums of prices drift, comparisons fail; `decimal` is base-10 with 28-29 significant digits, exact for currency arithmetic at ~10x the CPU cost (irrelevant next to a DB call).
Senior answer includes: rounding policy is the *real* problem — where does the system round (per line? per order? per tax jurisdiction?) and is it consistent between app and gateway; also currency as a type concern (amount without currency code is a bug waiting).
Likely follow-ups: "How do you store money in SQL?" (decimal(18,4)-style; nopCommerce's schema does.)
Practice drill: compute 0.1 + 0.2 in C# `double` and `decimal` in a REPL; say why the difference matters at checkout.

## Q12: `Mutex` vs `lock` vs `SemaphoreSlim` — when each?

Round: C#/.NET deep-dive
What it's really testing: concurrency primitive selection with async awareness.
Repo anchor: named Mutex at [`OrderProcessingService.cs#L1689`](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1689); async-friendly alternatives in [MemoryCacheLocker](../../src/Libraries/Nop.Core/Caching/MemoryCacheLocker.cs) / [DistributedCacheLocker](../../src/Libraries/Nop.Core/Caching/DistributedCacheLocker.cs).
Junior answer sounds like: "lock for threads, Mutex for processes."
Mid-level answer adds: `lock`/`Monitor` — cheapest, single-process, cannot span `await`; `SemaphoreSlim.WaitAsync` — the async-compatible choice; `Mutex` — kernel object, cross-process, named — and thread-affine, hence this repo's sync-over-async workaround; none of them span *machines*, which is why a distributed lock (cache/DB-based) exists in the same repo.
Senior answer includes: the design smell reading — needing a lock across an external call often means the *protocol* is wrong (make the operation idempotent instead); lock granularity here is per-customer-id, which is why it's cheap.
Likely follow-ups: "C# 13 `Lock` type? Can you `await` inside `lock`?" (No — compiler error; that's the whole story of this anchor.)
Practice drill: whiteboard the per-customer `SemaphoreSlim` dictionary version and its cleanup problem.

## Q13: Delegates as parameters — what is `Func<ICacheKeyService, CacheKey>` doing in a repository signature?

Round: C#/.NET deep-dive
What it's really testing: functional composition in C#, caching API design.
Repo anchor: [`GetByIdAsync(int? id, Func<ICacheKeyService, CacheKey> getCacheKey = null, ...)`](../../src/Libraries/Nop.Data/EntityRepository.cs#L122); acquire-delegates in [`MemoryCacheManager.GetAsync#L132`](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132).
Junior answer sounds like: "It's a callback."
Mid-level answer adds: the caller *describes* the cache key without executing anything — the repository decides whether caching applies (null = default key, and the delegate receives the key service so keys stay centrally composed); the acquire-func pattern in `GetAsync(key, acquire)` means the fetch only runs on a miss.
Senior answer includes: closure-capture hazards (capturing loop variables or scoped services inside cached delegates), and why delegates beat inheritance for this: per-call-site variation without subclassing the repository.
Likely follow-ups: "What's the allocation cost of closures on a hot path?"
Practice drill: write (on paper) the call that caches "product by id" under a custom key, then the call that skips caching entirely.

## Q14: What does the GC mean for a long-running web app with a big in-memory cache?

Round: C#/.NET deep-dive
What it's really testing: practical memory literacy, not GC-internals trivia.
Repo anchor: [`MemoryCacheManager`](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L13) holding the catalog in process memory.
Junior answer sounds like: "GC frees unused objects automatically."
Mid-level answer adds: gen0/1/2 — short-lived request objects are cheap; a large cache lives in gen2/LOH, so full GCs get more expensive as the cache grows; cache entries also *pin* object graphs (a cached Product referencing services would leak them).
Senior answer includes: Server GC for throughput; memory pressure vs cache-size limits and eviction policy; when to move the cache out of process (Redis) — trading GC pressure and cross-node staleness for serialization cost and a network hop.
Likely follow-ups: "How would you diagnose a memory leak here?" (dotnet-counters, dotnet-gcdump, compare heap snapshots.)
Practice drill: name three object types you'd expect to dominate a heap snapshot of a busy store.

## Q15: Spot the N+1 — awaited calls inside loops

Round: C#/.NET deep-dive
What it's really testing: can you see the pattern in real code, not just define it.
Repo anchor: [`ShoppingCartController.ProductDetails_AttributeChange#L904-L918`](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918) — `foreach` over attribute values, awaiting `GetProductByIdAsync` per associated product.
Junior answer sounds like: "N+1 is many queries instead of one."
Mid-level answer adds: point at the loop — each iteration is a sequential awaited DB round-trip; fixes in order of preference: batch (`GetProductsByIdsAsync` — the repo has [`GetByIdsAsync`](../../src/Libraries/Nop.Data/EntityRepository.cs#L157)), join in the query, or cache. Note the mitigation already present: per-entity caching means iterations 2..N may be cache hits — measure before rewriting.
Senior answer includes: sequential-await vs `Task.WhenAll` (parallel awaits against a DB can just move the bottleneck to the connection pool); how you'd *detect* this in production (query counts per request, mini-profiler) rather than by reading every loop.
Likely follow-ups: "When is N+1 fine?" (N small and bounded, cache-hot, or code clarity dominates.)
Practice drill: [04-code-reading-gym/02-trace-tables.md](../04-code-reading-gym/02-trace-tables.md) trace 3 walks this exact code.

## Q16: Nullability — how does this codebase defend against null, and what would NRT change?

Round: C#/.NET deep-dive
What it's really testing: modern C# hygiene and migration judgment.
Repo anchor: `ArgumentNullException.ThrowIfNull(customer)` at [`AddToCartAsync#L1555`](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555); null-as-signal returns like `GetProductByIdAsync` → null → 404 at [`ProductController.cs#L131-L133`](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L131-L133).
Junior answer sounds like: "Check for null at the top of methods."
Mid-level answer adds: two distinct nulls here — *invalid input* (throw fast) vs *not found* (legitimate domain outcome flowing to a 404); nullable reference types would encode the second in signatures (`Task<Product?>`) so callers are forced to handle it.
Senior answer includes: retrofitting NRT onto a 20-year codebase is a migration project (annotate leaf libraries first, `#nullable enable` per file, watch for false confidence at reflection/ORM boundaries where the compiler can't see materialization).
Likely follow-ups: "`default!` and `null!` — when are they lies you accept?"
Practice drill: pick one service method and write its NRT-honest signature.
