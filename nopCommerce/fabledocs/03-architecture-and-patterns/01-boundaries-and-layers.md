# Boundaries and Layers

A **boundary** is a line where responsibility changes owners and assumptions must be re-checked. A layer is a boundary stacked horizontally. This repo's layers, what each owns, must not own, what actually enforces each rule — and where the leaks are.

## The layer contract

| Layer | Owns | Must NOT own | Actually enforced by |
| --- | --- | --- | --- |
| `Nop.Core` | entities, cache/event abstractions, infrastructure primitives | business rules, data access | project references (Core references no project) — but it *does* reference ASP.NET Core MVC packages |
| `Nop.Data` | query execution, schema, migrations, entity events | business decisions | references Core only ([Nop.Data.csproj](../../src/Libraries/Nop.Data/Nop.Data.csproj)) |
| `Nop.Services` | ALL business logic, orchestration | view models, rendering | references Data + Core ([Nop.Services.csproj, lines 32–33](../../src/Libraries/Nop.Services/Nop.Services.csproj)); **HTTP types are reachable** (see below) |
| `Nop.Web.Framework` | MVC plumbing: filters, DI wiring, work context | domain rules | — |
| `Nop.Web` | controllers, factories, view models, views | direct DB access, business rules | convention only |

**Correction to a popular claim** (an earlier version of this page made it): it is *not* true that "`Nop.Services` cannot mention `IActionResult` because the assembly doesn't reference MVC". [Nop.Core.csproj](../../src/Libraries/Nop.Core/Nop.Core.csproj) references `Microsoft.AspNetCore.Mvc.NewtonsoftJson` and `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` (lines 19–20), so MVC types flow transitively into every project above Core. And the service layer uses them: `ICustomerRegistrationService.SignInCustomerAsync` returns `Task<IActionResult>` ([ICustomerRegistrationService.cs#L62](../../src/Libraries/Nop.Services/Customers/ICustomerRegistrationService.cs#L62)); `IProductAttributeParser.ParseProductAttributesAsync` takes an `IFormCollection` ([IProductAttributeParser.cs#L147](../../src/Libraries/Nop.Services/Catalog/IProductAttributeParser.cs#L147)). `grep -rln "IFormCollection\|IActionResult\|HttpContext" --include=*.cs src/Libraries/Nop.Services | wc -l` returned **28** files on 2026-10-06; even `Nop.Core` has 6 files with `using Microsoft.AspNetCore…` (e.g. `Http/Extensions/HttpRequestExtensions.cs`, `Infrastructure/INopStartup.cs`).

So the compiler enforces **direction** (Services cannot reference Web, Data cannot reference Services) but not **HTTP-agnosticism**. The weakest wall is still Web→Services: nothing but convention stops a controller from composing raw queries or embedding business rules.

## Good boundaries, with receipts

- **Controller/service split done right:** [ShoppingCartController.AddProductToCart_Details](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791-L874) does HTTP things (form parsing, JSON shapes) and delegates decisions to [AddToCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549) / `UpdateShoppingCartItemAsync` via its `SaveItemAsync` helper ([ShoppingCartController.cs#L305](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L305)). The test of a real boundary is *a second consumer arrives and nothing has to move* — this one mostly passes (see leak 4 for the part that doesn't). Note: the in-repo [WebApi.Frontend plugin](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend) is only a configuration shell (four `.cs` files, 128 lines), so it is *not* evidence of a second consumer in this tree.
- **Entity/view-model wall:** factories ([ProductModelFactory#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502)) are the only place entities become models; views never see entities. This protects the DB shape from becoming a UI contract — the #1 coupling disease in CRUD apps.
- **Provider boundary:** three databases (MS SQL, MySQL, PostgreSQL — [DataProviders/](../../src/Libraries/Nop.Data/DataProviders)) + SQLite-for-tests ([SqLiteNopDataProvider.cs](../../src/Tests/Nop.Tests/SqLiteNopDataProvider.cs)) behind [INopDataProvider](../../src/Libraries/Nop.Data/INopDataProvider.cs) — an abstraction earning its keep (contrast with interfaces that merely mirror one class).

## Boundary leaks worth studying

1. **`IQueryable` through the repository.** [EntityRepository.Table#L525](../../src/Libraries/Nop.Data/EntityRepository.cs#L525) hands services raw query composition. Pragmatic (services can build efficient queries) but the repository's guarantees — caching, soft-delete filtering, even entity events for predicate deletes ([#L497-L506](../../src/Libraries/Nop.Data/EntityRepository.cs#L497-L506)) — no longer hold on that path. A boundary with a documented hole is livable; the skill is *knowing the hole is there*.
2. **Events published from the data layer.** [EntityRepository](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L349) publishes app-level events after each write — Data reaching "up" conceptually. The system buys automatic cache invalidation; the cost is that data-layer writes now run arbitrary consumer code synchronously ([EventPublisher](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52)) — a write can be slowed by any plugin's consumer.
3. **Writes that skip the repository.** `INopDataProvider` is also injected straight into services. `AclService.SaveAclAsync` and `StoreMappingService.SaveStoreMappingsAsync` flip the entity's flag with `_dataProvider.UpdateEntityAsync(entity)` ([AclService.cs#L312-L316](../../src/Libraries/Nop.Services/Security/AclService.cs#L312-L316), [StoreMappingService.cs#L266-L270](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L266-L270)) — no `EntityUpdatedEvent<Product>` is published for that change, so leak 2's cache-invalidation guarantee doesn't apply. Same entity, two write paths, two sets of side effects.
4. **Business rules in controllers (occasional).** The product-match check at [ShoppingCartController.cs#L842-L849](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L842-L849) is a domain rule living in the controller — any second consumer must remember to duplicate it. Conversely, the "only simple products" rule is checked **twice**: in the controller ([L803-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L803-L810), hard-coded English) and in the service ([ShoppingCartService.cs#L351-L352](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L351-L352), also hard-coded English). Duplication between layers is a smell; a gap is a vulnerability; this file has one of each.
5. **Ambient work context.** [WebWorkContext](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) gives *every* service access to "current customer/store" — convenient, but it means service behavior can depend on HTTP-ish ambient state. That is why the test base class must supply a fake `HttpContext`: it registers the real `WebWorkContext` ([BaseNopTest.cs#L473](../../src/Tests/Nop.Tests/BaseNopTest.cs#L473)) but mocks `IHttpContextAccessor` to return a `DefaultHttpContext` ([BaseNopTest.cs#L234-L241](../../src/Tests/Nop.Tests/BaseNopTest.cs#L234-L241)).
6. **Service contracts that return HTTP results.** Covered above: `SignInCustomerAsync` returning `IActionResult` decides the delivery mechanism inside the business layer.

## Feature trace: add-to-cart, invariant by invariant

Which layer checks what, for `POST AddProductToCart_Details`:

| Invariant | Controller | Service | Notes |
| --- | --- | --- | --- |
| product exists | [L793-L800](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L793-L800) (redirect JSON) | `ThrowIfNull` [L1557](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1557) | controller handles the user case, service the programmer case — correct split |
| simple product only | [L803-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L803-L810) | [L351-L352](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L351-L352) | duplicated |
| cart item belongs to this product | [L842-L849](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L842-L849) | — | controller only |
| cart item belongs to this customer | implicit: lookup within current customer's cart [L828-L831](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L828-L831) | — | ownership by construction, not by check |
| deleted / published / ACL / store | — | [L340-L360](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L340-L360) | service only — note the controller loads deleted products happily (`GetByIdAsync` defaults `includeDeleted = true`, [EntityRepository.cs#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122)) |
| cart / wishlist permission, bots | — | [L1560-L1576](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560-L1576) | service only |

Reading the table as a reviewer: the service owns most domain invariants (good), the controller owns two that a different consumer would need (leak 4), and one invariant (ownership) is enforced by *how the data was fetched* rather than by an explicit check — fine today, fragile if someone refactors the lookup.

## Exercise ladder

1. **Goal:** verify the project-reference direction.
   **Check:** open the three library csproj files and list every `ProjectReference`; draw the graph. Expected: Core → none; Data → Core; Services → Core, Data.
2. **Goal:** disprove "Services can't see MVC".
   **Check:** run the 28-file grep above, open [CustomerRegistrationService.cs#L456](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L456), and identify which package reference makes `IActionResult` resolvable.
3. **Goal:** find a write path that bypasses entity events.
   **Check:** `grep -rn "_dataProvider.UpdateEntityAsync" src/Libraries/Nop.Services --include=*.cs` — 5 hits on 2026-10-06 (3 in installation code, 1 in `AclService`, 1 in `StoreMappingService`). For the two non-installation hits, name which cache entry could go stale.
4. **Goal (mid):** move the product-match rule into the service without breaking plugins.
   **Check:** your design keeps `AddToCartAsync`/`UpdateShoppingCartItemAsync` signatures unchanged, adds the check where the update path already has `shoppingCartItem` in hand, and lists who might notice (anyone overriding `UpdateShoppingCartItemAsync`).
5. **Goal (mid):** find another business rule in a controller.
   **Check:** search `Nop.Web/Controllers` for `return Json(new` blocks with `success = false` and a hard-coded English message; for one hit, apply the second-consumer test and decide move/leave.

## Senior finding: the HTTP-agnostic wall is convention, not compiler

Verified in the csproj files and source above. Because `Nop.Core` references ASP.NET Core MVC packages, the "business layer has no HTTP dependency" property is not enforced anywhere; it is kept (mostly) by review. Evidence of erosion: 28 service files mention HTTP types, and at least one public service interface returns `IActionResult`. Why it matters: (a) background tasks and tests that call these services must fabricate HTTP state (leak 5); (b) a non-MVC consumer cannot reuse those services without adapting MVC results; (c) a reviewer who believes the wall is compiler-backed will not look for violations. A cheap guardrail is an architecture test (e.g. a reflection test in `Nop.Tests` asserting that no public `Nop.Services` interface method returns a type from `Microsoft.AspNetCore.Mvc`) — which also makes a good first contribution ([06-contribution-practice/01-good-first-tickets.md](../06-contribution-practice/01-good-first-tickets.md)).

## Debugging scenario: "a store-restricted product still shows in the other store's category list"

*Report:* after restricting a product to store A in admin, store B's category page still lists it for a while; the product *detail* page in store B correctly returns 404.

1. **Notice the asymmetry.** Detail and list take different paths: the detail page checks `StoreMappingService.AuthorizeAsync` on a loaded entity ([ProductController.cs#L141](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L141)); category listings use cached query results.
2. **Find the write.** The admin `Edit` saves the product via the repository (event published, by-id cache evicted) at [Admin ProductController.cs#L1253](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1253), and only later saves store mappings ([L1290](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1290)), whose flag write skips the repository (leak 3). The `StoreMapping` row inserts *do* go through a repository and publish `StoreMapping` events — so the question becomes "which caches does the `StoreMapping` cache consumer clear, and does the category-listing cache key depend on them?"
3. **Read, don't guess.** Open the `StoreMapping` cache event consumer under `Nop.Services/Stores/Caching` and the cache key used by the category product query; if the listing key isn't cleared by any event this edit produces, you've found the staleness and its bound (the key's cache time).
4. **Close the loop.** Either publish an `EntityUpdatedEvent` for the entity whose flag changed, or route the flag write through the repository. Both are boundary repairs: make the second write path behave like the first.

(Steps 3–4 are the investigation plan; the exact listing cache key was not traced for this page.)

## Transferable test

For any repo, ask per layer: (1) what would a second consumer of this layer need? (2) what compiler/tooling *enforces* the rule vs mere convention? (3) where are the documented holes? Interview phrasing: "boundaries are where I check invariants and where I expect bugs when two teams' assumptions meet."

Drill: find one more business rule living in a controller in `Nop.Web/Controllers` and write the one-sentence case for moving it (or leaving it). Basic: found one. Solid: correct move/leave call with the second-consumer test. Strong: also names the blast radius of moving it (which plugins/overriders might notice).

## Verification

2026-10-06: every anchor opened and checked. Corrected two claims from the earlier version: (1) "Nop.Services has no ASP.NET reference / cannot mention `IActionResult`" — false, see `Nop.Core.csproj` lines 19–20 and `ICustomerRegistrationService.cs` L62; (2) "the WebApi plugin bolts a REST API onto the same service" — the in-repo plugin contains no API endpoints. Also corrected "BaseNopTest must fake a work context" to the accurate "registers the real `WebWorkContext` and fakes the `HttpContext`". File counts from `grep -rln`.
