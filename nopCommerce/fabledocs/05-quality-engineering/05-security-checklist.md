# Security Checklist

Threat-by-threat map to this codebase, ending in the pre-merge checklist. Terms defined where used; every claim anchored or labeled *investigate*.

| Threat | What it is | Where this repo handles it | Notes / gaps to watch |
| --- | --- | --- | --- |
| **Broken authn** | weak login flow | lockout + counter ([CustomerRegistrationService#L154-L172](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154-L172)), CAPTCHA filter ([login action](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458)), MFA plugin hand-off (L175-L181), hashed passwords via `EncryptionService` | counter race under parallel attempts; distinct LockedOut message = mild enumeration |
| **Broken authz (capability)** | missing permission gates | `[AuthorizeAdmin]` area gate ([#L89](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89)) + per-action `[CheckPermission]` read/write split ([#L943 vs #L963](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965)) | missing attribute fails **open** inside the area — review item; automate ([ticket 15](../06-contribution-practice/01-good-first-tickets.md)) |
| **IDOR** | acting on another's object by id | *imperative* scoping: vendor filter in admin actions ([#L969](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969)); cart-item ownership check ([ShoppingCartController#L828-L849](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L828-L849)) | manual per action = the top audit target for any new endpoint |
| **Tenant isolation** | cross-store leakage | store mapping ([StoreMappingService#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221)) + ACL ([ProductController#L139](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139)) | unmapped = visible everywhere (fail-open default by design) |
| **XSS** | script injection via output | Razor encodes by default; `Html.Raw` is the escape hatch | audit every `Html.Raw` in Views/Themes; admin-entered HTML (topics, product descriptions) is *trusted-admin* content — know that trust decision exists |
| **CSRF** | forged cross-site POSTs | ASP.NET Core antiforgery on forms (auto-validated via global filter — *investigate* the exact registration in [Nop.Web.Framework/Infrastructure](../../src/Presentation/Nop.Web.Framework/Infrastructure)) | AJAX endpoints: verify token headers on the JSON POSTs |
| **SQL injection** | query manipulation | parameterized queries via Linq2Db expression trees throughout ([EntityRepository](../../src/Libraries/Nop.Data/EntityRepository.cs#L525)) | raw-SQL surfaces exist in data provider internals — any new raw SQL must be parameterized |
| **Bots/abuse** | scripted traffic | honeypot filter, bot detection (`IsSearchEngineAccount` gate at [AddToCartAsync#L1572](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1572)), CAPTCHA settings | rate limiting is **not** built in — front it with a proxy/WAF; say this in interviews |
| **Open redirect** | attacker-supplied return URLs | `returnUrl` flows through login ([CustomerController#L463](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L463)); note the code uses `LocalRedirect`/route redirects elsewhere ([ProductController#L173](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L173)) | verify `SignInCustomerAsync`'s handling of `returnUrl` validates localness — *investigate*; classic review catch |
| **Secrets** | credential leakage | connection string in `App_Data/appsettings.json` (installer-generated, gitignored); no secrets in repo | never log payment payloads ([review kata 8](../04-code-reading-gym/04-review-katas.md)); DataProtection keys in App_Data — file permissions matter |
| **SSRF** | server fetching attacker URLs | relevant surfaces: webhooks/integrations in plugins (PayPal, Zettle, etc.) | any feature fetching user-supplied URLs needs allow-listing; core has few such surfaces |
| **Uploads** | malicious files | media/download services restrict types (*investigate* `Nop.Services.Media` validation before relying on it) | serve uploads with correct content types; never from executable paths |
| **Dependency risk** | vulnerable packages | NuGet audit warnings **deliberately suppressed** (NU1901-04 in [Directory.Build.props](../../src/Directory.Build.props)) with a comment explaining churn | that suppression is a risk-acceptance decision — great interview discussion: who should get to make it, and what compensating control exists? |
| **Cookies/session** | theft/fixation | cookie auth via `CookieAuthenticationService` ([NopStartup registration](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215)); HTTPS enforcement filter ([HttpsRequirementAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/HttpsRequirementAttribute.cs)) | check Secure/HttpOnly/SameSite flags in the auth setup — *investigate* [AuthenticationStartup](../../src/Presentation/Nop.Web.Framework/Infrastructure/AuthenticationStartup.cs) |

## What a junior misses vs what a senior checks

Junior: "there's an attribute, it's secure." Senior: (1) object-level scope on every id-taking action; (2) fail-open defaults (unmapped entities, unattributed actions); (3) the *trust decisions* — admin HTML is trusted, plugins run fully trusted, NuGet warnings suppressed — each fine *if known and owned*, dangerous if invisible.

## Pre-merge security checklist (use on every PR you write here)

- [ ] Every new action taking an entity id: capability check (attribute) **and** scope check (query filtered by owner/store/vendor)
- [ ] New admin action: read vs write permission chosen correctly ([the L943/L963 split](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965))
- [ ] New entity: did you *decide* about `IAclSupported` / `IStoreMappingSupported` / `ISoftDeletedEntity` (deciding "no" consciously is fine)
- [ ] Any `Html.Raw` / manual URL building: encoded? local-redirect enforced?
- [ ] New form/AJAX POST: antiforgery story confirmed
- [ ] Logs: no credentials, tokens, card data; PII minimized
- [ ] External fetch of user-influenced URL: allow-list
- [ ] Money/quantity inputs: server recomputes; client chooses *what*, never *how much it costs*

Interview angle: walk this table top-to-bottom and you've answered "how do you think about web security" with a real system's evidence. The differentiator is naming the *fail-open defaults* and *trust decisions* — most candidates only list OWASP acronyms.
