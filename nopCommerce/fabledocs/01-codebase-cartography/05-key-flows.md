# Key Flows

Six end-to-end traces. Each is used elsewhere in the curriculum, but this page is the canonical version. Read a flow *with the files open* — the tables are maps, not substitutes.

---

## Flow 1: Product details page (UI/read path)

Why this flow matters: it is the highest-traffic read path in any store, and it demonstrates the repo's three signature moves in one screen: layered authorization, the model-factory pattern, and cache-backed reads.

Open these files first:
- [ProductController.cs#L129-L203](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129-L203) — the whole controller action
- [ProductModelFactory.cs#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502) — entity → view model
- [EntityRepository.cs#L122](../../src/Libraries/Nop.Data/EntityRepository.cs#L122) — cached `GetByIdAsync`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | MVC routing | Nop.Web.Framework | slug route resolves to `ProductDetails(productId)` | `int productId` | slug/id mismatch → 404 |
| 2 | Controller | ProductController.cs#L131-L133 | `GetProductByIdAsync`; null or `Deleted` → 404 | `Product` entity | soft-delete respected here, not in SQL |
| 3 | Controller | ProductController.cs#L135-L148 | availability gate: published + **ACL** (L139) + **store mapping** (L141) + date window; admins bypass (L146) | booleans | forgetting any one check leaks hidden products |
| 4 | Controller | ProductController.cs#L151-L160 | grouped-product redirect if not individually visible | redirect | permanent redirect — cacheable by proxies |
| 5 | Controller | ProductController.cs#L181-L195 | side effects: recently-viewed list, edit-link for admins, activity log | writes | side effects on a GET (see seniors' note) |
| 6 | Factory | ProductModelFactory.cs#L1502 | builds `ProductDetailsModel`: prices, attributes, pictures, associated products (recursive at L1727) | view model | fan-out of many service calls per render |
| 7 | View | Views/Product | Razor template chosen per product (L200) | HTML | template path comes from data |

Validation and authorization: three independent gates — publish state, [IAclService.AuthorizeAsync](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L139) (customer-role ACL), [IStoreMappingService.AuthorizeAsync](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221) (multi-store isolation). All three must pass; permission check at L146 creates a deliberate bypass for staff preview.

Persistence and side effects: reads are cached via repository cache keys; writes on this GET are the recently-viewed list and the activity log (L181, L194).

Tests that cover it: no direct controller test found; availability logic is covered at service level in [ProductServiceTests.cs#L78-L156](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L78-L156).

What juniors usually miss: that "can this customer see this product" is *three* checks, not one, and that each protects a different boundary (catalog state, role, store).

What seniors notice: side effects on a GET request (activity log, recently-viewed) — harmless here but a cache/CDN hazard; the admin bypass is an *authorization exception* that must be re-verified any time the gate changes; the factory's recursive call for associated products (L1727) is a potential N+1 amplifier.

Interview angle: "How would you implement product visibility rules in a multi-tenant catalog?" — walk this exact gate order, then name the bypass and its risk.

Drill: predict what happens for (a) a deleted product, (b) a published product mapped to another store, (c) an unpublished product viewed by an admin. Then verify each against the code.

Self-grade — Basic: names the 404 checks. Solid: explains ACL vs store-mapping as different boundaries. Strong: also flags the GET side effects and the admin bypass as review-worthy.

---

## Flow 2: Add to cart (server/API write path)

Why this flow matters: it's the repo's clearest example of *validation as data* (a warnings list instead of exceptions) and of the write → event → cache-eviction chain.

Open these files first:
- [ShoppingCartController.cs#L791-L874](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791-L874) — form parsing
- [ShoppingCartService.cs#L1549-L1684](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1684) — business rules
- [CacheEventConsumer.cs#L13-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L13-L53) — invalidation

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Controller | ShoppingCartController.cs#L793-L810 | load product; reject non-simple types | JSON early-returns | client depends on `{success, message}` shape — implicit contract |
| 2 | Controller | ShoppingCartController.cs#L814-L850 | find cart item being edited; verify it matches the product (L842) | `ShoppingCartItem?` | ownership check: item must belong to *this* customer's cart |
| 3 | Controller | ShoppingCartController.cs#L855-L864 | parse attributes/quantity/rental dates from raw `IFormCollection` | XML string (`attributesXml`) | attributes serialized as XML — legacy but load-bearing |
| 4 | Service | ShoppingCartService.cs#L1560-L1582 | permission gates (cart/wishlist enabled per customer), bot check, quantity > 0 | `List<string> warnings` | failures are *data*, not exceptions |
| 5 | Service | ShoppingCartService.cs#L1587-L1615 | find existing identical line → merge quantities, re-validate, update | entity update | validation runs against the *new total* quantity |
| 6 | Service | ShoppingCartService.cs#L1620-L1668 | else validate, enforce max items, insert new item | entity insert | max-items limit checked only on insert branch |
| 7 | Repository | EntityRepository.cs#L340-L355 / #L393 | insert/update + publish `EntityInserted/UpdatedEvent` | event | consumers run synchronously, in-process |
| 8 | Consumer | CacheEventConsumer.cs#L44-L53 | evict by-id and by-prefix cache entries | cache ops | miss a prefix → stale cart badge |

Validation and authorization: permission checks via [PermissionService.AuthorizeAsync](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L272) with an explicit `customer` argument (L1560, L1566); the deeper rule set (stock, attribute combinations, gift cards) lives in [GetShoppingCartItemWarningsAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1124).

Persistence and side effects: cart write, checkout-data reset (L1585), entity event → cache eviction.

Tests that cover it: **no direct `ShoppingCartService` test file found** (verified: [Nop.Services.Tests/Orders](../../src/Tests/Nop.Tests/Nop.Services.Tests/Orders) covers order services; a factory test exists at [ShoppingCartModelFactoryTests](../../src/Tests/Nop.Tests/Nop.Web.Tests/Public/Factories/ShoppingCartModelFactoryTests.cs)). This gap is a contribution opportunity.

What juniors usually miss: the warnings-list pattern — they expect exceptions; here the *caller* decides how to render failures, so business code stays UI-agnostic.

What seniors notice: no transaction wraps steps 5–8; the check at step 6 (max items) is read-then-write — two concurrent requests can both pass it (race). Also the implicit JSON contract in step 1 has no shared type with the client.

Interview angle: "Exceptions vs result objects for validation?" — this flow *is* the mid-level answer: expected business failures as data, exceptions for the unexpected.

Drill: list every distinct reason `AddToCartAsync` can refuse, with line numbers. (There are at least 7.)

Self-grade — Basic: 4+ reasons. Solid: all branches incl. update-vs-insert difference. Strong: also names the max-items race and where a transaction boundary would go.

---

## Flow 3: Login (auth/security boundary)

Why this flow matters: authentication with lockout, MFA hand-off, and enumeration-resistant error messages — everything a security-round interviewer wants to hear, in ~150 lines.

Open these files first:
- [CustomerController.cs#L457-L553](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L457-L553) — POST Login
- [CustomerRegistrationService.cs#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194) — `ValidateCustomerAsync`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Filter | CustomerController.cs#L458-L462 | `[ValidateCaptcha]`; action allowed even when store closed | attribute filters | auth endpoints must stay reachable |
| 2 | Controller | CustomerController.cs#L466-L467 | CAPTCHA failure → ModelState error | ModelState | |
| 3 | Controller | CustomerController.cs#L474-L479 | optional OTP-by-phone branch → redirect | redirect | separate flow, separate risks |
| 4 | Service | CustomerRegistrationService.cs#L140-L155 | lookup by username or email; check exists/deleted/active/registered/locked-out | `CustomerLoginResults` enum | ordered checks; lockout read at L154 |
| 5 | Service | CustomerRegistrationService.cs#L157-L173 | wrong password → increment `FailedLoginAttempts`, lock out after N tries, **persist** | entity update | counter write races under parallel attempts |
| 6 | Service | CustomerRegistrationService.cs#L175-L181 | MFA required? → return special result | enum | MFA state checked *after* password |
| 7 | Service | CustomerRegistrationService.cs#L187-L193 | success: reset counters, stamp `LastLoginDateUtc` | entity update | |
| 8 | Controller | CustomerController.cs#L489-L528 | switch on result; success → `SignInCustomerAsync` (cookie); each failure → localized generic-ish message | auth cookie | distinct messages for Deleted/NotActive/LockedOut — mild enumeration tradeoff |
| 9 | Controller | CustomerController.cs#L540-L542 | failed attempt → activity log | log row | observability for brute force |

Validation and authorization: this *is* authentication; authorization happens later per-request via [PermissionService](../../src/Libraries/Nop.Services/Security/PermissionService.cs#L258-L331). Note `WrongPassword`, `CustomerNotExist`, `NotRegistered` all collapse to one generic message (L521-L527) — deliberate anti-enumeration — while `LockedOut`/`NotActive` do not.

Persistence and side effects: failed-attempt counter and lockout timestamp are *persisted state* (survives restarts, works across nodes); activity log; optional failed-login notification email (L530-L538).

Tests that cover it: `CustomerRegistrationServiceTests` under [Nop.Services.Tests/Customers](../../src/Tests/Nop.Tests/Nop.Services.Tests/Customers).

What juniors usually miss: lockout must be checked *before* password verification (L154 before L157) or attackers keep free guesses during lockout.

What seniors notice: the counter increment (L160) is read-modify-write on the entity — concurrent wrong-password requests can under-count; the distinct LockedOut message confirms an account exists (accepted tradeoff for UX); password comparison delegates to hashed storage via `GetCurrentPasswordAsync`.

Interview angle: "Design login with brute-force protection." Narrate steps 4–7 and name the tradeoffs above — that's a strong mid-level answer.

Drill: an attacker scripts 100 wrong passwords against one account. Walk the code and state exactly what happens on attempt 1, N, N+1, and after lockout expiry.

Self-grade — Basic: describes lockout. Solid: correct check order and persisted state. Strong: names the race, the enumeration tradeoff, and what you'd log/alert on.

---

## Flow 4: Place order (persistence + money path)

Why this flow matters: the highest-stakes write in the system — payment, order creation, inventory, notifications — and home to the repo's most interview-worthy concurrency code.

Open these files first:
- [OrderProcessingService.cs#L1594-L1719](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594-L1719) — `PlaceOrderAsync`
- [OrderProcessingService.cs#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723) — `SaveOrderDetailsAsync`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Service | L1596-L1602 | guard clauses; build `PlaceOrderContainer` (totals, cart, customer) | container object | totals computed before payment |
| 2 | Service | L1610-L1612 | process payment via plugin (`ProcessPaymentAsync` at L1424) | `ProcessPaymentResult` | external call — can succeed while later steps fail |
| 3 | Service | L1616-L1621 | create `Order`, move cart items → order items | entities | **no wrapping DB transaction visible here** |
| 4 | Service | L1623-L1631 | discount/gift-card usage history; recurring payments | entities | partial-failure leaves history inconsistent |
| 5 | Service | L1634-L1644 | notifications, reset checkout, activity log, publish `OrderPlacedEvent` | event | consumers run in-process, synchronously |
| 6 | Service | L1647-L1650 | check status; if paid → `ProcessOrderPaidAsync` | | |
| 7 | Service | L1661-L1665 | any exception → log + collect into `PlaceOrderResult.Errors` | result object | payment may already be captured when this fires |
| 8 | Service | L1679-L1718 | optional double-submit guard: **named `Mutex` per customer id** + short-lived cache flag; note `.Result`/`.Wait()` (L1698, L1707, L1710) | lock | single-process only; sync-over-async is deliberate (comment L1685-L1688) because mutexes can't cross `await` |

Validation and authorization: upstream in checkout controllers; `PlaceOrderAsync` trusts its request but re-derives totals itself (step 1) — the client never supplies prices. That is the invariant to quote in interviews: **money amounts are computed server-side from persisted state, never accepted from the request**.

Persistence and side effects: order + items + histories + notes; queued emails (via Flow 5); `OrderPlacedEvent` fan-out to any `IConsumer<OrderPlacedEvent>`.

Tests that cover it: `OrderProcessingServiceTests` under [Nop.Services.Tests/Orders](../../src/Tests/Nop.Tests/Nop.Services.Tests/Orders).

What juniors usually miss: the error handling *collects* errors into a result rather than throwing, because a failed order must render a friendly page, not a 500 — same philosophy as Flow 2.

What seniors notice: payment capture (step 2) precedes order persistence (step 3) with no distributed transaction — a crash between them means captured money with no order; mitigations (idempotent `OrderGuid` at L1598, the lock, support tooling) reduce but don't eliminate it. The named `Mutex` only works on a single node — a load-balanced deployment needs the distributed-lock variant ([DistributedCacheLocker](../../src/Libraries/Nop.Core/Caching/DistributedCacheLocker.cs)).

Interview angle: this is a complete "design checkout" system-design answer *and* a "tell me about a risky pattern you've seen" behavioral answer. Full walkthrough: [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: draw the failure timeline — for each step 2–6, state what the customer sees and what data is inconsistent if the process dies right after that step.

Self-grade — Basic: sequence correct. Solid: identifies the payment-before-persist gap. Strong: proposes idempotency-key + reconciliation or outbox redesign and can say why nopCommerce likely didn't.

---

## Flow 5: Queued email send (background/async)

Why this flow matters: a complete outbox pattern in 93 lines — the single best file in the repo for learning async reliability vocabulary.

Open these files first:
- [QueuedMessagesSendTask.cs](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs) — the whole file
- [ScheduleTaskRunner.cs#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115) — how tasks are run

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Any flow | e.g. order placement | emails are **inserted into `QueuedEmail` table**, not sent inline | DB row | outbox: send failure can't fail the order |
| 2 | Scheduler | TaskScheduler/ScheduleTaskRunner.cs#L115 | periodic tick runs `IScheduleTask.ExecuteAsync`, optionally with run-once-per-period guard | | overlapping runs if guard disabled |
| 3 | Task | QueuedMessagesSendTask.cs#L43-L45 | load unsent emails with `SentTries < 3` (`maxTries = 3`) | list, grouped by account | bounded retries → dead-letter by exhaustion |
| 4 | Task | L51 | per account, take `MaxNumberOfEmails` | batch | throttling / backpressure |
| 5 | Task | L60-L87 | try send → stamp `SentOnUtc`; catch → log; **finally → `SentTries += 1` and persist** | entity update | `finally` guarantees progress even on crash-free failure |

Validation and authorization: none needed — internal trust boundary; the queue rows were created by trusted code.

Persistence and side effects: SMTP send (external), `QueuedEmail` row updates.

Tests that cover it: queued-email service tests under [Nop.Services.Tests/Messages](../../src/Tests/Nop.Tests/Nop.Services.Tests/Messages); the task itself has no direct test found — *investigate*.

What juniors usually miss: why not just send the email inside `PlaceOrderAsync`? Answer: SMTP is slow and flaky; coupling it to checkout couples your conversion rate to your mail server.

What seniors notice: retry increments in `finally` mean a *successful* send also bumps `SentTries` (harmless, since `SentOnUtc` is set — but subtle); there's no exponential backoff (retry cadence = task schedule); a poison message costs 3 sends then silently stops — visibility of exhausted messages depends on an admin grid, not an alert.

Interview angle: "How do you send emails reliably from a web app?" — describe this exact table + scheduled sender + bounded retries, then upgrade it with backoff and dead-letter alerting.

Drill: the SMTP server is down for 2 hours. Walk the code: what happens to each queued email, and what state are they in afterward? What admin action recovers them?

Self-grade — Basic: outbox described. Solid: retry/`finally` mechanics exact. Strong: names the missing backoff and dead-letter alerting, and the idempotency question (can a customer get the same email twice? when?).

---

## Flow 6: Admin edits a product (authz + admin write path)

Why this flow matters: the admin area is a second application sharing the same services — this flow shows layered authorization (area gate + per-action permission) and the vendor-isolation rule.

Open these files first:
- [AuthorizeAdminAttribute.cs#L89-L103](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89-L103) — area gate
- [Admin ProductController](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/ProductController.cs#L943-L965) — per-action `[CheckPermission]`
- [CheckPermissionAttribute.cs#L106-L119](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L106-L119) — how the attribute authorizes

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Filter | AuthorizeAdminAttribute.cs#L89 | `ACCESS_ADMIN_PANEL` permission or `ChallengeResult` | | area-wide gate, applied via base controller |
| 2 | Filter | Admin ProductController.cs#L943, #L963 | `[CheckPermission(PRODUCTS_VIEW)]` vs `[CheckPermission(PRODUCTS_CREATE_EDIT_DELETE)]` | | read vs write permissions are **separate records** |
| 3 | Filter | CheckPermissionAttribute.cs#L119 | `PermissionService.AuthorizeAsync(systemName)` per required permission | bool | permission → role mapping cached |
| 4 | Controller | Admin ProductController.cs#L969 (BulkEditSave region) | vendor scoping: `GetCurrentVendorAsync` filters what a vendor may touch | | **row-level** authorization done in code, not by the attribute |
| 5 | Service/Repo | Nop.Services.Catalog → EntityRepository | update + entity event + cache eviction (same spine as Flow 2) | | admin writes invalidate storefront caches via the same events |

Validation and authorization: three layers — admin-area permission, action permission, vendor row-scope. The first two are declarative (attributes); the third is imperative and therefore the one juniors forget when adding new admin actions.

Persistence and side effects: standard repository spine; admin activity log.

Tests that cover it: **no `PermissionService` test file found** ([Nop.Services.Tests/Security](../../src/Tests/Nop.Tests/Nop.Services.Tests/Security) holds only `EncryptionServiceTests`); admin controllers are untested directly. Both are contribution opportunities.

What juniors usually miss: `[CheckPermission]` proves the user may edit *some* product; it never proves they may edit *this* product. Object-level checks (IDOR defense) are manual — step 4.

What seniors notice: the read/write permission split (L943 vs L963) is the pattern to copy for any new admin feature; declarative + imperative authorization layered like this is exactly what an interviewer means by "defense in depth."

Interview angle: "What's IDOR and how do you prevent it?" — point at step 4: attribute-level authz cannot express row ownership; you must filter by the authenticated principal's scope in the query.

Drill: find one more admin controller that does vendor scoping and one that has no row-level concern (why not?).

Self-grade — Basic: two authz layers named. Solid: all three, with the declarative/imperative distinction. Strong: can state the IDOR rule and where a new admin endpoint could silently miss it.
