# Annotation Drills

Eight drills. For each: open the cited range, and annotate — on paper or in scratch comments — five things: **inputs** (and who controls them), **outputs**, **dependencies**, **invariants** (what must stay true), **side effects & failure modes**. Then grade yourself with the rubric at the bottom. Do at most two per sitting; the value is depth, not coverage.

---

## Drill 1 — [CustomerRegistrationService.ValidateCustomerAsync#L138-L194](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L138-L194)

Focus questions while annotating:
- Which input is attacker-controlled? (Both.)
- Order the checks L144–L155. Which reordering would create a vulnerability?
- L160–L170: what state is written on *failure*? Why must failure writes exist at all?
- Invariant to find: lockout state must be evaluated before password work.
- Failure mode to find: two simultaneous wrong-password requests read the same `FailedLoginAttempts`.

## Drill 2 — [ShoppingCartService.AddToCartAsync#L1549-L1616](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1549-L1616)

- Inputs: eight parameters — which combinations are mutually exclusive? (rental dates vs non-rental; wishlistId vs cart type.)
- Output: why `IList<string>` and not `bool` or an exception? What does the empty list *mean*?
- Find the line where validation switches from "cheap checks" to "expensive checks" and say why the order matters.
- Invariant: an existing identical line item is merged, never duplicated (L1589–L1593). What defines "identical"?
- Failure mode: max-items check (L1637) happens only on the insert branch — construct the sequence that exceeds the max anyway.

## Drill 3 — [EntityRepository.InsertAsync + UpdateAsync + DeleteAsync#L340-L455](../../src/Libraries/Nop.Data/EntityRepository.cs#L340-L455)

- Dependency to notice: `IEventPublisher` in a *data-layer* class. What layering rule does that bend, and what does the system buy?
- Invariant: every default-parameter write publishes an event. List every way a caller can break that invariant (the `publishEvent` flag, the predicate delete at L497, raw `Table` usage).
- Side effect chain: write → event → [CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) eviction. What happens if a consumer throws? (Check [EventPublisher.cs#L27-L50](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L27-L50) before answering.)

## Drill 4 — [OrderProcessingService.PlaceOrderAsync#L1594-L1677](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1594-L1677) (stop before the mutex)

- Annotate each awaited call in the success path with: reversible / irreversible / compensatable.
- Output: `PlaceOrderResult` collects errors — trace one thrown exception (L1661) and state exactly what the customer sees vs what the log sees.
- Invariant: `OrderGuid` non-empty (L1598). What is that guid *for*? (Idempotency toward the payment plugin.)
- Failure mode: rank the steps by "worst place for the process to die" and justify your #1.

## Drill 5 — [QueuedMessagesSendTask.ExecuteAsync#L41-L90](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L41-L90)

- Inputs: none — so what *is* the input? (Table state. Annotate the query at L44–L45 as the real parameter list.)
- Invariant: `SentTries` increments on every attempt, success or failure (`finally`, L83–L87). Prove the loop terminates for a permanently failing email.
- Side effect: SMTP send is external and *not idempotent* — find the crash window where an email could be sent twice. (Sent succeeds → process dies before `SentOnUtc` persists.)
- Dependency: `emailAccount.MaxNumberOfEmails` (L51) — throttling; what happens when it's 0?

## Drill 6 — [ProductController.ProductDetails#L129-L203](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L129-L203)

- Annotate every early return with the HTTP semantics it produces (404, redirect-permanent, local redirect) and *why that particular one*.
- The admin bypass (L146–L148): write the precise boolean expression for "page is visible."
- Side effects on this GET: list all three (L181, L190, L194) and classify: which would be wrong to execute on a cached/CDN'd response?
- Failure mode: `PrepareProductDetailsModelAsync` recursion for associated products ([factory L1727](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1727)) — what input data makes this expensive?

## Drill 7 — [CheckPermissionAttribute#L16-L130](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16-L130)

- Inputs: permission system names as strings — where are the constants defined, and what happens on a typo? (Runtime deny? runtime allow? find out.)
- Invariant: filter must run before the action body, always. What MVC mechanism guarantees it?
- Failure mode: an action with *no* attribute. Trace what gates remain (only the area-level [AuthorizeAdmin](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L89)) and state the resulting risk in one sentence.
- Note the duplicate-filter guard (L113) — why would the same filter run twice?

## Drill 8 — [MemoryCacheManager.GetAsync#L132-L175](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132)

- Inputs: `CacheKey` + acquire delegate. Annotate what the delegate *captures* (closure) and why a captured scoped service inside a long-lived cache would be a bug.
- Invariant to test for: is the acquire function guaranteed to run at most once per key under concurrency? Read the code; answer with line evidence, not hope.
- Failure mode: acquire throws — is anything cached? Is the exception cached (negative caching)?
- Output: cached value with `CacheTime` from the key — who decided that TTL, and how far away is that decision from this code?

---

## Self-grading rubric (apply per drill)

| Level | Observable behavior |
| --- | --- |
| Basic | Correctly lists inputs/outputs and names the method's job; anchors claims to line numbers. |
| Solid | Identifies at least one invariant *not stated in comments*, one realistic failure mode, and every side effect including logs/events. |
| Strong | Explains why the code is shaped this way (what alternative was rejected), constructs a concrete failure scenario with inputs and timing, and can say what test would pin the invariant. |

Interview angle: annotation is the skill behind "walk me through unfamiliar code" screens. The habit to build: say inputs → outputs → side effects → invariants *in that order*, out loud, before opining on quality.
