# Domain Glossary

The nouns of this system, with where they live. E-commerce vocabulary doubles as interview vocabulary — "how would you model X" questions assume you know these distinctions. The first table is the quick reference; the sections after it show how the most confusable terms actually behave in code.

| Term | Meaning here | Code home |
| --- | --- | --- |
| **Store** | One storefront (domain, name, theme). One deployment serves many stores — the multi-store axis. | `Nop.Core/Domain/Stores`; current store via `IStoreContext` |
| **Store mapping** | Which stores may show an entity (`IStoreMappingSupported`). Unmapped = visible everywhere. | [StoreMappingService.cs#L204](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204) |
| **Customer** | Every actor — including guests and search-engine bots — is a `Customer` row with roles. There is no separate "User". | `Nop.Core/Domain/Customers`; current via [WebWorkContext#L196](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) |
| **Customer role** | Group carrying permissions (Administrators, Registered, Guests, Vendors… — names in [NopCustomerDefaults.cs#L13-L28](../../src/Libraries/Nop.Core/Domain/Customers/NopCustomerDefaults.cs#L13-L28)). Authorization = role → permission-record mapping. | [PermissionService.cs#L258](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258) |
| **Vendor** | Third-party seller inside a store (multi-vendor axis — distinct from multi-store!). Vendors get row-scoped admin access. | vendor scoping in [Admin ProductController](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969) |
| **ACL** | Per-entity allow-list of customer roles (`IAclSupported`). A third visibility axis besides published-state and store mapping. | check at [ProductController.cs#L139](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139) |
| **Product type** | Simple vs Grouped (parent of associated products) — grouped products aren't buyable directly ([cart rejects them](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L803)). | `Nop.Core/Domain/Catalog` |
| **Product attribute** | Configurable option (size, color). Selections travel as an **XML string** (`attributesXml`) through cart and order. Near-synonym trap: *specification attributes* are display-only filters, a different entity. | parsing in [ShoppingCartController.cs#L855-L864](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L855-L864) |
| **Shopping cart item** | One cart/wishlist line. Cart and wishlist are the *same table*, split by `ShoppingCartType` ([enum: ShoppingCart = 1, Wishlist = 2](../../src/Libraries/Nop.Core/Domain/Orders/ShoppingCartType.cs#L6-L17)). | [ShoppingCartService.cs#L753](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L753) |
| **Order / OrderItem** | Immutable snapshot of a purchase — prices copied, not referenced (`UnitPriceInclTax`, `PriceExclTax`, `OriginalProductCost`, `AttributesXml` all live on [OrderItem.cs#L31-L71](../../src/Libraries/Nop.Core/Domain/Orders/OrderItem.cs#L31-L71)). | created in [SaveOrderDetailsAsync#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723) |
| **Queued email** | Outbox row awaiting the send task. | [QueuedMessagesSendTask.cs](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) |
| **Schedule task** | DB-registered recurring job run in-process. | [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| **Message template** | Tokenized email template (order placed, password recovery…), localized per store. | `Nop.Services/Messages` |
| **Generic attribute** | Schemaless key/value bolted onto any entity — the escape hatch when a column doesn't exist (e.g. selected MFA provider at [CustomerRegistrationService.cs#L176](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L176)). | [GenericAttributeService.cs](../../src/Libraries/Nop.Services/Common/GenericAttributeService.cs) |
| **Setting** | Store-scoped, runtime-editable typed config ([pattern 10](../03-architecture-and-patterns/05-pattern-catalog.md)). | e.g. `_customerSettings` in login flow; loading in [SettingService.cs#L316](../../src/Libraries/Nop.Services/Configuration/SettingService.cs#L316) |
| **Permission record** | Named capability (`ACCESS_ADMIN_PANEL`, `Catalog.ProductsView`) mapped to roles. Constants in [StandardPermission.cs](../../src/Libraries/Nop.Services/Security/StandardPermission.cs); default role grants in [DefaultPermissionConfigManager.cs](../../src/Libraries/Nop.Services/Security/DefaultPermissionConfigManager.cs). | usage [Admin ProductController#L943](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943) |
| **Slug / UrlRecord** | SEO-friendly URL per entity (`ISlugSupported`); one table maps slugs → entity type + id, and [SlugRouteTransformer.cs#L109-L110](../../src/Presentation/Nop.Web.Framework/Mvc/Routing/SlugRouteTransformer.cs#L109-L110) turns that into a controller route. | [UrlRecordService.cs](../../src/Libraries/Nop.Services/Seo/UrlRecordService.cs) |
| **Work context** | Ambient "current request state": customer, language, currency, vendor. The service everything injects. | [WebWorkContext.cs](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) |
| **Plugin** | Runtime-loaded extension project; can add payment methods, shipping, widgets, or override services. | [src/Plugins](../../src/Plugins) (32 projects) |
| **Widget** | Plugin-rendered fragment injected into named zones of storefront/admin views. | e.g. [Widgets.GoogleAnalytics](../../src/Plugins/Nop.Plugin.Widgets.GoogleAnalytics) |
| **Topic** | CMS page (about-us, T&C). | [Nop.Services/Topics](../../src/Libraries/Nop.Services/Topics) |
| **Price list** | Newer (UpgradeTo500) entity group: named, dated, prioritised price adjustments with customer / customer-role targeting. | [PriceList.cs](../../src/Libraries/Nop.Core/Domain/PriceLists/PriceList.cs); tables created by [PriceListMigration.cs#L13-L19](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L13-L19) |
| **Soft delete** | `Deleted` flag instead of row removal (`ISoftDeletedEntity`) for entities that history must keep resolving (products, customers…). | [EntityRepository.cs#L433-L452](../../src/Libraries/Nop.Data/EntityRepository.cs#L433-L452) |
| **System account** | Built-in `Customer` rows that are not people: the search-engine user and the background-task user. | [CustomerService.cs#L478](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L478), [#L516](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L516); test in [CustomerExtensions.cs#L13-L20](../../src/Libraries/Nop.Core/Domain/Customers/CustomerExtensions.cs#L13-L20) |

Confusables to keep straight in interviews: **multi-store vs multi-vendor vs multi-language** are three orthogonal axes; **product attribute vs specification attribute**; **permission (capability) vs ACL (per-entity role filter)**; **customer (includes guests/bots) vs registered customer (a role)**.

## "Customer" is wider than you think — a trace

The single most useful glossary fact in this repo: *a request always has a `Customer`*. The resolution chain is in `WebWorkContext.SetCurrentCustomerAsync` ([WebWorkContext.cs#L212-L284](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L212-L284)). In order:

1. If there is no `HttpContext`, or the path is the schedule-task endpoint, use the **background-task system account** ([L217-L223](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L217-L223)).
2. Else, if the user agent looks like a crawler, use the **search-engine system account** ([L225-L230](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L225-L230)).
3. Else, try the **authenticated** customer from the auth cookie ([L232-L236](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L232-L236)).
4. If authenticated, apply **impersonation** (an admin acting as a customer) via a generic attribute ([L238-L255](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L238-L255)).
5. Else, look up the **guest** by the customer cookie GUID — and only accept it if that customer is *not* registered ([L257-L268](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L257-L268)).
6. Else, **insert a new guest row** ([L270-L274](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L270-L274)) and set the cookie ([L277-L284](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L277-L284)).

Vocabulary consequences:

- "Anonymous cart" is a misnomer here — the cart belongs to a real guest `Customer` row (`ShoppingCartItem.CustomerId`), which is why guest carts survive across requests and why guest-cleanup exists ([CustomerService.cs#L751](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L751)).
- "Registered" is a **role**, not a type. `ValidateCustomerAsync` refuses login to a customer without it ([CustomerRegistrationService.cs#L150-L152](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L150-L152)).
- "Bot" is a **system account**, and business code checks for it by name: the cart refuses search engines with `IsSearchEngineAccount()` ([ShoppingCartService.cs#L1572-L1576](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1572-L1576)).

## The visibility axes — one product, five gates

"Can this customer see this product?" is answered by *several* glossary terms at once. The storefront product page evaluates them in [ProductController.cs#L132-L148](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L132-L148):

| Gate | Term | Implementation | Default when not configured |
| --- | --- | --- | --- |
| Exists & not deleted | soft delete | `product == null \|\| product.Deleted` → 404 (L132) | n/a |
| Published | publishing state | `!product.Published && !AllowViewUnpublishedProductPage` (L137) | must be published |
| ACL | ACL | [AclService.cs#L239-L284](../../src/Libraries/Nop.Services/Security/AclService.cs#L239-L284): `SubjectToAcl == false` → allowed; `IgnoreAcl` setting → allowed | **visible to all roles** |
| Store mapping | store mapping | [StoreMappingService.cs#L221-L245](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L221-L245): `storeId == 0`, `IgnoreStoreLimitations`, or `LimitedToStores == false` → allowed | **visible in all stores** |
| Availability window | product availability dates | `ProductIsAvailable` (L143) | available |

Then comes an override: if the viewer holds `ACCESS_ADMIN_PANEL` *and* `Catalog.ProductsView`, any "not available" verdict is ignored so admins can preview (L146-L148).

The **cart** re-checks the same axes server-side in `GetStandardWarningsAsync` ([ShoppingCartService.cs#L330-L364](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L330-L364)) — note that ACL and store-mapping failures there reuse the *unpublished* message (L356, L360), so a shopper cannot tell "hidden from you" from "unpublished". That's a deliberate information-hiding choice worth naming in an interview.

## Two "configuration-ish" terms that are really data

**Setting.** In most apps "settings" means a config file. Here a *setting* is a database row, and a *settings class* (anything implementing `ISettings`, e.g. `CatalogSettings`, `CustomerSettings`, `OrderSettings`) is a typed projection of those rows for one store. The DI registration makes this explicit: every `ISettings` type is discovered by reflection and registered as a **scoped factory** that resolves the current store and calls `LoadSettingAsync(type, storeId)` ([NopStartup.cs#L273-L283](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L273-L283)). Implications a junior usually misses:

- A setting value can differ **per store** in the same process; "what is `MaximumShoppingCartItems`?" has no single answer.
- Settings are injected like services (`_shoppingCartSettings` in `ShoppingCartService`), so a test or a background task that runs with a different current store sees different behaviour.
- The factory blocks on async work with `.Result` (L279, L282) — a second sync-over-async site besides the famous checkout mutex (see [02-stack-and-language-mastery/01-language-runtime-model.md](../02-stack-and-language-mastery/01-language-runtime-model.md)).
- Note the distinction from **app settings** (`AppSettings` / `CommonConfig` / `CacheConfig`): those come from `appsettings.json` and environment variables ([Program.cs#L14-L23](../../src/Presentation/Nop.Web/Program.cs#L14-L23)) and are process-wide. `PlaceOrderWithLock` is a *setting* (DB, per store, installed as `false` — [InstallRequiredData.cs#L1888](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L1888)); `DefaultCacheTime` is an *app setting* ([CacheConfig.cs#L11](../../src/Libraries/Nop.Core/Configuration/CacheConfig.cs#L11)). Mixing the two up is the most common "I changed it and nothing happened" cause.

**Generic attribute.** A `(KeyGroup, EntityId, Key, Value, StoreId)` row that can hang any string off any entity ([GenericAttribute.cs](../../src/Libraries/Nop.Core/Domain/Common/GenericAttribute.cs)). The code uses it for things that "should" be columns: the selected MFA provider ([CustomerRegistrationService.cs#L176](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L176)), the impersonated customer id ([WebWorkContext.cs#L241-L242](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L241-L242)). The trade: no migration needed and plugins can store per-entity data without schema changes; in exchange there is no type, no FK, and cleanup must be done by key — e.g. guest cleanup deletes generic attributes by `KeyGroup == nameof(Customer)` in a separate statement right after deleting the customers ([CustomerService.cs#L751-L754](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L751-L754)). When you see `GetAttributeAsync<T>(entity, "SomeKey")`, read it as "a column that lives in another table".

## Exercise ladder

1. **Goal:** distinguish the three "multi-" axes in code, not prose.
   **Check:** find the interface or service for each: `IStoreMappingSupported` (stores), `Product.VendorId` + `GetCurrentVendorAsync` (vendors), `ILocalizedEntity` (languages). `grep -n "VendorId" src/Libraries/Nop.Core/Domain/Catalog/Product.cs` must return a hit; `Product.cs#L13` must list the other two interfaces.
2. **Goal:** prove cart and wishlist share a table.
   **Check:** open [ShoppingCartType.cs](../../src/Libraries/Nop.Core/Domain/Orders/ShoppingCartType.cs) and the item insert at [ShoppingCartService.cs#L1657-L1673](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1657-L1673); confirm `ShoppingCartType` is a column on the same `ShoppingCartItem` row and `CustomWishlistId` is set only for wishlists (L1662).
3. **Goal:** explain why `OrderItem` is a snapshot.
   **Check:** list the price fields on [OrderItem.cs#L31-L71](../../src/Libraries/Nop.Core/Domain/Orders/OrderItem.cs#L31-L71) and find one that has no equivalent on `Product` (`AttributeDescription` — a rendered, human-readable copy of the selections at purchase time).
4. **Goal (mid):** enumerate which roles get admin-area access by default.
   **Check:** `grep -n "ACCESS_ADMIN_PANEL\|PRODUCTS_VIEW" src/Libraries/Nop.Services/Security/DefaultPermissionConfigManager.cs`. Expected: lines 16 and 69 grant both to Administrators **and Vendors**. Now re-read the admin-preview override above — and see the finding below.
5. **Goal (mid):** name the default direction of every visibility axis and say which are fail-open.
   **Check:** your answer must cite `!entity.SubjectToAcl → true` ([AclService.cs#L244-L245](../../src/Libraries/Nop.Services/Security/AclService.cs#L244-L245)) and `!entity.LimitedToStores → true` ([StoreMappingService.cs#L233-L234](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L233-L234)).

## Senior finding: "admin preview" is not vendor-scoped

Verified by reading, not exploited. The preview override at [ProductController.cs#L146-L148](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L146-L148) requires `ACCESS_ADMIN_PANEL` and `Catalog.ProductsView`. Both are granted to the **Vendors** role by default ([DefaultPermissionConfigManager.cs#L16](../../src/Libraries/Nop.Services/Security/DefaultPermissionConfigManager.cs#L16), [#L69](../../src/Libraries/Nop.Services/Security/DefaultPermissionConfigManager.cs#L69)). Unlike the "edit" link a few lines later, which *does* check `currentVendor.Id == product.VendorId` ([L187-L190](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L187-L190)), the preview bypass has no vendor check. Result: a vendor account can open the storefront page of *another vendor's* unpublished, ACL-restricted, or other-store product if they know its slug or id. Severity is low-to-moderate (information disclosure between tenants of the marketplace, e.g. unreleased products), and the fix is the same vendor predicate the edit link already uses. This is a textbook glossary bug: the code treats "has admin access" as "is the store owner", but the glossary says vendors also have admin access.

## Debugging scenario: "a product shows up in the wrong store"

*Report:* store B's customers can see a product the merchandiser assigned only to store A.

1. **Check the flag, not the mapping rows.** Store mapping is two-part: the `LimitedToStores` flag on the product *and* `StoreMapping` rows. If the flag is false, rows are ignored ([StoreMappingService.cs#L233-L234](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L233-L234)).
2. **Check the global override.** `CatalogSettings.IgnoreStoreLimitations` short-circuits to "allowed" ([L230-L231](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L230-L231)). Someone may have enabled it for performance.
3. **Check the viewer.** Is the reporter an admin or a vendor? Then they are seeing the preview override, not a store-mapping bug (see finding).
4. **Check the cache.** The admin edit flow saves the product (publishing an update event that evicts it) and *then* saves store mappings ([Admin ProductController.cs#L1253](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1253), [#L1290](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1290)); `SaveStoreMappingsAsync` flips `LimitedToStores` through `INopDataProvider` directly ([StoreMappingService.cs#L266-L270](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L266-L270)), which publishes no product event. If a storefront request re-cached the product between those two writes, the cached copy can carry the old flag until the 60-minute default cache time ([CacheConfig.cs#L11](../../src/Libraries/Nop.Core/Configuration/CacheConfig.cs#L11)) expires. Clearing the cache from admin resolving the symptom would support this hypothesis. (Inference from code order; not reproduced.)

## Verification

2026-10-06: every anchor on this page was opened and checked; role names, default permission grants (lines 16 and 69), the visibility gate order, and the `WebWorkContext` resolution chain were read directly from source. The cache-window scenario in the debugging section is an inference from call order and is labelled as such.
