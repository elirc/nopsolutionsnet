# nopCommerce as a Training Lab

A learning curriculum built **from this repository** for one specific learner: a junior fullstack engineer (comfortable with CRUD web dev in .NET/C# or React/Node/TS) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **The transferable skill** — why the pattern exists, when it fails, and how to recognize and articulate it in any other repo or in an interview.

## What this repo is

nopCommerce is a full-featured open source e-commerce platform: ASP.NET Core MVC (`net10.0`), Razor views, a layered architecture of three class libraries — [Nop.Core](../src/Libraries/Nop.Core) (domain entities, caching, infrastructure), [Nop.Data](../src/Libraries/Nop.Data) (repositories, FluentMigrator migrations, Linq2Db data access), [Nop.Services](../src/Libraries/Nop.Services) (all business logic) — under two presentation projects: [Nop.Web](../src/Presentation/Nop.Web) (the storefront + `/Admin` area) and [Nop.Web.Framework](../src/Presentation/Nop.Web.Framework) (MVC plumbing: filters, DI startup, work context). It supports multi-store, multi-language, multi-vendor, ACL-per-entity, a plugin system (32 plugin projects in [src/Plugins](../src/Plugins)), scheduled background tasks, and a queued-email outbox. It has been in production at tens of thousands of stores for ~15 years, so it is a rare chance to study *mature* production tradeoffs — including some deliberate ugliness (see the named-mutex block in [OrderProcessingService.cs](../src/Libraries/Nop.Services/Orders/OrderProcessingService.cs#L1679-L1718)) that a tutorial codebase would never show you.

## The tracks

| Track | What it builds | Start at |
| --- | --- | --- |
| [01 Cartography](01-codebase-cartography/README.md) | A mental map: where everything lives, reading order, key flows | `01-system-map.md` |
| [02 Stack mastery](02-stack-and-language-mastery/README.md) | C#/.NET/ASP.NET Core mental models, anchored to real files | `01-language-runtime-model.md` |
| [03 Architecture](03-architecture-and-patterns/README.md) | Layers, persistence, auth, async, pattern catalog, critique | `05-pattern-catalog.md` |
| [04 Reading gym](04-code-reading-gym/README.md) | Active drills: annotate, trace, contrast, review | `01-annotation-drills.md` |
| [05 Quality](05-quality-engineering/README.md) | Testing, debugging, performance, security, observability | `01-testing-strategy.md` |
| [06 Contribution](06-contribution-practice/README.md) | Tickets and projects from junior to senior scope | `01-good-first-tickets.md` |
| [07 Career](07-career-and-collaboration/README.md) | Reviews, PRs, RFCs, maintainer communication | `01-code-review-mindset.md` |
| [08 Interview prep](08-interview-prep/README.md) | Question cards, system design, STAR stories, cram plan | `README.md` |
| [09 Reference](09-reference/verification-log.md) | Cheatsheet, risk register, rubrics, verification log | `command-cheatsheet.md` |

## Recommended paths

- **One weekend:** [00-fast-track.md](00-fast-track.md) only. Run it, trace two flows, make one safe change.
- **Two weeks:** fast track → cartography (all) → key flows → pattern catalog → 3 annotation drills → 2 junior tickets → skim interview README.
- **Eight weeks:** all tracks in order, one module/week, doing every drill and at least one mid-level ticket; weeks 7–8 = interview prep + a senior kata.
- **Ongoing contributor:** cartography → contribution practice → career track; use quality track per-PR.
- **Brand-new junior:** fast track → `01-cartography/02-file-reading-order.md` (junior path) → stack mastery → annotation drills. Skip architecture critique until month two.
- **Junior who knows .NET:** skip module 02 except the "sharp edges" sections; go straight to key flows and the pattern catalog.
- **Mid-level, new to this repo:** `01-system-map.md` → `05-key-flows.md` → `06-architecture-critique.md` → mid-level tickets.
- **Senior doing architecture review:** `03-architecture-and-patterns/06-architecture-critique.md` + `09-reference/risk-register.md`.
- **Interview in two weeks:** go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md). It pulls in everything else you need, day by day.

## Conventions used everywhere

- **Anchors.** Codebase claims cite `path#Lstart-Lend`, verified against `develop` @ `08720d3f05` (2026-07-16). Line numbers drift; if a link misses, search the symbol name. Drift log: [09-reference/verification-log.md](09-reference/verification-log.md).
- **Fake code is labeled.** Any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Unlabeled excerpts are real.
- **Verified vs inferred.** Commands marked __verified__ were run during authoring; __inferred__ come from CI config/README and should work but weren't executed here.
- **Drills are graded.** Each drill has Basic/Solid/Strong criteria. Grade yourself honestly; "Strong" is the mid-level interview bar.
- **Interview angle.** Sections flagged "Interview angle" name the question this material answers and what a mid-level vs senior answer sounds like.

## The mindset ladder

- A **junior** asks: *how do I make it work?*
- A **mid-level** engineer asks: *is this the right pattern here?*
- A **senior** asks: *what does this commit us to, who pays the cost, and how do we reduce the risk?*

Interviews for mid-level roles test exactly the second and third questions. Nobody fails a mid-level loop for forgetting syntax; people fail for having no opinion about tradeoffs and no concrete example to back one up. This repo gives you hundreds of concrete examples. Use them.
