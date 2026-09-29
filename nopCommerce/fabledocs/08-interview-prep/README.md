# 08 — Interview Prep

Target: **mid-level fullstack loops**, using this repo as your portfolio of concrete evidence. Contents:

| File | Round | Cards |
| --- | --- | --- |
| [01-csharp-dotnet-deep-dive.md](01-csharp-dotnet-deep-dive.md) | language/runtime | 16 |
| [02-frontend-framework-questions.md](02-frontend-framework-questions.md) | frontend | 10 |
| [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) | API + data | 12 |
| [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) | system design | full walkthrough + 5 variations |
| [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) | practical rounds | 7 timed sims |
| [06-behavioral-star-stories.md](06-behavioral-star-stories.md) | behavioral | 8 worksheets |
| [07-two-week-cram-plan.md](07-two-week-cram-plan.md) | all | day-by-day |

## How mid-level loops are structured

Typically: recruiter screen → technical screen (language + small coding) → onsite: practical coding (build/debug in a real-ish codebase), a design conversation (scaled to mid: one feature or one service, not planet-scale), a code-review or debugging round, and behavioral. Mid-level is *not* graded on exotic algorithms — it's graded on: writes working code, reasons about tradeoffs with examples, debugs methodically, communicates while doing all three.

## The golden rule

**Answer with a concrete example, a tradeoff, and a failure mode — never with a definition.** Compare:

- "Cache invalidation is removing stale entries when data changes." (definition — junior)
- "In nopCommerce, every repository write publishes an entity event and a consumer evicts key prefixes — invalidation is systemic, so new caches are correct by default. The tradeoff is coarseness: one product update flushes whole prefix families, so hit rate dips during bulk edits. And it's memory-cache-per-node, so it silently breaks under load balancing unless you switch to the distributed cache." (evidence + tradeoff + failure mode — this is the mid-level sound)

Every card in this module is built to produce answers of the second kind. ≥60% are anchored to real code you can name — that specificity is *rare* among candidates and reads as seniority.

## How to practice

Out loud, timed (90s per card answer), anchors open on second monitor for checking *after* answering. Rotate: never two sessions on the same file. The [cram plan](07-two-week-cram-plan.md) schedules all of it.
