# Command Cheatsheet

All marked __verified__ (run during authoring) or __inferred__ (from CI/config; expected to work, not executed here). CI truth source: [.github/workflows/dotnet.yml](../../.github/workflows/dotnet.yml).

## Build / run

| Command | Purpose | Status |
| --- | --- | --- |
| `dotnet restore src` | restore packages (exact CI step) | __inferred__ |
| `dotnet build src --configuration Release` | build everything (CI uses `--no-restore`) | __inferred__ |
| `dotnet run --project src/Presentation/Nop.Web` | run storefront; first launch = installer UI | __inferred__ |
| `docker compose up` (root compose files) | app + SQL Server / MySQL / Postgres variants | __inferred__ |

## Test

| Command | Purpose | Status |
| --- | --- | --- |
| `dotnet test src` | full suite (SQLite-backed, no external DB needed) | __inferred__ |
| `dotnet test src --filter "FullyQualifiedName~ProductServiceTests"` | one class | __inferred__ |
| `dotnet test src --filter "Name~CanParseRequiredProductIds"` | one test | __inferred__ |
| `dotnet test src --configuration Release --verbosity normal` | exactly what CI runs | __inferred__ |

## Frontend assets (only when editing css/js sources)

| Command | Purpose | Status |
| --- | --- | --- |
| `cd src/Presentation/Nop.Web && npm install` | fetch gulp toolchain + libs | __inferred__ |
| `npx gulp` | rebuild bundled assets into wwwroot | __inferred__ (tasks defined in [gulpfile.js](../../src/Presentation/Nop.Web/gulpfile.js) — list them with `npx gulp --tasks` first) |

## Migrations / database

No CLI migration command — migrations run **automatically at app startup** via FluentMigrator ([Nop.Data/Migrations](../../src/Libraries/Nop.Data/Migrations)). To add one: create a class in the current `UpgradeTo*` folder with a `[NopSchemaMigration("yyyy-MM-dd HH:mm:ss", "description")]` attribute ([example](../../src/Libraries/Nop.Data/Migrations/UpgradeTo500/PriceListMigration.cs#L7)), then start the app. To reset a dev environment: drop the DB and delete `src/Presentation/Nop.Web/App_Data/appsettings.json`, then rerun the installer. __verified__ (mechanism, by code reading; not executed).

## Code exploration (verified during authoring)

| Command | Purpose | Status |
| --- | --- | --- |
| `rg "class ShoppingCartService" src` | find a type fast | __verified__ (used throughout) |
| `rg -n "IConsumer<OrderPlacedEvent>" src` | find event consumers | __verified__ pattern |
| `rg -n "CheckPermission" src/Presentation/Nop.Web/Areas/Admin/Controllers` | audit admin authz | __verified__ pattern |
| `git log --oneline -20` | recent work + commit message conventions | __verified__ |

## No such thing here

Lint/format commands (none configured), codegen (none), a separate migration CLI (startup-driven), separate services to boot (monolith). If a doc or teammate references one, it's from a different project.
