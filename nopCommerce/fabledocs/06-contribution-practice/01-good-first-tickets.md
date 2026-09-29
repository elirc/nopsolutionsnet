# Good First Tickets

Fifteen junior-scoped tickets, spread across the repo. All follow existing patterns, have small blast radius, and produce an interview story. Do them locally even if you never open a PR — the value is the reps. If you do submit upstream, read [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) first and check the project's GitHub issue conventions (nopCommerce tracks work as issues; drive-by PRs without an issue are often declined).

Common template abbreviations: **AC** = acceptance criteria, **ISP** = interview story potential.

---

## Ticket 1: Localize hardcoded English messages in add-to-cart JSON responses
Difficulty: Easy — Estimated time: 2–3 h
Skills practiced: localization pattern, grep discipline, small safe diffs.
Story: As a non-English store owner, I want cart error messages localized so my customers see them in their language.
Why this is a good contribution: [ShoppingCartController.cs#L808](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L808) returns hardcoded `"Only simple products could be added to the cart"`, and [ShoppingCartService.cs#L1562, L1568, L1574](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L1560-L1576) hardcode `"Shopping cart is disabled"`, `"Wishlist is disabled"`, `"Search engine can't add to cart"` — while neighboring messages use `_localizationService.GetResourceAsync` (e.g. L1580). The pattern to follow is on the very next branch.
AC:
- [ ] Each hardcoded string replaced with a `GetResourceAsync("ShoppingCart.Xxx")` call
- [ ] New resource keys added to the default-language seed (find where existing `ShoppingCart.*` keys live in [Nop.Services/Installation](../../src/Libraries/Nop.Services/Installation))
- [ ] Existing behavior unchanged for English
Read these anchors first: L1578-L1582 (the in-file pattern to copy).
What could go wrong: missing resource key renders the key name to users — verify seed + upgrade path both add it.
Review questions: do bot-facing messages (L1574) even need localization, or is that message never user-visible? (Defensible either way — say which and why.)
ISP: "I found inconsistent localization in a 2M-line codebase, followed the established pattern, and shipped it with seed-data migration."

## Ticket 2: Add a regression test for lockout-before-password-check ordering
Difficulty: Easy — Estimated time: 2 h
Skills practiced: NUnit, integration-style test base, security reasoning.
Story: As a maintainer, I want a test pinning that locked-out accounts are rejected *before* password verification, so a refactor can't silently reorder the checks.
Why: the ordering at [CustomerRegistrationService.cs#L154-L157](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154-L157) is security-relevant and only implicitly tested (*investigate existing coverage in* [Nop.Services.Tests/Customers](../../src/Tests/Nop.Tests/Nop.Services.Tests/Customers)).
AC:
- [ ] Test: locked-out customer + *correct* password → `LockedOut`, and `FailedLoginAttempts` unchanged
- [ ] Test: lockout expiry in the past + correct password → `Successful`, counters reset
Read these anchors first: [BaseNopTest.cs#L109-L116, L572](../../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L116) for how to get services and mutate customers.
What could go wrong: test DB seed state leaking between tests — reset the customer in teardown like [ProductServiceTests.cs#L47](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L47) does.
ISP: "I added a security regression test that encodes *why* the check order matters, not just what the code does."

## Ticket 3: Boundary test for product availability dates
Difficulty: Easy — Estimated time: 1 h
Skills practiced: reading existing test style, boundary analysis.
Story: As a maintainer, I want the exact-boundary case (AvailableEndDate == now) pinned so off-by-one regressions are caught.
Why: [ProductServiceTests.cs#L78-L156](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L78-L156) covers before/after but boundary equality coverage is worth verifying (*investigate*).
AC: one test per boundary (start==now, end==now), matching the file's naming style (`ShouldBeAvailableWhen...`).
ISP: "Smallest useful PR: I noticed untested boundary semantics and encoded a decision the code was making implicitly."

## Ticket 4: Extract `maxTries` magic number in queued email task
Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: settings pattern, minimal-diff refactoring.
Story: As a store owner, I want the email retry limit configurable (or at least a named constant) instead of a literal 3.
Why: [QueuedMessagesSendTask.cs#L43](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43) hardcodes `maxTries = 3`; the repo's settings pattern (`EmailAccountSettings` or a messages-related settings class) is the established home.
AC: constant/setting introduced; default remains 3; no behavior change.
What could go wrong: turning it into a *setting* touches seed + admin UI — the constant-only version is the honest junior scope; say so in the PR.
Review questions: is configurability actually wanted, or is a named constant enough? (Ask before building the bigger version — maintainers reject speculative settings.)
ISP: "I scoped a change *down* after considering blast radius — and can explain why."

## Ticket 5: Add missing test for `StoreMappingService.AuthorizeAsync` default behavior
Difficulty: Easy — Estimated time: 2 h
Skills practiced: multi-store model, generic constraints, test data setup.
Story: As a maintainer, I want the "entity with no store mappings is visible everywhere" rule pinned by a test.
Why: that default at [StoreMappingService.cs#L204-L221](../../src/Libraries/Nop.Services/Stores/StoreMappingService.cs#L204-L221) is a security-adjacent invariant people forget; a named test documents it.
AC: tests for unmapped entity (authorized), mapped-to-this-store (authorized), mapped-to-other-store-only (denied).
ISP: "I turned an implicit tenancy invariant into an executable spec."

## Ticket 6: Admin grid: surface `SentTries` for queued emails stuck at max retries
Difficulty: Easy/Medium — Estimated time: 3–4 h
Skills practiced: admin area, model factories, search models.
Story: As a store owner, I want to filter the queued-email admin list to "exhausted" messages so I notice silent failures.
Why: retries silently stop after 3 tries ([QueuedMessagesSendTask.cs#L43-L45](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43-L45)); the admin grid for queued email already exists — adding a filter follows the search-model pattern used everywhere in [Areas/Admin](../../src/Presentation/Nop.Web/Areas/Admin).
AC: filter checkbox "failed only"; grid column shows tries; no schema change.
What could go wrong: search-model plumbing touches factory + controller + view — copy an existing filter field end-to-end.
ISP: "I improved failure *visibility* for an async pipeline — observability instinct, cheaply."

## Ticket 7: Fix inconsistent early-return shape in AddProductToCart JSON
Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: implicit API contracts.
Story: As a frontend maintainer, I want all failure responses from add-to-cart to carry the same JSON shape.
Why: [ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810) returns `{redirect}` in one branch and `{success, message}` in another; audit all branches of the two AddProductToCart actions and document (or normalize) the contract. **Start by writing down the current contract — the fix may be docs, not code**, since storefront JS depends on the current shapes.
AC: a short markdown table of every response shape; only normalize if the consuming JS (search `wwwroot/js` for the ajax handlers) tolerates it.
ISP: "I mapped an implicit client-server contract before touching it — contract-first thinking."

## Ticket 8: Add `Interview story` — er, add activity-log coverage for wishlist adds
Difficulty: Easy — Estimated time: 2 h
Skills practiced: cross-referencing parallel flows.
Story: As a store owner, I want wishlist adds logged like cart adds so the activity log tells the full story.
Why: compare the cart path's activity logging with the wishlist path (verify with grep for `PublicStore.AddToWishlist` — it exists; confirm both AddProductToCart actions hit it). If coverage is already symmetric, the deliverable becomes a note — *verify before coding*, which is the actual lesson.
AC: evidence-based: either the missing log call added, or a comment in your notes proving symmetry.
ISP: "I verified an assumed gap before building — and was ready to close the ticket with 'no change needed.'"

## Ticket 9: Add a `dotnet test --filter` guide to the repo docs
Difficulty: Easy — Estimated time: 1 h
Skills practiced: developer experience, docs contribution.
Story: As a new contributor, I want a documented way to run one test class instead of the full suite.
Why: CI runs the whole suite ([dotnet.yml](../../.github/workflows/dotnet.yml)); a CONTRIBUTING note with filter examples lowers the entry bar.
AC: PR to CONTRIBUTING.md with 3 verified filter examples.
ISP: "My first PR improved onboarding — and I verified every command I documented."

## Ticket 10: Null-guard the email account lookup in QueuedMessagesSendTask
Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: defensive coding vs invariant reasoning.
Story: As an operator, I don't want the whole send batch to die because one email references a deleted email account.
Why: [QueuedMessagesSendTask.cs#L49-L51](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L49-L51) uses `emailAccount.MaxNumberOfEmails` without a null check after `GetEmailAccountByIdAsync`. *Label honestly:* this is a "possible risk" — first verify whether deletion of an in-use account is prevented elsewhere; if it is, document the invariant instead.
AC: either a skip-and-log branch for null accounts + test, or a written proof the state is unreachable.
Review questions: what should happen to those queued rows — skip forever, or reassign?
ISP: "I investigated a suspected crash path and either fixed it or proved the invariant — both outcomes are wins."

## Ticket 11: Consistent `StringComparison` in form-key scanning
Difficulty: Easy — Estimated time: 1 h
Skills practiced: micro-correctness, culture-sensitivity awareness.
Story: As a maintainer, I want form-key matching to use `OrdinalIgnoreCase` (convention for protocol strings) rather than `InvariantCultureIgnoreCase`.
Why: [ShoppingCartController.cs#L816](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L816) compares form keys with `InvariantCultureIgnoreCase`; ordinal is the .NET guideline for identifiers. Tiny, but check repo-wide convention first — if the codebase consistently uses invariant, **conform, don't crusade**.
AC: grep report of both usages; change only if ordinal is already the local majority.
ISP: "I learned that consistency with the codebase beats textbook correctness — and can defend when each applies."

## Ticket 12: Add pause-proof test for `PagedList` boundaries
Difficulty: Easy — Estimated time: 2 h
Skills practiced: pagination math, core utility testing.
Story: As a maintainer, I want page-index/size edge cases (0 items, exact multiple, out-of-range index) pinned on [PagedList.cs](../../src/Libraries/Nop.Core/PagedList.cs).
Why: every grid in the app flows through it; cheap high-leverage coverage (*investigate existing tests in* [Nop.Core.Tests](../../src/Tests/Nop.Tests/Nop.Core.Tests)).
AC: 4+ boundary tests in the existing style.
ISP: "I test infrastructure code where a bug multiplies across every page of the app."

## Ticket 13: Document the three-gate visibility rule for products
Difficulty: Easy — Estimated time: 1–2 h
Skills practiced: technical writing from code evidence.
Story: As a plugin developer, I want the published/ACL/store-mapping gate documented so my custom pages don't leak hidden products.
Why: the rule lives only in code ([ProductController.cs#L135-L148](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L135-L148)); plugin authors re-implement it wrong.
AC: short doc (or XML doc comments on the relevant service methods) naming all three gates + the admin bypass.
ISP: "I documented a security-relevant invariant discovered by reading, not being told."

## Ticket 14: Make `ProductDetails_AttributeChange` reuse batched product loads
Difficulty: Medium (stretch junior) — Estimated time: 3–4 h
Skills practiced: N+1 remediation with measurement.
Story: As a customer on a product with many attribute values, I want attribute-change price updates to stay fast.
Why: per-iteration awaited loads at [ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918); the repo already has [GetByIdsAsync](../../src/Libraries/Nop.Data/EntityRepository.cs#L157) for batching. **Measure first** — per-entity caching may already absorb it; your PR must include before/after query counts.
AC: batched load; measurement evidence; behavior identical.
What could go wrong: associated products may be cache-hot, making this a no-op perf win with real complexity cost — be prepared to close as won't-fix.
ISP: "I profiled before optimizing and let the data decide — the strongest possible perf story."

## Ticket 15: Add a smoke test asserting every admin mutating action carries a write permission
Difficulty: Medium (stretch junior) — Estimated time: 4 h
Skills practiced: reflection over controllers, security automation.
Story: As a maintainer, I want CI to fail if someone adds an admin POST action without a `[CheckPermission]` attribute.
Why: the gap fails *open* (see [pattern 8](../03-architecture-and-patterns/05-pattern-catalog.md)); a reflection test over [Areas/Admin/Controllers](../../src/Presentation/Nop.Web/Areas/Admin/Controllers) turns a review-checklist item into an invariant. Expect a legitimate exclusion list (login, dashboards) — the test's value is forcing new exclusions to be *explicit*.
AC: reflection test + documented exclusion list; passes on current tree.
ISP: "I automated a security review rule — this story lands in every interview."
