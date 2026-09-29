# Systematic Debugging

The method, then five repo-specific scenarios to rehearse it on. Debugging in interviews (and on the job) is graded on *process*, not luck: *reproduce → narrow → hypothesize → test cheaply → fix the root cause → add regression coverage*. Narrate those six words out loud and you're already above the bar.

## The method, tuned to this repo

1. **Reproduce.** Get it failing on demand. In this repo: local run with sample data, or an NUnit test against the SQLite base ([BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L116)) — a failing test is a reproduction *and* the future regression test.
2. **Narrow the layer.** The architecture gives you clean bisection points: view → controller → model factory → service → repository → SQL. Ask "is the bad value already bad here?" at each seam.
3. **Hypothesize one cause.** Written down, falsifiable.
4. **Test cheaply.** Cheapest probes first: read the code path; check the **Log** (admin → System → Log — most services log via `ILogger`, e.g. [OrderProcessingService.cs#L1663](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1663)); check the **activity log**; *then* attach the debugger.
5. **Fix the root cause,** not the symptom — and check the fix against every caller (`virtual` methods here have plugin callers you can't see).
6. **Add regression coverage** in the matching test folder.

Repo-specific tools: admin System→Log and activity log; `dotnet test --filter` (__inferred__); DB query logging via your provider; browser devtools for the jQuery/AJAX storefront calls; and the **cache suspicion rule** — when data is stale but the DB is right, suspect a missed [CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) prefix before anything else.

---

## Scenario 1: "I updated a product's price in admin but the storefront still shows the old price"

Reproduction: edit price in admin, load product page, price unchanged; DB row shows new price.
First question: is the bug in the write path (admin) or the read path (cache)?
Narrowing path:
1. Check DB → new price present → write path is fine; it's a read/cache problem.
2. Restart app → correct price appears → definitively a cache-invalidation miss (memory cache died with the process).
3. Now find *which* cache: the by-id entity cache is evicted by [CacheEventConsumer.ClearCacheAsync#L44-L53](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53) on update — so suspect a *derived* cache (price calculation, category product lists) whose prefix wasn't in the product's consumer.
4. Grep `Nop.Services/Caching` for the Product cache consumer; list evicted prefixes; diff against the cache key used by the price display path.
Useful probes: temporarily log cache hits/misses; check whether the update went through `UpdateAsync` with `publishEvent: false` ([EntityRepository.cs#L393](../../src/Libraries/Nop.Data/EntityRepository.cs#L393)) — no event, no eviction.
Likely root causes: derived cache key missing from consumer; update path bypassing repository events; load-balanced node with per-node memory cache (check deployment before blaming code).
Regression test to add: service-level test — update entity, assert dependent cached query returns fresh data.
Senior lesson: "stale after write, fresh after restart" is the signature of invalidation bugs; the class of fix (add prefix to consumer) matters more than the instance.
Interview version: narrate the restart-bisection — it cleanly separates "wrong data" from "stale data" in one move.

## Scenario 2: "Customers report being locked out even with the right password"

Reproduction: intermittent; support tickets only.
First question: is lockout being *set* wrongly (write) or *checked* wrongly (read)?
Narrowing path:
1. Pull one affected customer's row: `FailedLoginAttempts`, `CannotLoginUntilDateUtc`. Future-dated lockout with correct password logins in the activity log? → set wrongly.
2. Read [ValidateCustomerAsync#L154-L172](../../src/Libraries/Nop.Services/Customers/CustomerRegistrationService.cs#L154-L172): lockout check compares `DateTime.UtcNow`; the lockout write uses `AddMinutes(FailedPasswordLockoutMinutes)`.
3. Hypothesis A: settings misconfigured (lockout minutes huge) — check `CustomerSettings` in admin. Cheap.
4. Hypothesis B: bot/credential-stuffing traffic is failing logins against the account (each wrong attempt increments the counter — the *customer* isn't the only source of attempts). Check activity log `PublicStore.FailedLogin` entries ([CustomerController.cs#L540](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L540)) for the account.
5. Hypothesis C: server clock/UTC drift — verify `UtcNow` on all nodes.
Likely root causes: B is most common in the wild — the customer is collateral damage of an attack on their email.
Regression/mitigation: not a code bug → mitigation is CAPTCHA-after-N-failures or IP throttling; document the finding.
Senior lesson: "bug reports" can be *correct behavior under adversarial input*. Verify who is generating the events before touching the code.
Interview version: this narrates beautifully — three hypotheses, each with a one-query test, ordered by cost.

## Scenario 3: "Order confirmation emails sometimes never arrive"

Reproduction: intermittent, per-customer.
First question: were the emails *queued* (write side) or queued-but-not-*sent* (drain side)?
Narrowing path:
1. Find the order → check `QueuedEmail` rows for it. No row → enqueue path bug (check message templates active, message settings). Row exists → drain problem.
2. Row exists: inspect `SentOnUtc` (null?) and `SentTries`. Tries = 3, never sent → exhausted retries ([maxTries at QueuedMessagesSendTask.cs#L43](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L43)); check the Log for the send exceptions (L81).
3. Tries = 0 → the task isn't running: admin → System → Schedule tasks; check last run time of the send task; check the task is enabled and the runner is alive ([ScheduleTaskRunner#L115](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115)).
4. SMTP exceptions found → test the email account settings (admin has a "send test email").
Likely root causes: SMTP auth/quota errors exhausting the 3 tries during an outage window; task disabled after a deploy; email account misconfig.
Regression coverage: alerting is the real fix — see ticket 6 in [good-first-tickets](../06-contribution-practice/01-good-first-tickets.md).
Senior lesson: async pipelines need *state-based* debugging (inspect the queue table) not *log-first* debugging; the table is the truth.
Interview version: "queued vs drained" is the universal first question for any async delivery bug — email, webhooks, jobs.

## Scenario 4: "Product page is slow — sometimes 3+ seconds"

Reproduction: specific products only, worse on first load.
First question: is time spent in the DB, in the app, or in rendering?
Narrowing path:
1. "Worse on first load, fast after" → cache-miss cost. Which cache? Product page assembly is the factory: [PrepareProductDetailsModelAsync#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502).
2. Identify what's special about slow products: many attributes? many associated products (recursive prep at [L1727](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1727))? huge picture sets?
3. Cheap probe: enable provider query logging, count queries for a slow product vs a fast one. A 10x query count difference → N+1 in the factory fan-out (compare [the attribute-change loop, ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918)).
4. Confirm with a stopwatch around the factory call vs total action time.
Likely root causes: per-associated-product recursive prep; missing cache on a hot fragment; oversized images (check network tab first — it's free).
Regression coverage: a perf assertion is brittle; instead add query-count assertions if tooling allows, or document a budget.
Senior lesson: "measure first" is not a slogan — the network tab and a query counter cost minutes and regularly overturn the initial hypothesis.
Interview version: state your measurement plan *before* naming suspects; that ordering is what's being graded.

## Scenario 5: "Two identical orders from one customer, same minute"

Reproduction: rare; found in order reports.
First question: same `OrderGuid` or different? (This one question halves the space.)
Narrowing path:
1. Same guid → the same `ProcessPaymentRequest` was processed twice → look inside `PlaceOrderAsync` idempotency handling ([L1598](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1598)); different guids → the *checkout UI* submitted twice (double-click, retry on timeout) → two legitimate-looking requests.
2. For different guids: is `PlaceOrderWithLock` enabled ([L1679](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679))? If not, there's your window.
3. If enabled: is this deployment multi-node? The named `Mutex` (L1689) is per-process — two nodes take two different mutexes. Check the load balancer config; the code comment (L1685-L1688) never claimed cross-node safety.
4. Check payment gateway records — one capture or two? (Determines refund urgency vs bookkeeping fix.)
Likely root causes: multi-node deployment with the single-node guard; or lock disabled + slow payment gateway making users double-click.
Regression coverage: an integration test can't span nodes — the durable fix is a *unique constraint* on the idempotency key, which turns the race into a DB error you can test.
Senior lesson: concurrency bugs are diagnosed from *artifacts* (guids, timestamps, node ids in logs), not from reproduction; design the artifacts in beforehand.
Interview version: this is a debugging round and a system-design answer in one — finish with "the fix is making the database enforce uniqueness, because only it sees all nodes."

---

Grade yourself per scenario — Basic: reaches a plausible cause. Solid: bisects by layer with the cheapest probe at each step and names the exact file/line to inspect. Strong: also states the regression coverage and the *class* of bug, and could run the narration in an interview without notes.
