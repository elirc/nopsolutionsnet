# Two-Week Cram Plan

Assumes ~2.5h/day weekdays, ~5h weekend days, interview at the end of week 2. Everything references files in this curriculum. **Speak-aloud time is non-negotiable** — reading feels like progress; talking is the skill being tested.

## Week 1 — build the evidence base

**Day 1** — Run the [fast track morning](../00-fast-track.md) (get it running + flow 1). Evening: [system map](../01-codebase-cartography/01-system-map.md). *Aloud (10 min):* describe the repo's architecture to a wall.
**Day 2** — [Key flows](../01-codebase-cartography/05-key-flows.md) 1–3 with files open. *Aloud:* narrate add-to-cart end-to-end, 3-minute cap.
**Day 3** — Key flows 4–6. Cards: [C#/.NET deep-dive](01-csharp-dotnet-deep-dive.md) Q1–Q6, 90s each aloud. Log gaps.
**Day 4** — [Pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) patterns 1–8. Cards Q7–Q12.
**Day 5** — Patterns 9–16. Cards Q13–Q16 + re-run any Day-3/4 card you fumbled. *Aloud:* the golden-rule cache answer ([module README](README.md)) until fluent.
**Day 6 (weekend)** — Morning: full [system-design walkthrough](04-system-design-from-this-repo.md) on a whiteboard/paper, 40 min timed, no notes; then review against the doc. Afternoon: [annotation drills](../04-code-reading-gym/01-annotation-drills.md) 1, 4, 5. Write [STAR stories 1–2](06-behavioral-star-stories.md) drafts.
**Day 7 — checkpoint (weekend)** — Self-assess against [learning-rubrics](../09-reference/learning-rubrics.md): can you (a) narrate 3 flows without notes, (b) answer 12/16 language cards at Solid, (c) run the design walkthrough inside 45 min? Whatever failed is Week 2's priority. Rest after the checkpoint — genuinely.

## Week 2 — pressure-test and polish

**Day 8** — [Debugging sims 1+3](05-debugging-and-code-review-rounds.md), timed, aloud. Review [systematic-debugging](../05-quality-engineering/03-systematic-debugging.md) after (not before).
**Day 9** — [Review sims 1+2](05-debugging-and-code-review-rounds.md). Read [code-review mindset](../07-career-and-collaboration/01-code-review-mindset.md) after. Cards: [frontend](02-frontend-framework-questions.md) Q1–Q5.
**Day 10** — [API/data cards](03-api-and-data-modeling-questions.md) Q1–Q6 aloud. Debugging sim 2 (the double order — the hardest one). Rehearse STAR stories 1–2 to a human if at all possible.
**Day 11** — API/data Q7–Q12. Frontend Q6–Q10. Review sim 3. Fill any remaining STAR worksheets from work you've actually done (don't invent — thin true stories beat rich fabricated ones).
**Day 12** — System-design **variation prompts** (pick 3 of the 5 in [the walkthrough](04-system-design-from-this-repo.md)), 15 min each aloud. Debugging sim 4.
**Day 13 (weekend)** — Full mock loop, self-run with timers: 40 min design + 20 min debugging sim (rerun #1 — it should feel easy now; that's the point) + 20 min review sim + 30 min behavioral (all 8 prompts from the [mapping table](06-behavioral-star-stories.md)). Afternoon: review only what broke.
**Day 14 — checkpoint + taper** — Re-run the Day-7 checks plus: 5 random cards from each file at Solid+, one STAR story cold from a random prompt. Then stop by early evening. No new material on day 14 — tapering is real for cognitive performance too.

## The night before / day of

Skim only: your STAR mapping table, the golden-rule answer, and the checkout deep-dive section of the design walkthrough (your strongest 8 minutes — steer toward it when the interviewer offers a choice). In the interview, when stuck: narrate your bisection ("I'd check whether the value is already wrong at the service boundary…") — process talk *is* the answer in practical rounds.

## If you have less than two weeks

Priority order: design walkthrough (Day 6) → debugging sims 1+2 → language cards Q1–Q8 → STAR stories 1–2 → everything else. The single highest-leverage artifact in this curriculum for interviews is the [system-design walkthrough](04-system-design-from-this-repo.md) — a candidate who can ground "design checkout" in real production code is memorable.
