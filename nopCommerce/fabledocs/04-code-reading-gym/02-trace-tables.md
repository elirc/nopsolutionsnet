# Trace Tables

Fill in the table yourself first (columns: step, file/line, value shape, owner, transformation, risk), then compare with the filled reference rows. The discipline: at every step you must be able to say *what the data looks like right now* — most bugs are shape assumptions.

## Trace 1 (UI → API): quantity, from input box to database

The user types "3" on a product page and clicks Add to Cart. Follow the *quantity*.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | product page form (Razor) | `"3"` string in form field `addtocart_{id}.EnteredQuantity` | browser | user input | attacker controls it |
| 2 | [ShoppingCartController.cs#L858](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L858) | `IFormCollection` entry | MVC | `ParseEnteredQuantity` → `int` | parse failure default? (open the parser and find out) |
| 3 | [ShoppingCartService.cs#L1578-L1582](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1578-L1582) | `int quantity` | service | validated `> 0` | negative/zero blocked; upper bound checked later in warnings |
| 4 | [#L1596](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1596) | `newQuantity = existing + 3` | service | **merged** with existing line | user "adds 3" but cart may jump to 5 — UI surprise |
| 5 | [#L1612](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1612) | `ShoppingCartItem.Quantity` | entity | assignment | — |
| 6 | [EntityRepository.UpdateAsync#L393](../../src/Libraries/Nop.Data/EntityRepository.cs#L393) | SQL parameter | data | UPDATE + event | last-write-wins vs concurrent tab |

Your turn: repeat for the *price* on the same flow and find the step where customer-entered price is only honored if the product allows it.

## Trace 2 (persistence): a new order's `OrderTotal`

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [PlaceOrderAsync#L1602](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1602) | `PlaceOrderContainer` decimals | service | totals computed from cart + discounts + tax + shipping | **never from client input** — the invariant |
| 2 | [ProcessPaymentAsync call #L1424](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1424) | `ProcessPaymentRequest.OrderTotal` | payment plugin | sent to gateway | gateway rounds differently? reconciliation risk |
| 3 | [SaveOrderDetailsAsync#L723](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L723) | `Order.OrderTotal` decimal | entity | snapshot persisted | immutable from here — refunds are new records |
| 4 | admin/order pages | view model string | factory | currency formatting, localization | display-only |

Your turn: where would a *discount applied after payment but before save* be caught — or would it? Trace the container's lifetime to answer.

## Trace 3 (auth): `FailedLoginAttempts` under attack

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [ValidateCustomerAsync#L157](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L157) | entity `int`, e.g. 3 | service | read via customer lookup | read is cache/context-dependent |
| 2 | [#L160](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L160) | 4 | service | `++` in memory | two parallel requests both compute 4 |
| 3 | [#L161-L168](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L161-L168) | threshold compare | service | maybe sets `CannotLoginUntilDateUtc`, resets to 0 | boundary: `>=` with configurable threshold |
| 4 | [#L170](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L170) | UPDATE | repository | persist + entity event | lost update under parallelism |
| 5 | next attempt [#L154](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154) | `DateTime?` compare | service | lockout enforced | clock source = app server UTC |

Your turn: write the SQL a *race-proof* version would use instead of steps 1–4. (One `UPDATE ... SET FailedLoginAttempts = FailedLoginAttempts + 1 ... ; SELECT` — say why that fixes it.)

## Trace 4 (error path): payment declined during checkout

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | payment plugin | `ProcessPaymentResult{Success=false, Errors=[...]}` | plugin | gateway response → result object | plugin quality varies |
| 2 | [PlaceOrderAsync#L1652-L1659](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1652-L1659) | localized strings appended | service | errors → `PlaceOrderResult.Errors` | no order row created (good) |
| 3 | [#L1670-L1674](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1670-L1674) | aggregated log string | logger | errors → error log with customer | PII in logs? check |
| 4 | checkout controller/view | model errors | UI | rendered to customer | message must not leak gateway internals |

Your turn: contrast with the *exception* path (L1661-L1665) — same destination, different entry. What class of failure takes each path?

## Trace 5 (cache): product name after an admin edit

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | admin edit POST | `ProductModel.Name` string | admin UI | model-bound, validated | |
| 2 | Mapster mapping → entity | `Product.Name` | admin controller/factory | model → entity | mapping config drift (AutoMapper→Mapster migration was commit `99997cecd6`!) |
| 3 | [UpdateAsync#L393](../../src/Libraries/Nop.Data/EntityRepository.cs#L393) | UPDATE + `EntityUpdatedEvent<Product>` | repository | persist + publish | `publishEvent:false` would skip step 4 |
| 4 | [CacheEventConsumer#L44-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) | prefix evictions | consumer | by-id key removed, prefixes cleared | derived caches must share prefixes |
| 5 | next storefront request | cache miss → fresh entity | [MemoryCacheManager#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) | re-acquire + store | multi-node: *other* nodes never heard the event |

Your turn: extend the table for a two-node deployment with per-node memory cache and mark the exact step where correctness dies. Then name the config change that fixes it (distributed cache) and its cost.

---

Rubric per trace — Basic: fills shapes correctly. Solid: identifies each owner change (that's where contracts live) and one real risk per table. Strong: answers the "your turn" without hints and can narrate the trace aloud in under two minutes — that narration *is* the interview skill.
