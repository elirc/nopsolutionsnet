# Runtime and Tooling Map

## Build & run surface

| Tool | Role | Evidence |
| --- | --- | --- |
| .NET SDK 10.0.100 (`rollForward: latestFeature`) | compile + run everything | [global.json](../../global.json) |
| `dotnet restore/build/test src` | the entire CI pipeline (windows-latest) | [.github/workflows/dotnet.yml](../../.github/workflows/dotnet.yml) |
| NUnit (+ the SQLite test host) | test framework | [Nop.Tests.csproj](../../src/Tests/Nop.Tests/Nop.Tests.csproj) |
| gulp 4 + npm | bundles storefront/admin CSS/JS from `node_modules` into `wwwroot` | [package.json](../../src/Presentation/Nop.Web/package.json), [gulpfile.js](../../src/Presentation/Nop.Web/gulpfile.js) |
| Docker | app image + per-database compose files | [Dockerfile](../../Dockerfile), [docker-compose.yml](../../docker-compose.yml) |
| deploy scripts | classic web deploy | [src/deploy.cmd](../../src), `src/Build` |

No Makefile/justfile; `dotnet` + `npx gulp` are the only entry points. There is **no linter/formatter config in CI** — style is enforced by review convention (match surrounding code).

## Runtime boundaries

Everything is **server-side .NET in one process**: MVC controllers, Razor rendering, background scheduler. The browser runs jQuery-era JavaScript from `wwwroot` (no SPA framework, no Node runtime in production). That means:

- "Frontend work" here = Razor views + view models + progressive-enhancement JS calling JSON endpoints (e.g. the add-to-cart AJAX shapes in [ShoppingCartController.cs#L796-L810](../../src/Presentation/Nop.Web/Controllers/ShoppingCartController.cs#L796-L810)).
- There is no browser/server code-sharing boundary to police (unlike Next.js) — the boundary that *does* need policing is **entity vs view model** (entities must not reach views; factories enforce this).
- Background work shares the web process ([ScheduleTaskRunner](../../src/Libraries/Nop.Services/ScheduleTasks/ScheduleTaskRunner.cs#L115)) — a runaway task competes with requests for CPU/pool threads.

## Configuration

Config is unusual and worth knowing cold: [Program.cs](../../src/Presentation/Nop.Web/Program.cs) loads `App_Data/appsettings.json` (created by the **installer UI on first run**, holds the connection string), then `App_Data/appsettings.{Environment}.json`, then environment variables. Most *behavioral* config isn't in files at all — it's [settings entities](../03-architecture-and-patterns/05-pattern-catalog.md) in the database, editable in admin at runtime. High-level env concerns: connection string, cache provider selection (memory vs Redis), `UseAutofac` toggle. Never commit `App_Data/appsettings.json` — it's install-specific and holds credentials.

## Database

Three supported providers (SQL Server, MySQL, PostgreSQL) behind [INopDataProvider](../../src/Libraries/Nop.Data/INopDataProvider.cs), with Linq2Db as the query layer — *not* Entity Framework. Schema comes from code: [entity builders](../../src/Libraries/Nop.Data/Mapping/Builders/Catalog/ProductBuilder.cs#L9) define tables, [FluentMigrator migrations](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7) evolve them. Tests swap in SQLite ([SqLiteNopDataProvider](../../src/Tests/Nop.Tests/SqLiteNopDataProvider.cs)) — the provider abstraction is what makes that possible; remember this when someone asks "why abstract the database?"

## Dev loop (commands)

```bash
# __inferred__ (matches CI; verify locally)
dotnet build src                                   # full solution build
dotnet run --project src/Presentation/Nop.Web      # run storefront (first run = installer)
dotnet test src                                    # full test suite (SQLite, no external DB)
dotnet test src --filter "FullyQualifiedName~ShoppingCart"   # targeted
# frontend assets, only when touching css/js sources:
cd src/Presentation/Nop.Web && npm install && npx gulp       # __inferred__ from gulpfile
```

Transferable skill: locating a repo's *real* commands means reading CI first ([dotnet.yml](../../.github/workflows/dotnet.yml) is 30 lines) — CI never lies, READMEs sometimes do.
