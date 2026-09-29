# Debugging and Code-Review Round Simulations

Seven timed simulations converted from earlier modules. Run them like the real thing: timer on, talking out loud the whole time (silence is the #1 practical-round failure), notes only after time expires.

## Format cheat-sheet

**Debugging rounds** grade: reproduce-first instinct, hypothesis discipline (one at a time, cheapest test first), narration, and whether you finish with a regression-prevention statement. **Review rounds** grade: finding the layer-2 correctness issue, calibrating severity, and kind, specific language. Both grade *time management* — a perfect answer at minute 50 fails.

## Debugging sim 1 — stale price (20 min)

Prompt (read only this, then start): "Admin updated a price; storefront shows the old one. DB has the new value. Go."
Interviewer follow-ups to self-inject at minutes 8 and 14: "It fixes itself after app restart — what does that tell you?" / "Now it happens only on some requests — new hypothesis?"
Reference path: [scenario 1](../05-quality-engineering/03-systematic-debugging.md). The minute-14 twist answer: load-balanced nodes with per-node memory cache.
Pass bar: restart-bisection stated by minute 10; final answer names the eviction spine ([CacheEventConsumer](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L44-L53)) and a regression test.

## Debugging sim 2 — the double order (25 min)

Prompt: "Support found two identical orders, same customer, 40 seconds apart. Walk me through your investigation."
Self-inject at 10: "Different OrderGuids." At 18: "The deployment scaled to 3 nodes last month."
Reference: [scenario 5](../05-quality-engineering/03-systematic-debugging.md); the artifact-first question (same guid or not?) should be your opening move.
Pass bar: reaches the single-node-lock limitation ([L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718)) with the unique-constraint fix, and states why the fix is testable when the lock isn't.

## Debugging sim 3 — missing emails (15 min)

Prompt: "Some customers never get order confirmations."
Self-inject at 7: "The rows exist, SentTries is 3, SentOnUtc is null."
Reference: [scenario 3](../05-quality-engineering/03-systematic-debugging.md). Pass bar: "queued vs drained" asked in the first two minutes; ends with the alerting gap, not just the SMTP fix.

## Debugging sim 4 — intermittent lockouts (15 min)

Prompt: "Users complain they're locked out despite correct passwords."
Self-inject at 8: "Activity log shows hundreds of failed logins for those accounts from foreign IPs."
Reference: [scenario 2](../05-quality-engineering/03-systematic-debugging.md). Pass bar: recognizes correct-behavior-under-attack; mitigation is product-level (CAPTCHA thresholds, throttling), not a code "fix."

## Review sim 1 — the caching PR (20 min)

Review [kata 2](../04-code-reading-gym/04-review-katas.md) (cache customers by email in login) as a live round: read, then deliver the review verbally as if the author is present.
Pass bar: the security-state-staleness issue found and labeled blocking; measurement question asked; tone stays collegial while declining the PR. Follow-up to self-inject: "The author says login is genuinely slow in production — now what?" (Answer shape: profile *which part*; cache the safe reads, never the lockout state; or fix the underlying query.)

## Review sim 2 — the refactor PR (20 min)

Review [kata 5](../04-code-reading-gym/04-review-katas.md) (extract service, breaking `protected virtual`).
Pass bar: identifies the ecosystem-API break; proposes the additive+shim sequencing; explicitly *supports the direction* while blocking the mechanics — direction/mechanics separation is a senior review signature.

## Review sim 3 — the schema PR (15 min)

Review [kata 7](../04-code-reading-gym/04-review-katas.md) (IsFeatured flag, installation-schema-only).
Pass bar: catches the missing `UpgradeTo*` migration (existing stores break); catches the visibility-gates omission on the homepage query; both explained in terms of *which users* get hurt (upgraders; customers of stores using ACL).

---

Scoring across all seven — Basic: 4+ pass bars met. Solid: 6+, with narration you'd be comfortable playing back. Strong: all seven, plus each ends inside time with a one-sentence "how we prevent this class of issue" — reviewers and interviewers both remember endings.
