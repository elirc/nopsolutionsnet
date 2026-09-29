# Maintainer Communication

Open source communication compresses to one principle: **do the thinking, share the evidence, ask the smallest question.** Maintainers give minutes, not hours; your job is to make those minutes count. (Same principle applies to senior teammates at work.)

## Asking for help without outsourcing thinking

Bad: "How does caching work in nopCommerce?" (answer: a chapter).
Good: "I traced product cache invalidation through `EntityRepository.UpdateAsync` → `EntityUpdatedEvent` → `CacheEventConsumer.ClearCacheAsync` (evicts ByIds/All prefixes). For a *predicate* delete (`DeleteAsync(Expression<...>)`, EntityRepository.cs#L497) I don't see events published — is skipping invalidation intentional there, or is the caller responsible?"

The good version proves you read three files, states a falsifiable understanding, and asks one yes/no-with-nuance question. This template earns responses for life:

```
Context: what I'm trying to do (1 line)
What I found: files/lines traced, with my current understanding stated as claims
The specific question: one, narrow, answerable in <5 minutes
What I'll do with the answer: shows it's load-bearing, not curiosity
```

## Bug reports

Minimal repro or it didn't happen. Structure: **versions** (nopCommerce version, DB provider, deployment shape — single vs multi-node matters enormously here, see [key flow 4](../01-codebase-cartography/05-key-flows.md)); **steps** (numbered, from clean sample-data install); **expected vs actual**; **evidence** (Log-table entries, not screenshots of them; the row states for async issues — e.g. `QueuedEmail.SentTries` values); **suspected area** with anchors *labeled as suspicion*: "possibly related to the prefix eviction in CacheEventConsumer.cs#L46 — unverified."

The "labeled as suspicion" discipline matters: wrongly-confident diagnoses send maintainers down your rabbit hole. Confidence-labeled reports build the reputation that gets your future reports read first.

## Proposing a feature

Lead with the problem, not the solution: merchants hit X (evidence), current workaround is Y (cost), here's a sketch (small), *is this direction acceptable before I build it?* The pre-build check is the whole game — the most demoralizing OSS outcome is a rejected finished PR, and it's almost always self-inflicted by skipping the issue-first step. Include the maintainer's-eye view: blast radius, which existing pattern it follows, upgrade/migration impact ([ticket template](../06-contribution-practice/01-good-first-tickets.md) has the full list).

## Responding to review

Three moves, whatever the comment: (1) **acted**: "Done — also added the test you implied." (2) **push back with evidence, once**: "I kept the warnings-list style instead of throwing because every sibling method in ShoppingCartService returns warnings (L1549, L1124) — happy to change if you prefer consistency with X instead." (3) **escalate to a call/issue when a thread exceeds ~3 rounds** — text has failed at that point.

Never: silent force-pushes that invalidate review state; "fixed" without saying how; taking review personally (the code is being reviewed; you are not).

## Respectful disagreement

Formula: shared goal → evidence → cost you're worried about → proposal → genuine openness. "We both want checkout latency low. The sync consumers on OrderPlacedEvent add plugin-dependent latency (EventPublisher runs them inline). I'm worried a slow consumer costs conversions. Could we accept the durable-consumer RFC as opt-in, keeping current behavior default? I may be overweighting a rare case — do you see it differently?"

Interview angle: behavioral rounds ask "tell me about disagreeing with a senior engineer / receiving hard feedback." The formula above *is* the answer's skeleton; source concrete instances from your kata/ticket work and log them in [STAR stories](../08-interview-prep/06-behavioral-star-stories.md).
