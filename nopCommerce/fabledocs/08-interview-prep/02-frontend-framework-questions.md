# Frontend Question Cards

Ten cards. This repo's frontend is server-rendered Razor + progressive-enhancement JS — which is a *feature* for interviews: you can compare rendering models from experience instead of tribal loyalty. Where a card has a React/SPA twin, it's named — answer both ways and you cover .NET shops and JS shops with one prep.

## Q1: Server-side rendering vs SPA — when is each right?
Round: frontend. Testing: architectural judgment over framework tribalism.
Repo anchor: the whole storefront — [ProductController → factory → Razor view](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L198-L202); AJAX islands for interactivity ([add-to-cart JSON](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791)).
Junior: lists SEO for SSR, interactivity for SPA. Mid adds: this repo's model — SSR pages + JSON endpoints for the interactive islands — is *the* low-complexity default for content-heavy commerce; a SPA buys app-like UX at the cost of duplicated routing/state/auth handling. Senior includes: the convergence story (React Server Components/Next.js re-invented "SSR + islands"), and picks based on team shape and cache strategy, not fashion.
Follow-up: "How would you add a rich configurator to one product page?" (A mounted component island — not a rewrite.)
Drill: say the tradeoff in 60s, once for a .NET audience, once for a React audience.

## Q2: Where does view state live, and who owns shape transformation?
Round: frontend. Testing: state placement discipline.
Repo anchor: view models built exclusively in factories ([PrepareProductDetailsModelAsync#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502)); views are dumb templates.
Junior: "the model has the data." Mid: the factory layer = a selector layer: entities → display-ready shapes, centralized, cacheable, testable; the React twin is keeping components dumb and shaping in selectors/loaders. Senior: the invariant — *the DB shape must never become the UI contract* — plus where this repo pays for it (2,000-line factories).
Drill: map factory → your React app's equivalent (loader? selector? BFF?) in one sentence.

## Q3: This AJAX endpoint returns `{success, message}` or `{redirect}` depending on the branch. Critique.
Round: frontend. Testing: client-server contract hygiene.
Repo anchor: [ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810).
Junior: "seems fine, the JS handles it." Mid: it's an *implicit union type* with no schema — every consumer must handle every branch, and nothing tells them the branches exist; fix: consistent envelope or typed DTOs. Senior: contract-first framing — the response shape is an API even when "it's just our own jQuery"; ties to [ticket 7](../06-contribution-practice/01-good-first-tickets.md).
Follow-up: "How do you version such an endpoint?"

## Q4: Progressive enhancement — what breaks with JS off?
Round: frontend. Testing: resilience thinking, accessibility adjacency.
Repo anchor: storefront forms are real `<form>` POSTs enhanced by JS (add-to-cart posts to a real action).
Junior: "modern sites need JS." Mid: forms-that-work-then-enhance = resilience for flaky networks, crawlers, assistive tech; the JSON endpoints are an *enhancement layer*, not the only path. Senior: cost-benefit by audience; where the repo genuinely requires JS (admin DataTables grids) and why that's an acceptable line (authenticated operators, not the public).

## Q5: How do themes/overridable views work, and what's the maintenance cost?
Round: frontend. Testing: extensibility tradeoffs in the view layer.
Repo anchor: [Themes folder](../../src/Presentation/Nop.Web/Themes), view resolution via the theme context in Nop.Web.Framework.
Junior: "you copy the view and edit it." Mid: theme = view-override layer resolved at render time; the cost mirrors [pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md) — every overridden view freezes against upstream changes; upgrades break themes silently. Senior: the CSS-variables/design-tokens alternative (override style, not markup) and when full view overrides are truly needed.

## Q6: Validate on client, server, or both?
Round: frontend. Testing: the two-layer validation rule.
Repo anchor: FluentValidation rules ([LoginValidator](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9-L34)) with client-side adapters ([ClientsideFluentValidation](../../src/Presentation/Nop.Web.Framework/ClientsideFluentValidation)) — one rule source, two enforcement points.
Junior: "both." Mid: both, but *derived from one definition* — this repo generates client rules from the server validators, killing drift; business rules still live deeper ([validation stack](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)). Senior: what can't be client-validated (uniqueness, stock) and designing for the round-trip UX of those.

## Q7: The product page shows a stale price after an admin edit. Frontend or backend bug?
Round: frontend (debugging flavor). Testing: full-stack bisection.
Repo anchor: [debugging scenario 1](../05-quality-engineering/03-systematic-debugging.md) — plus the *frontend* layer: browser cache, CDN, output caching.
Junior: blames the server cache. Mid: bisects — hard-refresh (browser cache?) → curl (CDN/proxy?) → then server-side cache spine. Senior: cache *layering* as a system — every layer needs an invalidation story, and "how many caches sit between the DB and the pixel?" is the first question.

## Q8: Accessibility — what would you audit first on this storefront?
Round: frontend. Testing: a11y pragmatism.
Repo anchor: storefront Razor views/themes (conceptual — no single anchor).
Junior: "add alt text." Mid: audit by user journey (browse → cart → checkout): form labels and error announcements (the warnings-list messages must reach screen readers — `aria-live` on cart responses), keyboard paths through attribute selectors, contrast in themes. Senior: a11y as acceptance criteria + automated checks in CI, and the legal/market framing that gets it funded.

## Q9: Why bundle with gulp in 2026, and when would you change it?
Round: frontend. Testing: build-tooling judgment vs novelty-chasing.
Repo anchor: [package.json + gulpfile](../../src/Presentation/Nop.Web/package.json); bundles committed, CI doesn't run node ([dotnet.yml](../../.github/workflows/dotnet.yml)).
Junior: "gulp is old, use Vite." Mid: the pipeline copies/bundles a stable set of libraries — churn cost of migrating exceeds benefit *while the frontend stays jQuery-shaped*; the trigger for change is adopting a component framework, not the calendar. Senior: notes the committed-artifacts tradeoff (reproducibility vs repo noise vs CI-node cost) as the actual decision.

## Q10: Add "items in cart" badge that's always correct. Design it.
Round: frontend (mini-design). Testing: state synchronization end-to-end.
Repo anchor: cart data via [GetShoppingCartAsync](../../src/Libraries/Nop.Services/Orders/ShoppingCartService.cs#L753); mutations return JSON ([AddProductToCart_Details](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L791)).
Junior: "update the DOM after add-to-cart." Mid: enumerate the desync sources — other tabs, back button, merged carts on login, server-side changes — and pick: server-rendered count on navigation + update-from-response on mutation (this repo's shape), vs polling, vs push. Senior: correctness budget ("eventually right on next navigation" is fine for a badge — say *why*: cost of push vs harm of staleness) — budget-matching is the senior tell all through this curriculum.
