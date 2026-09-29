# Behavioral STAR Story Worksheets

Eight worksheets. Stories 1–2 are complete from studying this repo alone; 3–8 become real once you do the corresponding ticket/kata — fill each worksheet **immediately after** finishing the work, while details (numbers, file names, wrong turns) are fresh. Concrete detail is what separates believed stories from generic ones.

Rehearsal check for every story: under 2 minutes? contains one number or file-level detail? ends with impact + what you'd do differently?

## Story 1: Learning a 2M-line codebase fast
Prompts: "Tell me about ramping up on unfamiliar/legacy code." / "How do you approach a new codebase?"
Source: this curriculum's cartography work.
Situation: needed working knowledge of nopCommerce — ~15-year-old, 40-subdomain .NET monolith.
Task: be able to trace and modify core flows safely within two weeks.
Action: read CI config before docs (30 lines told me build truth); mapped the DI registrations as a system index; traced two flows end-to-end (product page, add-to-cart) instead of reading broadly; kept a verification log with file/line anchors for every claim.
Result: could narrate checkout end-to-end including its concurrency guard and cache invalidation model; produced a risk register with 8 findings.
Senior-signal details: verification-log discipline (claims vs suspicions); choosing *flows* over *files* as the reading unit.
Resume bullet: Reverse-engineered and documented core order/caching/auth flows of a 2M-line production e-commerce platform, producing a prioritized architectural risk register.

## Story 2: Disagreeing with existing code respectfully (technical tradeoff)
Prompts: "A time you disagreed with a technical decision." / "Code you'd have written differently."
Source: the checkout mutex analysis ([PlaceOrderAsync L1679-L1718](../../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718)).
Situation: found sync-over-async + a named mutex on the money path — textbook "wrong."
Task: form a real judgment, not a reflex.
Action: read the authors' comment explaining the constraint (Mutex can't cross await, UNIX semaphore issue); reconstructed the deployment reality (single-host operators); designed the alternative (idempotency key + unique constraint) and *its* costs; concluded the code was a reasonable 2010s tradeoff whose replacement is now cheap.
Result: a written critique that steelmans the original before improving it.
Senior-signal: "I changed my assessment after finding the constraint" — interviewers rarely hear candidates say this and it lands hard.
Resume bullet: (supporting detail for design-review discussions, not a standalone bullet.)

## Story 3: Shipping a small fix with production discipline
Prompts: "Walk me through a recent PR." / "Attention to detail."
Source: [ticket 1 — localize hardcoded messages](../06-contribution-practice/01-good-first-tickets.md).
Fill in after completing: which strings, how you found the seed-data path for resource keys, what the upgrade-path consideration was, review feedback.
Target senior-signal: you handled *both* new-install and upgrade seed paths — the detail 90% of first-time contributors miss.

## Story 4: Verifying before building (avoiding wasted work)
Prompts: "A time you changed course." / "Handling ambiguity."
Source: [ticket 8 or 10](../06-contribution-practice/01-good-first-tickets.md) — both are designed to potentially end in "no change needed."
Fill in after: what you assumed, what the code proved, what you shipped instead (possibly documentation).
Target signal: engineers who *close tickets without code* when evidence says so are trusted with bigger scopes.

## Story 5: Cross-layer feature with a rollback plan
Prompts: "Most complex thing you've built recently." / "How do you de-risk launches?"
Source: any [mid-level ticket](../06-contribution-practice/02-mid-level-feature-tickets.md), M1/M6 recommended.
Fill in after: the design note you wrote first, the failure path you tested deliberately, the rollback lever.
Target signal: "I designed the rollback before the feature" is a senior sentence at mid-level price.

## Story 6: Finding and fixing a performance issue with data
Prompts: "A performance problem you solved." 
Source: [ticket 14 / kata K7](../06-contribution-practice/04-refactor-and-design-katas.md) — the N+1 with measurement harness.
Fill in after: before/after query counts, and *especially* if the data said "don't ship" — the don't-ship version of this story is stronger than the ship version.

## Story 7: Improving safety systemically (mistake-proofing)
Prompts: "A time you improved your team's process." / "Preventing a class of bugs."
Source: [ticket 15](../06-contribution-practice/01-good-first-tickets.md) — the reflection test forcing `[CheckPermission]` on admin actions.
Fill in after: how many actions the test surfaced, exclusion-list debates, what a new engineer's experience becomes (add an action → CI explains the rule).
Target signal: converting review vigilance into automated invariants.

## Story 8: Receiving hard review feedback
Prompts: "A time you got critical feedback."
Source: whichever ticket's (real or simulated via [review sims](05-debugging-and-code-review-rounds.md)) review stung most.
Fill in after: the feedback verbatim, the part that was right, what changed in your next PR.
Target signal: specificity about *what you changed*, zero defensiveness in the retelling.

---

Mapping table (which story for which prompt):

| Prompt | First choice | Backup |
| --- | --- | --- |
| conflict/disagreement | 2 | 8 |
| ambiguity | 4 | 1 |
| mistake/feedback | 8 | 4 |
| technical tradeoff | 2 | 5 |
| leadership/initiative | 7 | 5 |
| complex problem | 5 | 1 |
| working with legacy | 1 | 3 |
