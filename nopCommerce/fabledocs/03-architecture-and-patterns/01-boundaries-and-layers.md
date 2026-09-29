# Boundaries and Layers

A **boundary** is a line where responsibility changes owners and assumptions must be re-checked. A layer is a boundary stacked horizontally. This repo's layers, what each owns, must not own, and where the leaks are.

## The layer contract

| Layer | Owns | Must NOT own | Enforced by |
| --- | --- | --- | --- |
| `Nop.Core` | entities, cache/event abstractions, infrastructure primitives | business rules, data access, HTTP | project references (Core references nothing) |
| `Nop.Data` | query execution, schema, migrations, entity events | business decisions | references Core only |
| `Nop.Services` | ALL business logic, orchestration | HTTP types, view models | references Data+Core; no ASP.NET reference |
| `Nop.Web.Framework` | MVC plumbing: filters, DI wiring, work context | domain rules | — |
| `Nop.Web` | controllers, factories, view models, views | direct DB access, business rules | convention (not enforced!) |

The strongest wall is compiler-backed: `Nop.Services` cannot mention `IActionResult` because the assembly doesn't reference MVC. The weakest wall is Web→Services: nothing but convention stops a controller from composing raw queries or embedding business rules.

## Good boundaries, with receipts

- **Controller/service split done right:** [ShoppingCartController.AddProductToCart_Details](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791-L874) does HTTP things (form parsing, JSON shapes) and delegates every decision to [AddToCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549). You could bolt a REST API onto the same service untouched — and the [WebApi plugin](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend) does exactly that. That's the test of a real boundary: *a second consumer arrives and nothing has to move*.
- **Entity/view-model wall:** factories ([ProductModelFactory#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502)) are the only place entities become models; views never see entities. This protects the DB shape from becoming a UI contract — the #1 coupling disease in CRUD apps.
- **Provider boundary:** three databases + SQLite-for-tests behind [INopDataProvider](../../src/Libraries/Nop.Data/INopDataProvider.cs) — an abstraction earning its keep (contrast with interfaces that merely mirror one class).

## Boundary leaks worth studying

1. **`IQueryable` through the repository.** [EntityRepository.Table#L525](../../src/Libraries/Nop.Data/EntityRepository.cs#L525) hands services raw query composition. Pragmatic (services can build efficient queries) but the repository's guarantees — caching, even entity events for predicate deletes ([#L497](../../src/Libraries/Nop.Data/EntityRepository.cs#L497)) — no longer hold on that path. A boundary with a documented hole is livable; the skill is *knowing the hole is there*.
2. **Events published from the data layer.** [EntityRepository](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L355) publishes app-level events — Data reaching "up" conceptually. The system buys automatic cache invalidation; the cost is that data-layer writes now run arbitrary consumer code synchronously ([EventPublisher](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L20-L52)) — a write can be slowed by any plugin's consumer.
3. **Ambient work context.** [WebWorkContext](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) gives *every* service access to "current customer/store" — convenient, but it means service behavior can depend on HTTP-ish ambient state, which is why `BaseNopTest` must fake a work context to test services at all.
4. **Business rules in controllers (occasional).** The product-match check at [ShoppingCartController.cs#L842-L849](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L842-L849) is a domain rule living in the controller — a second consumer (API plugin) must remember to duplicate it. Find-more-of-these is review kata material.

## Transferable test

For any repo, ask per layer: (1) what would a second consumer of this layer need? (2) what compiler/tooling *enforces* the rule vs mere convention? (3) where are the documented holes? Interview phrasing: "boundaries are where I check invariants and where I expect bugs when two teams' assumptions meet."

Drill: find one more business rule living in a controller in `Nop.Web/Controllers` and write the one-sentence case for moving it (or leaving it). Basic: found one. Solid: correct move/leave call with the second-consumer test. Strong: also names the blast radius of moving it (which plugins/overriders might notice).
