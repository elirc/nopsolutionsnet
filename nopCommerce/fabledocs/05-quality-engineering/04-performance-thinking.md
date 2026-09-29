# Performance Thinking

Rule zero: **measure first**. Every hotspot below is a *hypothesis with evidence pointers*, not a confirmed problem — treating hunches as facts is how teams waste quarters. The interview-grade habit: name the measurement before naming the fix.

## The performance domains here

| Domain | Where it lives | First measurement |
| --- | --- | --- |
| Server/DB | services + Linq2Db queries | query log; count queries per request |
| Cache effectiveness | [MemoryCacheManager](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) | hit/miss counters (add temporarily) |
| Render | Razor + model factories | stopwatch around factory vs action total |
| Network/assets | `wwwroot`, gulp bundles | browser devtools waterfall — free, do it first |
| Memory | in-proc cache, gen2 | dotnet-counters, gcdump |
| Background | schedule tasks sharing the process | task run durations in admin |

## Likely hotspots (each with its evidence and its measurement)

1. **Factory fan-out on product/catalog pages.** [PrepareProductDetailsModelAsync#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502) aggregates prices, attributes, pictures, tier prices, associated products (recursively, [L1727](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1727)). Measure: query count on a product with 20 associated products vs 0. Fix ladder: batch loads → fragment caching → flatten recursion.
2. **Awaited calls in loops.** Pattern instance: [ShoppingCartController.cs#L904-L918](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L904-L918). Sequential round-trips; but per-entity caching may already absorb them — measure before touching ([ticket 14](../06-contribution-practice/01-good-first-tickets.md)).
3. **Cache stampede on hot misses.** After an admin edit evicts a hot product's caches ([CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53)), concurrent requests may all run the acquire function. Measure: log acquire executions per key under load. Check first whether [MemoryCacheManager.GetAsync#L132](../../src/Libraries/Nop.Core/Caching/MemoryCacheManager.cs#L132) already single-flights (that's [annotation drill 8](../04-code-reading-gym/01-annotation-drills.md) — answer with line evidence).
4. **Prefix eviction breadth.** One product update clears whole prefix families — hit rate craters during bulk admin edits/imports. Measure: hit rate during a bulk import. Fix: batch import APIs already exist; consider suspending per-row eviction during bulk ops (risky — think it through).
5. **Unbounded queries.** The email task loads up to `int.MaxValue` rows ([QueuedMessagesSendTask.cs#L44-L45](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L44-L45)) then trims per account in memory. Fine at thousands; pathological after an outage builds a huge backlog. The general lesson: **every unbounded query is a time bomb with a fuse measured in business growth.**
6. **Sync-over-async under load.** The checkout mutex block ([L1689-L1716](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1689-L1716)) parks pool threads; at high concurrent checkout volume this can starve the pool. Measure: thread-pool queue length under load test. (Also one more reason for the constraint-based fix in [critique #1](../03-architecture-and-patterns/06-architecture-critique.md).)
7. **Missing indexes on custom queries.** Schema indexes are curated centrally (see [AddIndexesMigration](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/AddIndexesMigration.cs) — the pattern of shipping index fixes as migrations); any new filtered query you add needs the matching index *and* the migration.

## How to find each classic, in this stack

- **N+1:** count queries per request (provider logging). No EF lazy-loading here, so every query is an explicit call — grep `await` inside `foreach` as a static heuristic, then measure.
- **Serial async:** look for independent awaited calls in sequence; batch or `Task.WhenAll` (careful: parallel DB calls consume pool connections — measure both ways).
- **Expensive renders:** stopwatch the factory call vs total action time ([scenario 4 walkthrough](03-systematic-debugging.md)).
- **Oversized payloads:** devtools first; the gulp bundle contents ([package.json](../../src/Presentation/Nop.Web/package.json)) show what ships.
- **Memory growth:** dotnet-counters gen2/LOH trends; suspect cache growth and closure captures.

Interview angle: perf questions are judged on *method*: baseline → measure → hypothesize → change one thing → re-measure, plus "know your budget" (a 3s product page is a bug; a 3s admin bulk edit may be fine). Use hotspot 2 as your worked example — it ends with "and the measurement might tell me to do nothing," which is the strongest possible finish.
