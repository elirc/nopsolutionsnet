# ASP.NET Core / MVC Mental Models

## The pipeline: a stack of middleware, order is law

Model: every request descends a configured list of middleware and ascends back. Anything registered earlier sees the request first and the response last. Auth before MVC, exception handling outermost, static files short-circuit early.

Here: [Program.cs](../../src/Presentation/Nop.Web/Program.cs) calls `app.ConfigureRequestPipeline()`, which fans out to `INopStartup.Configure` implementations ordered by their `Order` property (core module = 2000, [NopStartup.cs#L336](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L336)); the concrete order lives in [ApplicationBuilderExtensions.cs](../../src/Presentation/Nop.Web.Framework/Infrastructure/Extensions/ApplicationBuilderExtensions.cs). Sharp edge: pipeline assembly is spread across discoverable modules — to answer "what runs before my controller?", you must collect every startup class and sort by `Order`. Plugins insert themselves the same way.

Interview angle: "Middleware vs filters?" Middleware = HTTP-level, app-wide, knows nothing of MVC. Filters = MVC-level, know the action, can be per-action attributes. This repo shows the filter half beautifully (next section).

## Filters: cross-cutting per-action behavior

Model: MVC runs filter stages (authorization → resource → action → result) around each action. Attributes bind filters declaratively.

Here: the [Mvc/Filters folder](../../src/Presentation/Nop.Web.Framework/Mvc/Filters) is a 26-filter catalog — authorization ([CheckPermissionAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/CheckPermissionAttribute.cs#L16)), security posture ([HttpsRequirementAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/HttpsRequirementAttribute.cs)), bot honeypots ([ValidateHoneypotAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/ValidateHoneypotAttribute.cs)), CAPTCHA ([ValidateCaptchaAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/ValidateCaptchaAttribute.cs) on the [login action, CustomerController.cs#L458](../../src/Presentation/Nop.Web/Controllers/CustomerController.cs#L458)), and behavioral side effects ([SaveLastActivityAttribute](../../src/Presentation/Nop.Web.Framework/Mvc/Filters/SaveLastActivityAttribute.cs)). Reading that folder once teaches you more real-world MVC than any tutorial.

Sharp edge: filters are invisible at the call site of the action body — a junior reading `Login(...)` sees no CAPTCHA code and may re-add it. Always read the attributes *as part of* the method signature.

## DI: constructor injection everywhere, scoped by default

Model: the container is the application's object graph, composed at startup. Constructor parameters are the honest list of what a class needs.

Here: ~200 registrations in [NopStartup.ConfigureServices](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L195-L215), nearly all scoped; controllers and services declare dependencies via constructors (see the 4-dependency ctor of [QueuedMessagesSendTask](../../src/Libraries/Nop.Services/Messages/QueuedMessagesSendTask.cs#L23-L32) vs the dozens in `ProductModelFactory` — ctor length is a cohesion smell you can *read*). Sharp edges: `ValidateScopes=false` in [Program.cs](../../src/Presentation/Nop.Web/Program.cs) removes the captive-dependency safety net; `EngineContext.Current.Resolve<T>()` service-location appears in infrastructure corners ([CacheEventConsumer ctor](../../src/Libraries/Nop.Services/Caching/CacheEventConsumer.cs#L28-L32)) where constructor injection isn't possible.

Interview angle → [Q3](../08-interview-prep/01-csharp-dotnet-deep-dive.md). The phrase that lands: "constructor injection makes dependencies *reviewable*; service location hides them — this repo uses the latter only where object lifetime rules out the former."

## Razor + model factories: server rendering with a strict model boundary

Model: Razor views are templates over *view models* — never entities. Someone must build those models; in most MVC apps it happens ad hoc in controllers; here it's an entire layer.

Here: `Prepare*ModelAsync` factories ([ProductModelFactory#L1502](../../src/Presentation/Nop.Web/Factories/ProductModelFactory.cs#L1502)) own entity→model assembly; controllers stay ~30 lines; views select templates dynamically ([ProductController.cs#L200](../../src/Presentation/Nop.Web/Controllers/ProductController.cs#L200)). Validation of *incoming* models is FluentValidation ([LoginValidator](../../src/Presentation/Nop.Web/Validators/Customer/LoginValidator.cs#L9)) wired into model binding.

Sharp edge: the AJAX endpoints return anonymous JSON objects ([ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)) — contracts that exist only in the JS that parses them. When you touch one, grep `wwwroot/js` for the consumer.

Interview angle: "MVC vs SPA?" — honest answer from this repo: server rendering with progressive enhancement is operationally simpler (one deploy, SEO for free) and the factory layer is its state-management discipline; a SPA moves that discipline into the client. React comparison: a factory is `getServerSideProps` + selectors, done in C#.
