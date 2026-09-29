# Review Katas

Eight simulated PRs. For each: read the intent and diff summary, write your review (aim for 10 minutes each), then compare with the expected findings. Grade findings as **Blocking** (must fix), **Important** (should fix or justify), **Optional** (style/nit). Practice the *language* too — reviews are graded on kindness and specificity in real interview review-rounds.

## Kata 1: "Add Twitter/X share button to product page"

Author intent: marketing wants a share button.
Fake diff summary: adds HTML to the product view; adds `ShareUrl` property computed *in the view* via `Model.Sku + Request.QueryString`; no factory change.
Files this resembles: [ProductModelFactory.cs#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502), product Razor views.
Expected findings — Blocking: URL built from raw query string interpolated into an href without encoding (XSS vector). Important: computation belongs in the factory, not the view (the repo's model boundary); multi-store base-URL handling (current store context, not request host, decides the canonical URL). Optional: localization of the button label via resource key.
Good review comment example:
> Building `ShareUrl` in the view interpolates `Request.QueryString` unencoded — that's an XSS vector on any crafted query. Could we compute it in `PrepareProductDetailsModelAsync` (where the other URL-ish fields live) and use the store's canonical URL? Happy to point at an example.

## Kata 2: "Speed up login by caching customers"

Author intent: reduce DB hits on login.
Fake diff summary: wraps `GetCustomerByEmailAsync` result in a 10-minute static cache keyed by email; used inside `ValidateCustomerAsync`.
Files this resembles: [CustomerRegistrationService.cs#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194).
Expected findings — Blocking: the method *writes* to that same customer (failed attempts, lockout, L160-L191) — a cached stale read makes lockout enforcement skippable; security state must never be cached without event-driven eviction. Important: was login DB load even measured? (premature); cache keyed by attacker-controlled input (email) = unbounded key space. Optional: none — this PR should be declined kindly with the measurement question.

## Kata 3: "Add `DeleteAllExpiredCartsAsync` maintenance method"

Author intent: cleanup job for abandoned carts.
Fake diff summary: adds `_sciRepository.DeleteAsync(x => x.UpdatedOnUtc < cutoff)` to a new schedule task; registers it.
Files this resembles: [EntityRepository predicate delete #L497](../../src/Libraries/Nop.Data/EntityRepository.cs#L497), [ScheduleTaskRunner](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115).
Expected findings — Important: the predicate `DeleteAsync` skips entity events → stale cart caches for affected customers (see [drill 3](01-annotation-drills.md)); unbounded delete on a huge table can lock — batch it; cutoff should come from a setting, not a constant. Blocking: none if events/batching addressed. Optional: activity-log entry for observability.
This kata's lesson: knowing *which repository overloads bypass the write spine* is repo-specific gold.

## Kata 4: "Return product stock in the AJAX price endpoint"

Author intent: show live stock on attribute change.
Fake diff summary: adds `stockQuantity = product.StockQuantity` to the anonymous JSON in `ProductDetails_AttributeChange`.
Files this resembles: [ShoppingCartController.cs#L879-L920](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L879-L920).
Expected findings — Important: exact stock counts are competitive/scraping-sensitive data — product pages elsewhere show ranges/flags; check `CatalogSettings` for existing display rules and match them. Important: anonymous JSON = implicit contract; who consumes it and is the JS updated in this PR? Optional: naming consistency with existing response fields.
Lesson: "small additive field" PRs are where data-exposure bugs live.

## Kata 5: "Refactor: extract SendNotificationsAndSaveNotesAsync into OrderNotificationService"

Author intent: OrderProcessingService is 3,561 lines; split it.
Fake diff summary: moves the method + 4 helpers to a new service; updates DI; changes the method from `protected virtual` to the new class.
Files this resembles: [OrderProcessingService.cs#L1634](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1634), [NopStartup.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195).
Expected findings — Blocking: removing a `protected virtual` member breaks every subclass override in the plugin/customization ecosystem ([pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)) — needs an obsolete-delegating shim and a release-notes entry, or a major-version target. Important: the extraction is good *direction*; sequence it as add-new + delegate-old. Optional: name choice.
Lesson: in extensible codebases, refactors are API changes.

## Kata 6: "Fix: double orders" (the tempting one)

Author intent: replace the Mutex block with `lock(_lockObject)`.
Fake diff summary: static object lock around `placeOrder(details)` call; removes the named Mutex.
Files this resembles: [OrderProcessingService.cs#L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718).
Expected findings — Blocking: `lock` cannot contain `await` (compile error as sketched — did the author `.Result` inside? then nothing improved); a single static lock serializes **all** customers' checkouts (the current design locks per-customer); still single-node. Important: the actual fix is idempotency via unique constraint (see [critique #1](../03-architecture-and-patterns/06-architecture-critique.md)); any lock change here needs load-test evidence. Optional: —.
Lesson: rejecting a fix requires offering the better one.

## Kata 7: "Add IsFeatured flag to Product"

Author intent: merchandising wants featured products.
Fake diff summary: adds `bool IsFeatured` to `Product`; adds column via edit to the *installation* schema; adds admin checkbox; filters homepage by it.
Files this resembles: [ProductBuilder.cs#L17](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L17), [Migrations/Installation vs UpgradeTo500](../../src/Libraries/Nop.Data/Migrations).
Expected findings — Blocking: editing installation schema only = existing stores never get the column → runtime SQL errors; needs an `UpgradeTo*` migration too (the dual-write of this repo's schema system). Important: homepage filter must respect the existing three visibility gates (published/ACL/store — [key flow 1](../01-codebase-cartography/05-key-flows.md)); cache consumer for the new homepage query prefix. Optional: default value semantics.
Lesson: every schema-touching PR here has two write points (installer + upgrade) — miss one and only *new* installs work.

## Kata 8: "Log full request body on payment errors"

Author intent: debugging payment failures is hard.
Fake diff summary: on payment failure, logs serialized `ProcessPaymentRequest` including card fields, to the error log.
Files this resembles: [PlaceOrderAsync error handling #L1652-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1652-L1674).
Expected findings — Blocking: card data in logs = PCI violation, full stop; log a correlation id + gateway response code instead. Important: error log is admin-visible and DB-persisted — retention/PII policy applies to *existing* fields too (customer linkage is already there, L1674 — worth an issue). Optional: structured fields over string concat.
Lesson: observability PRs need the same security review as feature PRs.

---

Rubric — Basic: finds the headline issue in ≥6 katas. Solid: correctly *grades* findings (blocking vs important — over-blocking is a real reviewer failure) and phrases one kind, specific comment each. Strong: every rejection comes with a better path, and you can cite the repo pattern each finding rests on. Interview note: katas 2, 6, 8 map directly to common review-round screens.
