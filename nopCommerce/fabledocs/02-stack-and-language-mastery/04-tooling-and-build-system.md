# Tooling and Build System

[01-cartography/04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md) covers the inventory; this file covers the *sharp edges* and the transferable skills: how MSBuild assembles this solution, how plugins get into the running app, what the frontend pipeline actually produces, what CI does and does not check, and how to debug "my change didn't show up".

Nothing on this page was built or run while writing it; every claim is read from a build file, and commands are marked *inferred* where they are not copied from CI.

## MSBuild mental model

`dotnet build src` walks the solution's project graph and compiles in dependency order. Shared settings live once in [Directory.Build.props](../../src/Directory.Build.props) — MSBuild imports it automatically into every project below `src/`. It is 13 lines and sets exactly three things: `TargetFramework` `net10.0`, `ImplicitUsings` enabled, and a deliberate `NoWarn` for NuGet vulnerability warnings NU1901–NU1904 with a comment explaining that packages are updated constantly ([Directory.Build.props#L9-L12](../../src/Directory.Build.props#L9-L12)). That file is the answer to "where do I change something for *every* project?" — and its absence of `<Nullable>`, `<TreatWarningsAsErrors>` or analyzer packages is equally informative.

The SDK version is pinned separately in [global.json](../../global.json): `10.0.100` with `rollForward: latestFeature`, prerelease disallowed. Translation: any 10.0.1xx-or-later feature band SDK is accepted, but not 11.x. "Works on my machine" SDK drift starts here.

Project graph, as declared in the `.csproj` files:

| Project | References |
| --- | --- |
| `Nop.Core` | packages only — including `Microsoft.AspNetCore.Mvc.NewtonsoftJson` and `…Razor.RuntimeCompilation` ([Nop.Core.csproj, lines 19–20](../../src/Libraries/Nop.Core/Nop.Core.csproj)) |
| `Nop.Data` | `Nop.Core` + FluentMigrator runners, linq2db, three ADO.NET drivers ([Nop.Data.csproj](../../src/Libraries/Nop.Data/Nop.Data.csproj)) |
| `Nop.Services` | `Nop.Core`, `Nop.Data` ([Nop.Services.csproj, lines 32–33](../../src/Libraries/Nop.Services/Nop.Services.csproj)) |
| every plugin | `Nop.Web.csproj` itself ([Payments.Manual csproj, line 51](../../src/Plugins/Nop.Plugin.Payments.Manual/Nop.Plugin.Payments.Manual.csproj)) |

The last row surprises people: **plugins reference the web application project**, so in the graph they build *after* `Nop.Web`. That's what lets a plugin use controllers, view models and factories from the app — and why a plugin can never be compiled without the whole app.

Version-skew note worth knowing: `Nop.Core` targets `net10.0` but references `9.0.9` versions of several `Microsoft.AspNetCore.*` / `Microsoft.Extensions.Caching.*` packages (same csproj, lines 19–22). That is legal (the shared framework wins for framework assemblies), but it's the kind of thing to check first when you see a binding-redirect-style runtime error after a package update.

## How a plugin reaches the running app

Read one plugin csproj top to bottom — [Nop.Plugin.Payments.Manual.csproj](../../src/Plugins/Nop.Plugin.Payments.Manual/Nop.Plugin.Payments.Manual.csproj) is 61 lines:

1. **Output redirected into the web app.** `OutputPath` is `$(SolutionDir)\Presentation\Nop.Web\Plugins\Payments.Manual` and `OutDir` is set to the same (lines 12–13). There is no separate "copy plugin" step; the compiler writes straight into `Nop.Web/Plugins/<Name>`.
2. **Don't drag NuGet dependencies along.** `CopyLocalLockFileAssemblies` is `false` (line 17), with a comment telling you to flip it if your plugin brings its own NuGet package.
3. **Views and metadata are content.** `plugin.json`, `logo.png` and each `.cshtml` are `CopyToOutputDirectory=PreserveNewest` (lines 29–48). A new view you forgot to add here exists in the source tree but not in the running plugin.
4. **Post-build cleanup.** A `NopTarget` runs `AfterTargets="Build"` and invokes the `NopClear` target of [Build/ClearPluginAssemblies.proj](../../src/Build/ClearPluginAssemblies.proj) (lines 55–59), which executes a **committed, prebuilt** `ClearPluginAssemblies.dll` to delete assemblies the app already has. The source for that DLL lives alongside it ([Build/src/ClearPluginAssemblies](../../src/Build/src/ClearPluginAssemblies)) and [Note.txt](../../src/Build/Note.txt) explains why.
5. **Plugin output is not committed.** [.gitignore](../../.gitignore) ignores `src/Presentation/Nop.Web/Plugins/*` except the `Uploaded` placeholder — so a fresh clone has *no* plugins until you build.

Sharp edge: a plugin that fails to copy shows up as a runtime "plugin not found," not a build error. When a plugin change seems ignored, check the copied output under `src/Presentation/Nop.Web/Plugins`, not just the build log.

## The two build systems

C# and the frontend pipeline are independent. The gulp default task is `series(clean, copyDependencies, prepareCldr)` ([gulpfile.js#L8-L10](../../src/Presentation/Nop.Web/gulpfile.js#L8-L10)); `copyDependencies` copies selected files out of `node_modules` into **`wwwroot/lib_npm/`** ([gulp/tasks/copyDependencies.js#L9](../../src/Presentation/Nop.Web/gulp/tasks/copyDependencies.js#L9)), and `clean` deletes that same folder first ([gulp/tasks/clean.js#L2](../../src/Presentation/Nop.Web/gulp/tasks/clean.js#L2)). The dependency set is jQuery-era: gulp 4, Bootstrap 4.6, AdminLTE 3.2, jQuery 3.7 ([package.json](../../src/Presentation/Nop.Web/package.json)).

CI runs only the .NET side ([dotnet.yml](../../.github/workflows/dotnet.yml)), so the gulp output must already be in the repo — and it is: `git ls-files src/Presentation/Nop.Web/wwwroot/lib_npm | wc -l` returned **1269** tracked files on 2026-10-06, while `.gitignore` excludes only most of the `cldr-data/main` locales. Transferable lesson: always ask "which artifacts are generated, and which generated artifacts are *committed*?" Committed-generated files are a classic source of "my change disappeared" — an edit inside `wwwroot/lib_npm` is wiped the next time anyone runs `gulp`, because `clean` deletes the folder. Edit the source in `node_modules`' package (i.e. bump the dependency) or add your own file outside `lib_npm`.

## CI: what is checked, exactly

[dotnet.yml](../../.github/workflows/dotnet.yml) is 28 lines. On push and PR to **`develop`** (not `main` — lines 7–10) it runs on `windows-latest` with .NET `10.0.x`:

```bash
dotnet restore src
dotnet build --no-restore --configuration Release src
dotnet test --no-build --configuration Release --verbosity normal src
```

What that means in practice:

- One OS. Nothing in CI proves the Linux container image works; that path is covered only by the [Dockerfile](../../Dockerfile) (Alpine SDK image, `dotnet build NopCommerce.sln --no-incremental -c Release`, then `dotnet publish`).
- One database in tests: SQLite via the test project's own provider (see [05-quality-engineering/01-testing-strategy.md](../05-quality-engineering/01-testing-strategy.md)); the three production providers are not exercised by CI.
- No formatting gate, no analyzer package, no coverage threshold, no frontend build.

## Style tooling that *does* exist

There is a 235-line [.editorconfig](../../.editorconfig) at the repo root. It sets indentation, `charset = utf-8-bom` for C# files, and a set of `dotnet_naming_rule` entries — most at `warning` severity, with `async_methods_end_in_async` at `suggestion` (lines 128–130) and `file_scoped` namespaces as a suggestion (line 235). So style intent is encoded, but CI never fails on it: naming warnings appear in IDE/build output and are not treated as errors. The accurate interview framing is: "conventions are declared in `.editorconfig` and enforced socially in review, not mechanically in CI." This project substitutes *convention density* (look at any two services) for tooling enforcement — which works at high convention density and fails on younger teams. Knowing *why* a control is absent beats knowing the control exists.

## Test tooling

NUnit 4.5.1 + NUnit3TestAdapter, Moq 4.20.72, AwesomeAssertions 9.4.0, `Microsoft.Data.Sqlite` and `FluentMigrator.Runner.SQLite` ([Nop.Tests.csproj](../../src/Tests/Nop.Tests/Nop.Tests.csproj)). One test project mirroring the library structure. The heavy lift is [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98): builds a real DI container, initializes a SQLite database via the actual installation service ([L109-L128](../../src/Tests/Nop.Tests/BaseNopTest.cs#L109-L128)), so `GetService<T>()` ([L572](../../src/Tests/Nop.Tests/BaseNopTest.cs#L572)) hands you fully wired services. Cost: first-test startup is slow (full install with sample data); benefit: tests exercise real SQL and real service interactions. Targeted runs:

```bash
# __inferred__ (standard dotnet test filter syntax; not run here)
dotnet test src --filter "FullyQualifiedName~OrderProcessing"
dotnet test src --filter "Name~ShouldBeAvailableWhenStartDateIsNotSet"
```

Note that the second filter matches **two** tests — the name exists in both [ProductServiceTests.cs#L79](../../src/Tests/Nop.Tests/Nop.Services.Tests/Catalog/ProductServiceTests.cs#L79) and [BlogServiceTests.cs#L35](../../src/Tests/Nop.Tests/Nop.Services.Tests/Blogs/BlogServiceTests.cs#L35). Use `FullyQualifiedName~` with the class name when you mean one.

## Exercise ladder

1. **Goal:** find every place a build setting can come from.
   **Check:** list the files: `Directory.Build.props`, `global.json`, each `.csproj`, `Build/ClearPluginAssemblies.proj`. For `TargetFramework`, show it is set in *both* `Directory.Build.props` and individual plugin csproj files (e.g. Payments.Manual line 4) — and say which wins (the project file, because it is evaluated after the props import).
2. **Goal:** trace a plugin view from source to runtime.
   **Check:** for `Views/Configure.cshtml` in Payments.Manual, cite the csproj lines that copy it and the output folder it lands in (lines 12, 36–38).
3. **Goal:** prove the frontend output is committed and regenerated.
   **Check:** `git ls-files src/Presentation/Nop.Web/wwwroot/lib_npm | wc -l` (non-zero) plus `clean.js` line 2 (target folder). Conclude what happens to a hand edit.
4. **Goal (mid):** explain what CI cannot catch.
   **Check:** from `dotnet.yml` alone, list three classes of regression that would pass CI (examples: Linux-only path bugs, MySQL/PostgreSQL SQL dialect issues, broken gulp output).
5. **Goal (mid) — run locally:** compare solution vs project builds of a plugin.
   **Check (run locally; expected outcome derived from MSBuild semantics, not observed):** `dotnet build src/Plugins/Nop.Plugin.Payments.Manual/Nop.Plugin.Payments.Manual.csproj` without the solution leaves `$(SolutionDir)` undefined, so `OutputPath` resolves to `\Presentation\Nop.Web\Plugins\Payments.Manual` relative to the drive root rather than inside `src/`. Confirm by looking for that folder; then rebuild via `dotnet build src` and compare.

## Senior finding: plugin builds depend on `$(SolutionDir)`

Every plugin csproj builds its output path from `$(SolutionDir)` (example: [Payments.Manual line 12](../../src/Plugins/Nop.Plugin.Payments.Manual/Nop.Plugin.Payments.Manual.csproj), also its `ProjectReference` and `ClearPluginAssemblies` item on lines 51–52). `SolutionDir` is a property MSBuild defines when building *through a solution file*; building a single project file doesn't define it. So the build is correct only when invoked via `NopCommerce.sln` (as CI and the Dockerfile both do). Practical effect for a developer: "build just my plugin" from the command line produces output in an unexpected location, and the running app keeps loading the old plugin — the classic "my change didn't show up" with a clean build log. Confidence: high that the property is used this way (read from the csproj); medium on the exact resulting path on your machine — verify with rung 5. A robust alternative is a path relative to `$(MSBuildThisFileDirectory)`, which `ClearPluginAssemblies.proj` itself already uses (`$(MSBuildProjectDirectory)`).

Second, smaller observation: the build executes a committed binary (`src/Build/ClearPluginAssemblies.dll`, tracked in git alongside its source). Supply-chain reviewers will ask "how do we know the DLL matches the source?" — there is no CI step that rebuilds it.

## Debugging scenario: "my change disappeared"

Three different root causes produce the same complaint; triage by *what* changed:

| What you edited | Likely cause | Evidence to check |
| --- | --- | --- |
| a file in `wwwroot/lib_npm` | gulp `clean` + `copyDependencies` regenerated the folder | [clean.js#L2](../../src/Presentation/Nop.Web/gulp/tasks/clean.js#L2) |
| a plugin `.cs` or `.cshtml` | plugin built outside the solution (finding above), or the view isn't listed as `Content` | plugin csproj lines 12, 29–48 |
| a setting value in `appsettings.json` | you changed an *app setting* but the behaviour is a DB *setting* (or vice versa) | [Program.cs#L14-L20](../../src/Presentation/Nop.Web/Program.cs#L14-L20) vs `ISettings` registration in [NopStartup.cs#L273-L283](../../src/Presentation/Nop.Web.Framework/Infrastructure/NopStartup.cs#L273-L283) |

Drill: from a cold clone, list the exact commands (in order) to get a red-green cycle on one service test. Time yourself; under 15 minutes including restore is passing.

## Verification

2026-10-06: corrected the gulp destination from `wwwroot/lib` to `wwwroot/lib_npm` (from `copyDependencies.js` line 9) and the claim "no analyzers/StyleCop config" to the accurate "a 235-line `.editorconfig` with naming rules exists; nothing in CI enforces it". Package versions read from `Nop.Tests.csproj` and `Nop.Core.csproj`; tracked-file counts from `git ls-files`. The `$(SolutionDir)` behaviour is reasoned from MSBuild semantics and marked for local confirmation.
