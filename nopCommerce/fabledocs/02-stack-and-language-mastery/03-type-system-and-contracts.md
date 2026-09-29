# Type System and Contracts

How this codebase uses (and declines to use) the type system, and what each choice commits it to. "Contract" here means: what a caller may rely on — signatures, invariants, and behavior that can't change without breaking someone.

## Interfaces as capabilities

The repo's most distinctive type-level idea: entities declare cross-cutting abilities by interface — [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) implements seven — and generic services accept `where TEntity : BaseEntity, ICapability` ([StoreMappingService.AuthorizeAsync#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221)). The compiler proves you can't call store-authorization on an entity without store mappings. The gap: nothing forces a *new* entity to opt in — a missing interface silently means "no isolation." Types verify what you *say*, not what you *forgot to say*.

Interview angle → [Q5](../08-interview-prep/01-csharp-dotnet-deep-dive.md). Say: "capability interfaces move a runtime question ('does this entity support X?') to compile time — and the residual risk is the opt-in gap, which we close in review."

## One interface per service, ~200 times

Every service has exactly one interface and one implementation, bound in [NopStartup.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215). With one implementation, what does `IShoppingCartService` buy? Three real things: the DI swap point for plugin overrides, the seam for subclass-and-rebind customization, and a compile-time firewall keeping Web referencing abstractions. What it doesn't buy: abstraction — these interfaces mirror implementations 1:1, so they change together. Know both halves; "interfaces everywhere" as unconditional dogma is a junior tell, and *this repo has a reason* — the plugin ecosystem.

## Contracts hiding in weak types

The type system is *not* protecting several important contracts here — recognize each, because "find the untyped contract" is a superb review skill:

- **`attributesXml : string`** — a whole grammar of product attribute selections rides in a string ([AddToCartAsync#L1550](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1550)); only `IProductAttributeParser` understands it. Nothing stops you passing garbage.
- **Warnings as `IList<string>`** — the *set of possible failures* of add-to-cart is unenumerable from the signature ([#L1549](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549)); compare an enum-based result like [CustomerLoginResults](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138) which *is* enumerable — same problem, two solutions, one file apart. Great contrast to cite.
- **Permission names as `string` constants** — `[CheckPermission("...")]` ([CheckPermissionAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L25)) is greppable but not compiler-checked against the permission records seeded in the DB.
- **Anonymous JSON responses** — `{success, message, redirect}` shapes ([ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)) whose only spec is the JavaScript consumer.

## Nullability

The codebase predates nullable reference types; null carries two meanings — *not found* (repository returns null → [404 at ProductController.cs#L131-L133](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L131-L133)) and *not applicable* (`int? customwishlistid`). Guards are explicit: `ArgumentNullException.ThrowIfNull` at service entry ([#L1555](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555)). Interview angle → [Q16](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## Drill

Pick `AddToCartAsync`. Design (on paper) the *maximally typed* version: typed attribute selections, typed error union, typed quantities. Then write one sentence on why the maintainers would reject it. (Hint: every plugin on earth calls the current signature.) Basic: produces the design. Solid: names the compatibility cost. Strong: proposes an incremental path — new typed overload delegating to the old one — and where the deprecation line sits.
