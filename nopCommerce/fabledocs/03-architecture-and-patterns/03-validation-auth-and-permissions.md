# Validation, Authentication, and Permissions

The complete map of "who may do what, and what input is trusted where." Vocabulary: **authentication** = who you are; **authorization** = what you may do; **validation** = whether the input is acceptable; **isolation** = whose data you may see. Interviewers probe whether you keep these four separate — this repo implements each in a distinct mechanism, which makes it a perfect study text.

## The validation stack (outermost first)

| Layer | Mechanism | Example | Catches |
| --- | --- | --- | --- |
| 1. Anti-bot | filters: honeypot, CAPTCHA | [ValidateCaptcha on login](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458) | non-humans |
| 2. Model shape | FluentValidation on bound models | [LoginValidator#L9-L34](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9-L34) | missing/invalid fields |
| 3. Controller sanity | early returns | [product exists / is simple](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L793-L810) | nonsense requests |
| 4. Business rules | warnings lists in services | [AddToCartAsync#L1559-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582) | domain violations |
| 5. Invariant guards | `ThrowIfNull`, exceptions | [#L1555](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555) | programmer error |

The invariant across layers: **shape at the edge, semantics in the domain**. Layer 2 can't check stock; layer 4 shouldn't re-check email formats. Duplication between them is a smell; a *gap* between them is a vulnerability.

## Authentication

Cookie-based (`IAuthenticationService` → `CookieAuthenticationService`, registered in [NopStartup.cs#L210](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215)). Credential check: [ValidateCustomerAsync#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) — ordered gates (exists → deleted → active → registered → **locked out** → password), persisted failed-attempt counter with lockout, then MFA hand-off, then sign-in. External auth (Facebook) and MFA (TOTP) arrive as plugins. Guests are *authenticated-ish*: a guest `Customer` row exists per session — remember this when reasoning about "anonymous" carts; they're rows, not cookies.

Study points: lockout-before-password ordering (L154 vs L157); anti-enumeration message collapsing at the controller ([L521-L527](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L521-L527)); the deliberate exceptions to it (LockedOut has its own message — defensible UX tradeoff, know both sides).

## Authorization: three mechanisms, three granularities

1. **Permission records** (capability level): named permissions mapped to customer roles, checked via [PermissionService.AuthorizeAsync](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258-L331) (results cached per role). Declarative at controllers: [`[CheckPermission]`](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16) with **read vs write splits** ([PRODUCTS_VIEW vs PRODUCTS_CREATE_EDIT_DELETE](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965)); area-wide gate [`AuthorizeAdmin`](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89). Note permissions also gate *storefront* features ([cart enabled per customer, ShoppingCartService.cs#L1560](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560)).
2. **ACL** (entity level): per-record role allow-lists for `IAclSupported` entities — [checked on the product page](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139).
3. **Row scope / isolation** (instance level): vendor may touch only own products — *imperative* checks inside actions ([Admin ProductController vendor filter](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969)); store mapping isolates catalog per storefront ([StoreMappingService#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221)).

**What a junior misses vs what a senior checks:** the junior sees `[CheckPermission]` and declares an endpoint secure. The senior asks the IDOR question — *the attribute proves you may edit some product; what proves you may edit product 42?* — and looks for the imperative scope check, because that's the one that's manual, per-action, and forgettable. Then the senior asks the default-direction question: unmapped entity → visible everywhere; new action with no attribute → open (inside an area gate). Fail-open defaults are where audits earn their keep.

Drill: pick any admin controller you haven't read. For its most destructive action, write down: which of the three mechanisms fire, in what order, and what an attacker with a *valid low-privilege admin account* could reach. Basic: lists the attributes. Solid: traces all three mechanisms. Strong: finds one action where mechanism 3 is absent and judges (with evidence) whether that's a gap or genuinely scope-free.

Interview cross-links: [API/data cards Q-auth](../08-interview-prep/03-api-and-data-modeling-questions.md), security checklist → [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).
