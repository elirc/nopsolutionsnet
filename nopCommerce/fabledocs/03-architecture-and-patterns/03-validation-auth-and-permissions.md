# Validation, Authentication, and Permissions

The complete map of "who may do what, and what input is trusted where." Vocabulary: **authentication** = who you are; **authorization** = what you may do; **validation** = whether the input is acceptable; **isolation** = whose data you may see. Interviewers probe whether you keep these four separate — this repo implements each in a distinct mechanism, which makes it a perfect study text.

## The validation stack (outermost first)

| Layer | Mechanism | Example | Catches |
| --- | --- | --- | --- |
| 1. Anti-bot | filters: honeypot, CAPTCHA | [ValidateCaptcha on login](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458) | non-humans |
| 2. Model shape | FluentValidation, run by the `AutoValidation` action filter ([AutoValidationAttribute.cs#L96-L123](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L96-L123)) | [LoginValidator#L9-L34](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9-L34) | missing/invalid fields |
| 3. Controller sanity | early returns | [product exists / is simple](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L793-L810) | nonsense requests |
| 4. Business rules | warnings lists in services | [AddToCartAsync#L1559-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1559-L1582), [GetStandardWarningsAsync#L330-L364](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L330-L364) | domain violations |
| 5. Invariant guards | `ThrowIfNull`, exceptions | [#L1555](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1555) | programmer error |

The invariant across layers: **shape at the edge, semantics in the domain**. Layer 2 can't check stock; layer 4 shouldn't re-check email formats. Duplication between them is a smell; a *gap* between them is a vulnerability.

Notice that layer 2 is *conditional configuration*: `LoginValidator`'s rules depend on settings at construction time — email rules only when usernames are disabled, phone rules only when login-by-phone is enabled ([LoginValidator.cs#L13-L38](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L13-L38)). And it's tested: [LoginValidatorTests.cs](../../src/Tests/Nop.Tests/Nop.Web.Tests/Public/Validators/Customer/LoginValidatorTests.cs) covers the email/username combinations (four tests, from L22).

## Authentication

Cookie-based (`IAuthenticationService` → `CookieAuthenticationService`, registered at [NopStartup.cs#L210](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L210)). Credential check: [ValidateCustomerAsync#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) — ordered gates, then MFA hand-off, then sign-in. External auth (Facebook) and MFA (Google Authenticator) arrive as plugins. Guests are *authenticated-ish*: a guest `Customer` row exists per cookie ([WebWorkContext.cs#L257-L274](../../src/Presentation/Nop.Web.Framework/WebWorkContext.cs#L257-L274)) — remember this when reasoning about "anonymous" carts; they're rows, not cookies.

### Feature trace: `POST /login`, gate by gate

| # | Gate | Line | Result if it trips |
| --- | --- | --- | --- |
| 1 | CAPTCHA (if enabled for login) | [CustomerController.cs#L466-L467](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L466-L467) | model error |
| 2 | validator rules | `AutoValidation` filter | `ModelState.IsValid == false` ([L469](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L469)) |
| 3 | login-by-phone branch | [L474-L479](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L474-L479) | redirect to OTP flow |
| 4 | customer lookup by username or email | [CustomerRegistrationService.cs#L140-L145](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L140-L145) | `CustomerNotExist` |
| 5 | deleted / inactive / not registered | [L146-L152](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L146-L152) | `Deleted` / `NotActive` / `NotRegistered` |
| 6 | lockout window | [L154-L155](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154-L155) | `LockedOut` — **before** the password is checked |
| 7 | password | [L157-L173](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L157-L173) | increment counter; lock when it reaches `FailedPasswordAllowedAttempts` (if > 0); `WrongPassword` |
| 8 | MFA provider active? | [L175-L181](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L175-L181) | `MultiFactorAuthenticationRequired` → session + redirect ([CustomerController.cs#L498-L508](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L498-L508)) |
| 9 | success bookkeeping | [L186-L193](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L186-L193) | reset counter, clear lockout, set last login |
| 10 | sign-in | [CustomerController.cs#L489-L497](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L489-L497) → [SignInCustomerAsync#L456](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L456) | migrates the guest cart to the account, then signs in |

The controller maps results to messages at [L509-L527](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L509-L527): `CustomerNotExist`, `NotRegistered` and `WrongPassword` collapse into one generic "wrong credentials" message (anti-enumeration), while `Deleted`, `NotActive` and `LockedOut` each get their **own** message. Hold that thought for the finding below.

Two defaults that change the security posture of a fresh install, read from the installer:

- `FailedPasswordAllowedAttempts = 0` ([InstallRequiredData.cs#L1627](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L1627)) — and the lockout branch only runs when it is `> 0` (L161), so **lockout is off by default**.
- CAPTCHA `Enabled = false` and `ShowOnLoginPage = false` ([InstallRequiredData.cs#L2044](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L2044), [#L2053](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L2053)); the global rate limiter is also off by default (`PermitLimit = 0`, [CommonConfig.cs#L60](../../src/Libraries/Nop.Core/Configuration/CommonConfig.cs#L60)).

## Authorization: three mechanisms, three granularities

1. **Permission records** (capability level): named permissions mapped to customer roles, checked via [PermissionService.AuthorizeAsync](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258-L331). The chain: current customer → roles → for each role, a cached "is this permission granted to this role?" lookup keyed by `(permission, roleId)` ([PermissionService.cs#L305-L345](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L305-L345)). Declarative at controllers: [`[CheckPermission]`](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16) with **read vs write splits** ([PRODUCTS_VIEW vs PRODUCTS_CREATE_EDIT_DELETE](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965)); multiple names on one filter are OR-ed (the loop returns on the first granted name, [CheckPermissionAttribute.cs#L117-L121](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L117-L121)). Area-wide gate: [`AuthorizeAdmin`](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89), applied to every admin controller through [BaseAdminController.cs#L15](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/BaseAdminController.cs#L15). Note permissions also gate *storefront* features ([cart enabled per customer, ShoppingCartService.cs#L1560](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560)).
2. **ACL** (entity level): per-record role allow-lists for `IAclSupported` entities — [checked on the product page](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139), implemented as "not subject to ACL → allow; `IgnoreAcl` → allow; else intersect customer roles with the entity's allowed roles" ([AclService.cs#L239-L284](../../src/Libraries/Nop.Services/Security/AclService.cs#L239-L284)).
3. **Row scope / isolation** (instance level): vendor may touch only own products — *imperative* checks inside actions (e.g. [Admin ProductController.cs#L1196-L1199](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1196-L1199) in `Edit`; also [L969](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L969)); store mapping isolates catalog per storefront ([StoreMappingService#L204-L245](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L245)).

**What a junior misses vs what a senior checks:** the junior sees `[CheckPermission]` and declares an endpoint secure. The senior asks the IDOR question — *the attribute proves you may edit some product; what proves you may edit product 42?* — and looks for the imperative scope check, because that's the one that's manual, per-action, and forgettable. Then the senior asks the default-direction question: unmapped entity → visible everywhere; new admin action with no attribute → open to anyone past the area gate. `grep -L "CheckPermission" src/Presentation/Nop.Web/Areas/Admin/Controllers/*.cs` lists 8 of 63 files (2026-10-06) that rely on the area gate alone — a concrete list to review. Fail-open defaults are where audits earn their keep.

## Exercise ladder

1. **Goal:** reproduce the login gate order.
   **Check:** without looking at the table, list gates 4–9 from [CustomerRegistrationService.cs#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194); then compare.
2. **Goal:** determine whether lockout is on in a fresh install.
   **Check:** cite [InstallRequiredData.cs#L1627](../../src/Libraries/Nop.Services/Installation/InstallRequiredData.cs#L1627) and the `> 0` guard at [CustomerRegistrationService.cs#L161](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L161). Answer: off.
3. **Goal:** trace a permission check to its cache key.
   **Check:** from `[CheckPermission(StandardPermission.Catalog.PRODUCTS_CREATE_EDIT_DELETE)]` on [Admin ProductController.cs#L1188](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L1188), follow to `PrepareKeyForDefaultCache(NopSecurityDefaults.PermissionAllowedCacheKey, name, roleId)` at [PermissionService.cs#L336](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L336), and name the cache consumer that must evict it when a mapping changes (look in `Nop.Services/Security/Caching`).
4. **Goal (mid):** audit one action that lacks a permission attribute.
   **Check:** pick a file from the `grep -L` list above; for each action state which of the three mechanisms fire and what a *valid vendor account* could reach.
5. **Goal (mid):** write a test plan for lockout.
   **Check:** confirm no existing test covers it — `grep -ci lock src/Tests/Nop.Tests/Nop.Services.Tests/Customers/CustomerRegistrationServiceTests.cs` returns 0 (2026-10-06); the file's tests are `EnsureOnlyRegisteredCustomersCanLogin` and three password-format tests plus `CanChangePassword`. Your plan sets `FailedPasswordAllowedAttempts`, fails N times, asserts `LockedOut`, and restores settings in teardown.

## Senior findings (verified in source)

**F1 — Account-state enumeration through distinct error messages.** The controller deliberately collapses not-found / not-registered / wrong-password into one message, but returns distinct messages for `Deleted`, `NotActive` and `LockedOut` ([CustomerController.cs#L509-L520](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L509-L520)). Since those states are evaluated *before* the password ([CustomerRegistrationService.cs#L146-L155](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L146-L155)), any caller who submits an email with a garbage password learns whether that email belongs to a deleted, inactive or locked account. And if lockout is enabled, an attacker can *cause* the `LockedOut` state for a guessed email and then observe the distinct message — turning lockout into an existence oracle. Severity: low-to-moderate (user enumeration). It is a defensible UX trade-off (telling a real user "you're locked out" is kind), so the senior move is to name the trade-off and offer the alternative (generic message + email notification to the account owner, which the code already supports via `NotifyFailedLoginAttempt`, [L530-L538](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L530-L538)).

**F2 — Add-to-cart has side effects before validation.** `AddToCartAsync` calls `ResetCheckoutDataAsync(customer, storeId)` at [ShoppingCartService.cs#L1585](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1585) — *before* any item warnings are computed (L1603, L1620). That reset clears the customer's selected shipping option, offered shipping options, pickup point, desired delivery date, payment method and reward-points choice ([CustomerService.cs#L673-L706](../../src/Libraries/Nop.Services/Customers/CustomerService.cs#L673-L706)). So an add-to-cart that is *rejected* (out of stock, maximum items) still wipes checkout selections. Separately, in the update branch required products are added (L1598) before the main item is validated (L1603), and in the new-item branch they are added (L1628) before the maximum-items check (L1637) — so a rejected add can leave required products in the cart. Confidence: high on ordering (read directly); user impact is a confusing checkout, not a security issue. The general rule: **validate, then mutate.**

**F3 — Vendor accounts bypass storefront visibility via "admin preview".** Documented in [01-codebase-cartography/03-domain-glossary.md](../01-codebase-cartography/03-domain-glossary.md): the preview override at [ProductController.cs#L146-L148](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L146-L148) checks permissions that the Vendors role holds by default, without the vendor-ownership check used two blocks later.

## Debugging scenario: "a customer says they were locked out, but we never enabled lockout"

1. **Confirm the setting per store.** `FailedPasswordAllowedAttempts` is a *setting* (DB, per store), not an app setting; the installer value is 0, but an admin may have changed it for one store only. Check the settings table for that key and every store id.
2. **Read the lockout condition.** `CannotLoginUntilDateUtc > DateTime.UtcNow` ([L154](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154)) — so a lockout *timestamp* can persist from a period when lockout was on, even after it was turned off. Turning the setting back to 0 does not clear existing timestamps; only a successful login does (L188), and a locked customer cannot reach that line.
3. **Check for lost updates.** The counter is read-modify-write on the entity ([L160-L170](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160-L170)) with no concurrency token; parallel bad-password requests can under-count (never over-count), so "locked out after fewer attempts than configured" is *not* explained by the race — look elsewhere (step 1 or 2).
4. **Fix and prevent:** clear `CannotLoginUntilDateUtc` for the affected customer; propose that changing `FailedPasswordAllowedAttempts` to 0 also clears active lockouts (or that the check at L154 is skipped when lockout is disabled).

Interview cross-links: [API/data cards Q-auth](../08-interview-prep/03-api-and-data-modeling-questions.md), security checklist → [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md).

## Verification

2026-10-06: every anchor opened and checked; installer defaults read from `InstallRequiredData.cs` (L1627, L2044, L2053); message mapping read from `CustomerController.cs` L487–L528; the absence of lockout tests confirmed with grep. F2's ordering and F1's message mapping are read directly from source; their user impact was not reproduced in a running store.
