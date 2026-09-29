# Learning Rubrics

Observable behaviors, not vibes. Grade yourself per skill; "Interview-ready" is the bar for a mid-level loop. Re-assess at the [cram plan](../08-interview-prep/07-two-week-cram-plan.md) day-7 and day-14 checkpoints.

## Codebase navigation

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Finds a named class with search | Predicts file locations from the architecture before searching | Reads a new subsystem by its boundaries first; knows what *not* to read | you can locate the code behind any storefront behavior in <5 min and narrate how you found it |

## Flow tracing

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Follows the happy path of one layer | Traces UI→DB across layers incl. side effects | Also traces failure paths, concurrency windows, cache interactions | you can narrate add-to-cart and checkout aloud, with failure modes, in ≤3 min each ([key flows](../01-codebase-cartography/05-key-flows.md)) |

## Patterns & architecture

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Names patterns after being told | Recognizes patterns unprompted and states what problem each solves *here* | Argues when each pattern is wrong; proposes migrations with sequencing | for 10 random [catalog cards](../03-architecture-and-patterns/05-pattern-catalog.md), you give problem + tradeoff + failure mode from memory |

## Data & persistence

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Writes CRUD against the repository | Designs additive migrations; knows snapshot-vs-reference and soft-delete reasoning | Designs zero-downtime changes; identifies race windows and constraint-level fixes | you can run the expand/contract recipe and the [K6 kata](../06-contribution-practice/04-refactor-and-design-katas.md) cold |

## Security & authorization

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Adds the attribute someone tells them to | Distinguishes authn/authz/validation/isolation; checks object-level access | Audits fail-open defaults and trust decisions; automates guards | you can answer "how do you prevent IDOR" with this repo's three-mechanism stack ([validation-auth map](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)) |

## Testing

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Writes a happy-path test that passes | Tests failure paths, permissions, isolation; restores mutated state | Chooses test strategy per codebase; improves the harness itself | you can defend sociable-vs-mocked testing with this suite as evidence ([testing strategy](../05-quality-engineering/01-testing-strategy.md)) |

## Debugging

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Adds print statements until something appears | Bisects by layer, cheapest probe first, one hypothesis at a time | Diagnoses from artifacts (timestamps, guids, row states); prevents the *class* | all four [debugging sims](../08-interview-prep/05-debugging-and-code-review-rounds.md) pass their bars, narrated aloud |

## Code review

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Comments on style | Finds correctness issues; grades severity honestly | Reviews designs, not diffs; separates direction from mechanics; stays kind under disagreement | [review katas](../04-code-reading-gym/04-review-katas.md) at Solid+, incl. one kind rejection with a better path |

## Communication & collaboration

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Asks broad questions; PRs say "fixes bug" | PRs carry what/why/tested/risks; questions carry evidence | Writes RFCs; disagreements produce decisions, not friction | your last 3 (real or practice) PR descriptions pass the [five-section template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) |

## Interview execution (the meta-skill)

| Junior | Mid | Senior | Interview-ready when… |
| --- | --- | --- | --- |
| Answers with definitions | Every answer: concrete example + tradeoff + failure mode | Also: names constraints, changes position on new evidence gracefully | 90-second card answers hit the golden rule ≥80% of the time; both STAR bank stories land under 2 min |

## Self-assessment checklist (tick honestly)

- [ ] I can draw the system map from memory
- [ ] I can narrate 3 key flows aloud without notes
- [ ] I can name 8+ patterns with tradeoffs and anchors
- [ ] I have written and run at least 3 tests in this repo
- [ ] I have completed 2+ junior tickets and 1 mid ticket (or honest equivalents)
- [ ] I have run every debugging/review sim at least once, timed
- [ ] My STAR bank has ≥4 stories with real details
- [ ] I have done the full system-design walkthrough aloud twice
