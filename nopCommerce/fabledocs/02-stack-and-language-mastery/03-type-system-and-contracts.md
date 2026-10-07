# Type System and Contracts

How this codebase uses (and declines to use) the type system, and what each choice commits it to. "Contract" here means: what a caller may rely on — signatures, invariants, and behavior that can't change without breaking someone.

## Interfaces as capabilities

The repo's most distinctive type-level idea: entities declare cross-cutting abilities by interface — [Product.cs#L13](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L13) implements seven (`ILocalizedEntity`, `ISlugSupported`, `IAclSupported`, `IStoreMappingSupported`, `IDiscountSupported<>`, `ISoftDeletedEntity`, `IMetaTagsSupported`) — and generic services accept `where TEntity : BaseEntity, ICapability` ([StoreMappingService.AuthorizeAsync#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221), [AclService.AuthorizeAsync#L224-L239](../../src/Libraries/Nop.Services/Security/AclService.cs#L224-L239)). The compiler proves you can't call store-authorization on an entity without store mappings. The gap: nothing forces a *new* entity to opt in — a missing interface silently means "no isolation." Types verify what you *say*, not what you *forgot to say*.

Look at how small the capabilities are: [IStoreMappingSupported](../../src/Libraries/Nop.Core/Domain/Stores/IStoreMappingSupported.cs) is one property (`bool LimitedToStores`), [IAclSupported](../../src/Libraries/Nop.Core/Domain/Security/IAclSupported.cs) is one property (`bool SubjectToAcl`), [ISoftDeletedEntity](../../src/Libraries/Nop.Core/Domain/Common/ISoftDeletedEntity.cs) is one property (`bool Deleted`). Each is a *marker plus a fast-path flag*: the flag lets the service skip the mapping-table lookup in the common case (`if (!entity.LimitedToStores) return true;`).

Capabilities are also checked **at runtime by reflection**, which is a weaker contract than the generic constraint: the repository's soft-delete filter asks `typeof(TEntity).GetInterface(nameof(ISoftDeletedEntity)) == null` ([EntityRepository.cs#L101-L104](../../src/Libraries/Nop.Data/EntityRepository.cs#L101-L104)) and `DeleteAsync` pattern-matches `case ISoftDeletedEntity` ([L437-L442](../../src/Libraries/Nop.Data/EntityRepository.cs#L437-L442)). Same interface, two enforcement styles — compile time for authorization, run time for deletion.

Interview angle → [Q5](../08-interview-prep/01-csharp-dotnet-deep-dive.md). Say: "capability interfaces move a runtime question ('does this entity support X?') to compile time — and the residual risk is the opt-in gap, which we close in review."

## One interface per service

`Nop.Services` declares **148** interfaces (counted: `grep -rhoE "public (partial )?interface I\w+" --include=*.cs src/Libraries/Nop.Services | wc -l`), and the core startup binds them in [NopStartup.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215). Most have exactly one implementation in the repo. With one implementation, what does `IShoppingCartService` buy? Three real things: the DI swap point for plugin overrides, the seam for subclass-and-rebind customization, and a test seam. What it doesn't buy: abstraction — these interfaces mirror implementations 1:1, so they change together. Know both halves; "interfaces everywhere" as unconditional dogma is a junior tell, and *this repo has a reason* — the plugin ecosystem.

Contrast the interfaces that **do** earn their abstraction because they have several real implementations:

| Interface | Implementations in this repo | Selected by |
| --- | --- | --- |
| `IStaticCacheManager` | `MemoryCacheManager`, Redis / SQL Server / memory-distributed managers, `SynchronizedMemoryCacheManager` | `DistributedCacheConfig` at startup ([NopStartup.cs#L98-L130](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L98-L130)) |
| `INopDataProvider` | MS SQL, MySQL, PostgreSQL providers ([DataProviders/](../../src/Libraries/Nop.Data/DataProviders)) + the test SQLite provider | data settings |
| `ITaxProvider` | e.g. [AvalaraTaxProvider](../../src/Plugins/Nop.Plugin.Tax.Avalara/AvalaraTaxProvider.cs#L21), [FixedOrByCountryStateZipTaxProvider](../../src/Plugins/Nop.Plugin.Tax.FixedOrByCountryStateZip/FixedOrByCountryStateZipTaxProvider.cs#L22) | admin configuration, per store |

The second kind is a *strategy*; the first kind is a *seam*. Being able to say which one you are looking at is a mid-level skill.

## Result types: three designs for "this might fail"

| Design | Example | What the type tells a caller |
| --- | --- | --- |
| Closed enum | [CustomerLoginResults](../../src/Libraries/Nop.Core/Domain/Customers/CustomerLoginResults.cs) — 8 values, `Successful = 1` … `MultiFactorAuthenticationRequired = 8` | every outcome is enumerable; a `switch` can be exhaustive ([CustomerController.cs#L487-L528](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L487-L528)) |
| Result object with error strings | [BaseNopResult](../../src/Libraries/Nop.Services/BaseNopResult.cs) — `Success => !Errors.Any()`, `IList<string> Errors`; subclassed by `PlaceOrderResult` | *that* it failed and a human message, not *why* in a machine-checkable form |
| Bare list of warnings | `Task<IList<string>> AddToCartAsync(...)` ([ShoppingCartService.cs#L1549](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549)) | empty list = success; nothing else |

The enum lets the controller choose different messages per outcome (and that choice has security consequences — see [03-architecture-and-patterns/03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)). The string designs make localization the producer's job and make it impossible for a caller to react to a *specific* failure without string matching. Note what the enum design costs, too: `ValidateCustomerAsync` returns only the code, so the controller re-queries the customer it already loaded ([CustomerController.cs#L491-L493](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L491-L493)).

## Enums persisted as integers

Entities store enums as `int` columns with a typed wrapper property: `ProductTypeId` ([Product.cs#L18](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L18)) and `ProductType { get => (ProductType)ProductTypeId; set => … }` ([Product.cs#L538-L542](../../src/Libraries/Nop.Core/Domain/Catalog/Product.cs#L538-L542)). Why the wrapper isn't a column: schema auto-mapping only includes properties whose type is in a fixed table — `int`, `long`, `string`, `bool`, `decimal`, `DateTime`, `byte[]`, `Guid` ([FluentMigratorExtensions.cs#L28-L38](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L28-L38), filter at [L273-L279](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L273-L279)) — and an enum type is not in it.

Contract consequences:

- **Enum numeric values are a persisted contract.** `ProductType` is `SimpleProduct = 5`, `GroupedProduct = 10` ([ProductType.cs](../../src/Libraries/Nop.Core/Domain/Catalog/ProductType.cs)). Renumbering silently re-types every existing row.
- **The cast never validates.** `(ProductType)0` is a legal C# value that matches no member. Note that `0` — the `int` default — is deliberately *not* a valid product type here, which turns "forgot to set it" into "unknown type" instead of "simple product".

## Contracts hiding in weak types

The type system is *not* protecting several important contracts here — recognize each, because "find the untyped contract" is a superb review skill:

- **`attributesXml : string`** — a whole grammar of product attribute selections rides in a string ([AddToCartAsync#L1550](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1550)); only [IProductAttributeParser](../../src/Libraries/Nop.Services/Catalog/IProductAttributeParser.cs#L9) understands it. Nothing stops you passing garbage.
- **Warnings as `IList<string>`** — the *set of possible failures* of add-to-cart is unenumerable from the signature ([#L1549](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549)); compare the enum-based result above — same problem, two solutions. Some of those strings aren't even localized ([L1562](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1562), [L1568](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1568), [L1574](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1574), [L352](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L352)).
- **Permission names as `string` constants** — `[CheckPermission("...")]` ([CheckPermissionAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L25)) is greppable but not compiler-checked against the permission records seeded in the DB.
- **Entity type names as strings** — ACL and store-mapping rows key on `entity.GetType().Name` ([AclService.cs#L247](../../src/Libraries/Nop.Services/Security/AclService.cs#L247)); renaming an entity class is a data migration.
- **Anonymous JSON responses** — `{success, message, redirect}` shapes ([ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)) whose only spec is the JavaScript consumer.

## Nullability

The codebase does not enable nullable reference types: [Directory.Build.props](../../src/Directory.Build.props) sets only `TargetFramework`, `ImplicitUsings` and `NoWarn`, and `grep -rl "<Nullable>" --include=*.csproj src` returns nothing. Null carries two meanings — *not found* (repository returns null → [404 at ProductController.cs#L131-L133](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L131-L133)) and *not applicable* (`int? customwishlistid`). Guards are explicit: `ArgumentNullException.ThrowIfNull` at service entry ([#L1555](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555)). The cost shows up in overload families with inconsistent guards: `PermissionService.AuthorizeAsync(PermissionRecord, Customer)` returns `false` for a null customer ([PermissionService.cs#L277-L278](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L277-L278)), but the string overload it delegates to does not check `customer` before calling `GetCustomerRolesAsync(customer)` ([L305-L310](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L305-L310)). With NRT enabled, the compiler would have flagged that asymmetry. Interview angle → [Q16](../08-interview-prep/01-csharp-dotnet-deep-dive.md).

## Feature trace: the types along add-to-cart

| Hop | Type in hand | Where |
| --- | --- | --- |
| HTTP form | `IFormCollection` | [ShoppingCartController.cs#L791](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791) |
| parsed selections | `string` (XML) — parser receives `IFormCollection` and a `List<string>` of errors | [IProductAttributeParser.cs#L147](../../src/Libraries/Nop.Services/Catalog/IProductAttributeParser.cs#L147), call at [ShoppingCartController.cs#L861](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L861) |
| quantity / price | `int`, `decimal` | [ShoppingCartController.cs#L855-L858](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L855-L858) |
| service result | `IList<string>` warnings | [ShoppingCartService.cs#L1549-L1553](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1553) |
| persisted row | `ShoppingCartItem` with `AttributesXml` string, `ShoppingCartType` enum | [ShoppingCartService.cs#L1657-L1673](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1657-L1673) |
| HTTP response | anonymous object → JSON | [ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810) |

Two of six hops carry real types; the rest are strings or anonymous shapes. Note also that a *service* interface (`IProductAttributeParser`, in `Nop.Services`) takes an ASP.NET `IFormCollection` — see the finding.

## Exercise ladder

1. **Goal:** enumerate a closed result type.
   **Check:** open [CustomerLoginResults.cs](../../src/Libraries/Nop.Core/Domain/Customers/CustomerLoginResults.cs) and list all 8 members with values; then confirm the controller `switch` handles each (some via `default`).
2. **Goal:** prove enum wrappers aren't columns.
   **Check:** read [FluentMigratorExtensions.cs#L273-L279](../../src/Libraries/Nop.Data/Extensions/FluentMigratorExtensions.cs#L273-L279) and state the exact predicate that excludes `ProductType` but includes `ProductTypeId`.
3. **Goal:** find every capability interface an entity can opt into.
   **Check:** `grep -rn "public partial interface I.*Supported\|interface ISoftDeletedEntity" src/Libraries/Nop.Core/Domain`; for each, find one generic service method with that constraint.
4. **Goal (mid):** design a typed add-to-cart result without breaking callers.
   **Check:** your design must keep `Task<IList<string>> AddToCartAsync(...)` compiling for existing plugins (e.g. by adding a new method that returns a typed result and having the old one project it to strings) and must name where the deprecation line sits.
5. **Goal (mid):** locate HTTP types inside the service layer.
   **Check:** `grep -rln "IFormCollection\|IActionResult\|HttpContext" --include=*.cs src/Libraries/Nop.Services | wc -l` → 28 files on 2026-10-06. Pick two and argue whether each belongs in Services.

## Senior finding: HTTP result types in a service contract

`ICustomerRegistrationService.SignInCustomerAsync` returns `Task<IActionResult>` ([ICustomerRegistrationService.cs#L62](../../src/Libraries/Nop.Services/Customers/ICustomerRegistrationService.cs#L62), implementation [CustomerRegistrationService.cs#L456](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L456)), and `ExternalAuthenticationService` has several `IActionResult`-returning methods ([ExternalAuthenticationService.cs#L92](../../src/Libraries/Nop.Services/Authentication/External/ExternalAuthenticationService.cs#L92)). A service whose *return type* is an MVC result can only be consumed by an MVC controller; a REST/API consumer (any API plugin — the in-repo [WebApi.Frontend](../../src/Plugins/Nop.Plugin.Misc.WebApi.Frontend) is only a configuration shell) must either re-implement sign-in or unwrap a redirect result. It compiles because `Nop.Core` itself references ASP.NET Core MVC packages ([Nop.Core.csproj line 19–20](../../src/Libraries/Nop.Core/Nop.Core.csproj)), so every layer above it can see MVC types. This is a real contract leak, not a style nit: the type signature has decided the delivery mechanism. A non-breaking path is a new method returning a domain result (customer, return URL, cookie intent) with the old one adapting it.

## Debugging scenario: "imported products can't be added to the cart"

*Report:* products bulk-inserted with a SQL script show on the storefront but the add-to-cart button responds with "Only simple products could be added to the cart".

1. **Read the check that fired:** [ShoppingCartController.cs#L803-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L803-L810) compares `product.ProductType != ProductType.SimpleProduct`.
2. **Look at the stored value:** `ProductType` is just `(ProductType)ProductTypeId` (Product.cs L538-L542). The script likely left `ProductTypeId = 0` — or used `1` assuming a 0/1 enum. Neither is a member: valid values are 5 and 10.
3. **Explain why nothing failed earlier:** the cast never throws, there is no DB check constraint generated for enums, and the product page doesn't validate the type.
4. **Fix the data, then the process:** update rows to 5; for future imports, use the service layer or the admin import (which sets enums through typed properties) rather than raw SQL. Interview phrasing: "an enum persisted as int is a contract between the code and the data; raw SQL bypasses the only party that knows the contract."

## Verification

2026-10-06: every anchor opened and checked. Corrected the heading "One interface per service, ~200 times" to the counted 148 interfaces; confirmed `<Nullable>` is absent from all csproj files and `Directory.Build.props`; confirmed `Nop.Core.csproj` references `Microsoft.AspNetCore.Mvc.NewtonsoftJson` and `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` (lines 19–20). The debugging scenario is constructed from the code path; it was not reproduced.
