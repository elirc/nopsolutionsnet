# Domain Glossary

The nouns of this system, with where they live. E-commerce vocabulary doubles as interview vocabulary — "how would you model X" questions assume you know these distinctions.

| Term | Meaning here | Code home |
| --- | --- | --- |
| **Store** | One storefront (domain, name, theme). One deployment serves many stores — the multi-store axis. | `Nop.Core/Domain/Stores`; current store via `IStoreContext` |
| **Store mapping** | Which stores may show an entity (`IStoreMappingSupported`). Unmapped = visible everywhere. | [StoreMappingService.cs#L204](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204) |
| **Customer** | Every actor — including guests and search-engine bots — is a `Customer` row with roles. There is no separate "User". | `Nop.Core/Domain/Customers`; current via [WebWorkContext#L196](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) |
| **Customer role** | Group carrying permissions (Administrators, Registered, Guests…). Authorization = role → permission-record mapping. | [PermissionService.cs#L258](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258) |
| **Vendor** | Third-party seller inside a store (multi-vendor axis — distinct from multi-store!). Vendors get row-scoped admin access. | vendor scoping in [Admin ProductController](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969) |
| **ACL** | Per-entity allow-list of customer roles (`IAclSupported`). A third visibility axis besides published-state and store mapping. | check at [ProductController.cs#L139](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139) |
| **Product type** | Simple vs Grouped (parent of associated products) — grouped products aren't buyable directly ([cart rejects them](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L803)). | `Nop.Core/Domain/Catalog` |
| **Product attribute** | Configurable option (size, color). Selections travel as an **XML string** (`attributesXml`) through cart and order. Near-synonym trap: *specification attributes* are display-only filters, a different entity. | parsing in [ShoppingCartController.cs#L855-L864](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L855-L864) |
| **Shopping cart item** | One cart/wishlist line. Cart and wishlist are the *same table*, split by `ShoppingCartType`. | [ShoppingCartService.cs#L753](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L753) |
| **Order / OrderItem** | Immutable snapshot of a purchase — prices copied, not referenced. | created in [SaveOrderDetailsAsync#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723) |
| **Queued email** | Outbox row awaiting the send task. | [QueuedMessagesSendTask.cs](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) |
| **Schedule task** | DB-registered recurring job run in-process. | [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) |
| **Message template** | Tokenized email template (order placed, password recovery…), localized per store. | `Nop.Services/Messages` |
| **Generic attribute** | Schemaless key/value bolted onto any entity — the escape hatch when a column doesn't exist (e.g. selected MFA provider at [CustomerRegistrationService.cs#L176](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L176)). | `Nop.Services/Common/GenericAttributeService` |
| **Setting** | Store-scoped, runtime-editable typed config ([pattern 10](../03-architecture-and-patterns/05-pattern-catalog.md)). | e.g. `_customerSettings` in login flow |
| **Permission record** | Named capability (`ACCESS_ADMIN_PANEL`, `Catalog.Products.View`) mapped to roles. Constants in `StandardPermission`. | usage [Admin ProductController#L943](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943) |
| **Slug / UrlRecord** | SEO-friendly URL per entity (`ISlugSupported`); one table maps slugs → entity type + id. | `Nop.Services/Seo/UrlRecordService` |
| **Work context** | Ambient "current request state": customer, language, currency, vendor. The service everything injects. | [WebWorkContext.cs](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L196) |
| **Plugin** | Runtime-loaded extension project; can add payment methods, shipping, widgets, or override services. | [src/Plugins](../../src/Plugins) |
| **Widget** | Plugin-rendered fragment injected into named zones of storefront/admin views. | e.g. [Widgets.GoogleAnalytics](../../src/Plugins/Nop.Plugin.Widgets.GoogleAnalytics) |
| **Topic** | CMS page (about-us, T&C). | `Nop.Services/Topics` |

Confusables to keep straight in interviews: **multi-store vs multi-vendor vs multi-language** are three orthogonal axes; **product attribute vs specification attribute**; **permission (capability) vs ACL (per-entity role filter)**; **customer (includes guests/bots) vs registered customer (a role)**.
