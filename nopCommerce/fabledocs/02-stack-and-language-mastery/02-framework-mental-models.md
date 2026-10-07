# ASP.NET Core / MVC Mental Models

Four framework models — pipeline, filters, DI, server rendering — each tied to the code that implements it here, followed by a full request trace, an exercise ladder, verified findings, and a debugging scenario.

## The pipeline: a stack of middleware, order is law

Model: every request descends a configured list of middleware and ascends back. Anything registered earlier sees the request first and the response last. Auth before MVC, exception handling outermost, static files short-circuit early.

Here: [Program.cs](../../src/Presentation/Nop.Web/Program.cs) calls `app.ConfigureRequestPipeline()`, which is a one-line forwarder ([ApplicationBuilderExtensions.cs#L52-L55](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs#L52-L55)) to the engine; the engine finds every `INopStartup`, sorts by `Order`, and calls each one's `Configure` ([NopEngine.cs#L132-L144](../../src/Libraries/Nop.Core/Infrastructure/NopEngine.cs#L132-L144)). So **the pipeline order is not written in any single file** — it is the sort order of `Order` properties spread across classes. Reconstructed from source (each line is a `Use…` call inside that startup's `Configure`):

| Order | Startup | Middleware it adds (file:line) |
| --- | --- | --- |
| 0 | `ErrorHandlerStartup` | `UseNopExceptionHandler`, `UseBadRequestResult`, `UsePageNotFound` ([ErrorHandlerStartup.cs#L30-L36](../../src/Presentation/Nop.Web.Framework/Infrastructure/ErrorHandlerStartup.cs#L30-L36)) |
| 99 | `NopStaticFilesStartup` | response compression, web optimizer, static files ([NopStaticFilesStartup.cs#L35-L41](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStaticFilesStartup.cs#L35-L41)) |
| 100 | `NopCommonStartup` | keep-alive, install URL, session, request localization, PDF, themes ([NopCommonStartup.cs#L55-L70](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopCommonStartup.cs#L55-L70)) |
| 400 | `NopRoutingStartup` | `UseRouting`, and `UseRateLimiter` only when `PermitLimit > 0` ([NopRoutingStartup.cs#L30-L34](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopRoutingStartup.cs#L30-L34)) |
| 500 | `AuthenticationStartup` | `UseNopAuthentication` ([AuthenticationStartup.cs#L35](../../src/Presentation/Nop.Web.Framework/Infrastructure/AuthenticationStartup.cs#L35)) |
| 600 | `AuthorizationStartup` | `UseAuthorization` ([AuthorizationStartup.cs#L29](../../src/Presentation/Nop.Web.Framework/Infrastructure/AuthorizationStartup.cs#L29)) |
| 900 | `NopEndpoints` | `UseNopEndpoints` ([NopEndpoints.cs#L30](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopEndpoints.cs#L30)) |

(The proxy, markup-minification, MVC, data and core-service startups also participate; the full `Order` table is in [01-codebase-cartography/01-system-map.md](../01-codebase-cartography/01-system-map.md).) Sharp edge: to answer "what runs before my controller?", you must collect every startup class — including plugins' — and sort by `Order`. A plugin with `Order => 450` would slot its middleware between routing and authentication.

Interview angle: "Middleware vs filters?" Middleware = HTTP-level, app-wide, knows nothing of MVC. Filters = MVC-level, know the action, can be per-action attributes. This repo shows the filter half beautifully (next section).

## Filters: cross-cutting per-action behavior

Model: MVC runs filter stages (authorization → resource → action → result) around each action. Attributes bind filters declaratively.

Here: the [Mvc/Filters folder](../../src/Presentation/Nop.Web.Framework/Mvc/Filters) is a 26-file catalog (counted with `ls | wc -l`) — authorization ([CheckPermissionAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16)), security posture ([HttpsRequirementAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/HttpsRequirementAttribute.cs)), bot honeypots ([ValidateHoneypotAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/ValidateHoneypotAttribute.cs)), CAPTCHA ([ValidateCaptchaAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/ValidateCaptchaAttribute.cs) on the [login action, CustomerController.cs#L458](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458)), and behavioral side effects ([SaveLastActivityAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/SaveLastActivityAttribute.cs)). Reading that folder once teaches you more real-world MVC than any tutorial.

Where the attributes are *applied* matters as much as what they do. Base classes carry most of them:

- **Every controller:** `[AutoValidation]` on [BaseController.cs#L32](../../src/Presentation/Nop.Web.Framework/Controllers/BaseController.cs#L32) — this is how FluentValidation runs (see below).
- **Storefront controllers:** [BasePublicController.cs#L7-L12](../../src/Presentation/Nop.Web/Controllers/BasePublicController.cs#L7-L12) adds `WwwRequirement`, `CheckLanguageSeoCode`, `CheckAccessPublicStore`, `CheckAccessClosedStore`, `CheckDiscountCoupon`, `CheckAffiliate`.
- **Admin controllers:** [BaseAdminController.cs#L12-L18](../../src/Presentation/Nop.Web/Areas/Admin/Controllers/BaseAdminController.cs#L12-L18) adds `Area`, `AutoValidateAntiforgeryToken`, `ValidateIpAddress`, `AuthorizeAdmin`, `ValidateVendor`, `SaveSelectedTab`, `NotNullValidationMessage`.

Many filters support **per-action opt-out**: the filter looks for an action-scoped instance of itself with `ignore: true` and skips (see the pattern in [AuthorizeAdminAttribute.cs#L75-L83](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AuthorizeAdminAttribute.cs#L75-L83) and [AutoValidationAttribute.cs#L68-L78](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L68-L78)). The login POST uses this twice: `[CheckAccessClosedStore(ignore: true)]` and `[CheckAccessPublicStore(ignore: true)]` ([CustomerController.cs#L460-L462](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L460-L462)) so you can log in to a closed store.

A deliberate exception that proves the rule: the public [ScheduleTaskController](../../src/Presentation/Nop.Web/Controllers/ScheduleTaskController.cs#L6-L9) inherits plain `Controller`, *not* `BasePublicController`, with the comment that the base filters "can create guest account(s), etc". Choosing a base class is choosing a filter stack.

Sharp edge: filters are invisible at the call site of the action body — a junior reading `Login(...)` sees no CAPTCHA code and may re-add it. Always read the attributes, **including the base class's**, *as part of* the method signature.

## Validation is a filter, not model binding

FluentValidation is wired in two halves: validators are registered from application-part assemblies ([ServiceCollectionExtensions.cs#L344-L349](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L344-L349)), and the `AutoValidation` action filter resolves `IValidator<T>` for each class-typed action argument and writes failures into `ModelState` ([AutoValidationAttribute.cs#L96-L123](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L96-L123)). Then the action checks `ModelState.IsValid` itself (e.g. [CustomerController.cs#L469](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L469)). Two consequences: validation runs *after* model binding and *before* your action body, and if no validator is found for a type, the filter silently `continue`s ([L106-L107](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L106-L107)) — absence of a validator is not an error.

## DI: constructor injection everywhere, scoped by default

Model: the container is the application's object graph, composed at startup. Constructor parameters are the honest list of what a class needs.

Here: the core `NopStartup.ConfigureServices` makes **158** `services.Add*` registrations (counted: 143 `AddScoped`, 13 `AddSingleton`, 2 `AddTransient`) — sample: [NopStartup.cs#L195-L215](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215). Repositories are an open generic, `IRepository<>` → `EntityRepository<>` ([NopDbStartup.cs#L63](../../src/Libraries/Nop.Data/NopDbStartup.cs#L63)); event consumers are discovered by reflection and registered per closed `IConsumer<T>` interface ([NopStartup.cs#L307-L314](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L307-L314)); controllers are registered as services so they can be overridden ([ServiceCollectionExtensions.cs#L351-L352](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L351-L352)). Controllers and services declare dependencies via constructors (see the 4-dependency ctor of [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L23-L32) vs the dozens in `ProductModelFactory` — ctor length is a cohesion smell you can *read*).

Lifetimes worth memorizing because they bite: `IEventPublisher` is a **singleton** ([NopStartup.cs#L237](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L237)) that resolves **scoped** consumers on every publish through the static engine ([EventPublisher.cs#L23](../../src/Libraries/Nop.Services/Events/EventPublisher.cs#L23)); `ITaskScheduler` is a singleton ([NopStartup.cs#L303](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L303)); `IStaticCacheManager` is a singleton or scoped depending on cache mode ([NopStartup.cs#L98-L130](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L98-L130)).

Sharp edges: `ValidateScopes=false` in [Program.cs](../../src/Presentation/Nop.Web/Program.cs#L36) removes the captive-dependency safety net (the comment explains: startup code resolves scoped services from the root); `EngineContext.Current.Resolve<T>()` service-location appears in infrastructure corners ([CacheEventConsumer ctor](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L28-L32)) where constructor injection isn't possible.

Interview angle → [Q3](../08-interview-prep/01-csharp-dotnet-deep-dive.md). The phrase that lands: "constructor injection makes dependencies *reviewable*; service location hides them — this repo uses the latter only where object lifetime rules out the former."

## Razor + model factories: server rendering with a strict model boundary

Model: Razor views are templates over *view models* — never entities. Someone must build those models; in most MVC apps it happens ad hoc in controllers; here it's an entire layer (23 factory files in [Nop.Web/Factories](../../src/Presentation/Nop.Web/Factories), counted with `ls`).

Here: `Prepare*ModelAsync` factories ([ProductModelFactory#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502)) own entity→model assembly; controllers stay short; views select templates dynamically ([ProductController.cs#L200](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L200)). Validation of *incoming* models is FluentValidation ([LoginValidator](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9)) run by the `AutoValidation` filter described above.

Sharp edge: the AJAX endpoints return anonymous JSON objects ([ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)) — contracts that exist only in the JS that parses them. When you touch one, grep `wwwroot/js` for the consumer.

Interview angle: "MVC vs SPA?" — honest answer from this repo: server rendering with progressive enhancement is operationally simpler (one deploy, SEO for free) and the factory layer is its state-management discipline; a SPA moves that discipline into the client. React comparison: a factory is `getServerSideProps` + selectors, done in C#.

## Feature trace: `POST /login`, framework layers only

| # | Layer | What runs | Evidence |
| --- | --- | --- | --- |
| 1 | middleware (0) | exception handler wraps everything | [ErrorHandlerStartup.cs#L30](../../src/Presentation/Nop.Web.Framework/Infrastructure/ErrorHandlerStartup.cs#L30) |
| 2 | middleware (100) | session + request localization | [NopCommonStartup.cs#L61-L64](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopCommonStartup.cs#L61-L64) |
| 3 | middleware (400–600) | routing → authentication → authorization | table above |
| 4 | filters (class) | `BasePublicController` stack; closed-store and public-store checks are *ignored* for this action | [CustomerController.cs#L460-L462](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L460-L462) |
| 5 | filters (action) | `ValidateCaptcha` sets the `captchaValid` argument | [CustomerController.cs#L458](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458) |
| 6 | filter (action) | `AutoValidation` resolves `IValidator<LoginModel>` → `LoginValidator` rules | [LoginValidator.cs#L11-L38](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L11-L38) |
| 7 | action | `ModelState.IsValid` gate, then the service call | [CustomerController.cs#L469-L486](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L469-L486) |

## Exercise ladder

1. **Goal:** list the filters that run on a storefront action you didn't write.
   **Check:** pick `ProductController.ProductDetails`; open its base class chain and write every attribute you find. Must include the six on `BasePublicController` and `AutoValidation` from `BaseController`.
2. **Goal:** prove the pipeline order without running the app.
   **Check:** `grep -rn "public int Order =>" src/Presentation/Nop.Web.Framework/Infrastructure src/Libraries/Nop.Data/NopDbStartup.cs src/Presentation/Nop.Web/Infrastructure` and sort the results numerically; your sorted list must put `ErrorHandlerStartup` (0) before `NopRoutingStartup` (400) before `NopEndpoints` (900).
3. **Goal:** count admin controllers that rely only on the area gate.
   **Check:** `grep -L "CheckPermission" src/Presentation/Nop.Web/Areas/Admin/Controllers/*.cs` lists 8 of 63 files on 2026-10-06 (including `BaseAdminController` itself). For two of them, decide whether "any admin-area user may do this" is the intended rule.
4. **Goal (mid):** explain why a missing validator is silent.
   **Check:** cite [AutoValidationAttribute.cs#L106-L107](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L106-L107) and [ServiceCollectionExtensions.cs#L345-L349](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L345-L349); then answer F1 below before reading it.
5. **Goal (mid):** identify a lifetime mismatch the container won't warn you about.
   **Check:** name the singleton that consumes scoped services on every call (`EventPublisher`) and explain why it is safe: it resolves through `EngineContext.Current` at publish time inside the current scope, instead of capturing consumers in its constructor.

## Senior findings (verified in source)

**F1 — Validator registration is name-filtered, and missing validators fail open.** Validators are registered only from application parts whose assembly name **starts with "Nop"** ([ServiceCollectionExtensions.cs#L347](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L347)). The `AutoValidation` filter treats "no validator registered" as "nothing to validate" ([AutoValidationAttribute.cs#L106-L107](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L106-L107)). Put together: a third-party plugin whose assembly is named, say, `Acme.Shipping` can ship `AbstractValidator<T>` classes that are never registered, and its models will pass validation silently. All 32 in-repo plugins are named `Nop.Plugin.*`, so they are unaffected — this bites external authors. Confidence: high on mechanism; a cheap mitigation is a startup log line listing validator types found per plugin assembly.

**F2 — The global rate limiter puts all anonymous users of a host in one bucket.** When enabled (`PermitLimit > 0`, default 0 — [CommonConfig.cs#L60](../../src/Libraries/Nop.Core/Configuration/CommonConfig.cs#L60)), the fixed-window limiter partitions by `httpContext.User.Identity?.Name ?? Request.Headers.Host` ([ServiceCollectionExtensions.cs#L123-L132](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ServiceCollectionExtensions.cs#L123-L132)). Every request without an identity name — guests, bots, health checks — shares the *host's* partition, so one aggressive client can exhaust the per-minute permit for every anonymous shopper on that storefront, while the partition key is also attacker-influenced (the `Host` header). Turning it on as a DDoS control can therefore produce a self-inflicted outage. Confidence: high on mechanism (it's one expression); impact depends on whether operators enable it.

## Debugging scenario: "my new action ignores its validator"

*Symptom:* you added `[HttpPost] Save(MyModel model)` to a plugin controller and `MyModelValidator : BaseNopValidator<MyModel>`, but invalid input reaches the action with `ModelState.IsValid == true`.

1. **Does the controller inherit `BaseController`?** If it inherits plain `Controller`, `[AutoValidation]` ([BaseController.cs#L32](../../src/Presentation/Nop.Web.Framework/Controllers/BaseController.cs#L32)) is not applied at all.
2. **Is the action opting out?** Look for `[AutoValidation(ignore: true)]` on the action — the filter honours action-scoped overrides ([AutoValidationAttribute.cs#L68-L78](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L68-L78)).
3. **Is the validator registered?** Check the plugin assembly name against the `StartsWith("Nop")` filter (F1).
4. **Is the argument a class instance?** The filter skips null arguments and non-class types ([L98-L103](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/AutoValidationAttribute.cs#L98-L103)); a failed bind that yields `null` is never validated.
5. **Does the action read `ModelState`?** The filter only *records* errors; it does not short-circuit (L140-L143 always call `next()` unless a result was set). An action that never checks `ModelState.IsValid` will proceed regardless.

## Verification

2026-10-06: every anchor opened and checked. Registration count corrected from "~200" to 158 (`grep -cE "services\.Add(Scoped|Singleton|Transient)"`); the pipeline table was built by reading each startup's `Configure` body; the previous claim that the "concrete order lives in ApplicationBuilderExtensions.cs" was corrected (that file only forwards to the engine). Filter-folder (26), factory-folder (23) and admin-controller (63 files, 8 without `CheckPermission`) counts were taken with `ls`/`grep -L`.
