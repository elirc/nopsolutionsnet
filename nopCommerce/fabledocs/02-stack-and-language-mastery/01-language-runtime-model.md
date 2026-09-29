# Language & Runtime Model

Four mental models, from first principles, each pinned to code that depends on it.

## 1. Async = freeing threads, not creating them

Model: a web server owns a small pool of threads. `await` on real I/O parks the *method* (compiler-generated state machine) and returns the *thread* to the pool; the continuation resumes on any pool thread when the I/O completes. Throughput comes from threads never waiting.

Where this repo uses it well: the entire call graph is async end-to-end — controller ([ProductController.cs#L129](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129)) → service → repository ([EntityRepository.cs#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122)) → driver. No blocking mixed in — with one famous exception.

Sharp edge: [PlaceOrderAsync's mutex block](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1689-L1716) uses `.Result`/`.Wait()` because an OS `Mutex` must be released by the acquiring thread and `await` may resume elsewhere. Deliberate, commented, single-node-only. Understand it, quote it, don't copy it.

Pitfall checklist: `.Result`/`.Wait()` on pool threads (starvation); `async void` (uncatchable exceptions); fire-and-forget tasks (lost failures); sequential awaits in loops when work is independent ([the loop at ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918)).

Drill: count await hops in the login flow ([CustomerController.cs#L463-L496](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L463-L496)). Interview angle → cards [Q1, Q2, Q12](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 2. Memory: gen0 churn is free-ish, gen2 residents are rent

Model: request-scoped objects (models, warnings lists) die young — cheap. Long-lived objects (the in-memory cache) migrate to gen2 where collection is expensive; big arrays/strings go to the LOH.

Where it matters here: [MemoryCacheManager](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L13) keeps the hot catalog in process memory — the app trades GC pressure for zero-latency reads. Closures passed as acquire delegates ([GetAsync#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132)) can accidentally capture request-scoped objects into that long-lived cache — the classic .NET leak shape.

Pitfall checklist: capturing scoped services in cached delegates; unbounded caches; string concat in loops on hot paths; `IEnumerable` re-enumeration running queries twice.

Interview angle → card [Q14](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 3. Value vs reference, and why money is `decimal`

Model: structs copy, classes alias. `double` is binary floating point — 0.1 doesn't exist in it; `decimal` is base-10 exact for currency scales.

Where it matters here: every price and total is `decimal` (e.g. [AddToCartAsync's customerEnteredPrice, L1551](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1551)); entities are classes, so services mutate shared instances — note how [ValidateCustomerAsync](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160-L170) mutates the `customer` object then persists it; anyone holding that reference sees the mutation *before* the save.

Pitfall checklist: `double` for money; mutating cached entity instances (you just edited the cache for everyone); `==` on reference types comparing identity when you meant value.

Interview angle → card [Q11](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## 4. Virtual dispatch as an extension contract

Model: `virtual` means the runtime picks the method by the object's actual type. Cheap at the machine level; expensive at the *organizational* level — every virtual public method is a promise to unknown overriders.

Where it matters here: everything is `public virtual` + `partial` ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)); the DI container is the swap point ([NopStartup.cs#L204](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L204)). This is the product's customization strategy, not a style quirk.

Pitfall checklist: calling virtual methods from constructors; overriding without preserving base invariants; assuming your private helper's call order is yours to change.

Interview angle → card [Q6](../08-interview-prep/01-csharp-dotnet-deep-dive.md).
