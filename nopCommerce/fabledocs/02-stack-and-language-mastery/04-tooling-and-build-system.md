# Tooling and Build System

Shorter than the other files because [01-cartography/04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md) covers the inventory; this file covers the *sharp edges* and the transferable skills.

## MSBuild mental model

`dotnet build src` walks the solution's project graph and compiles in dependency order (Core → Data → Services → Framework → Web → plugins → tests). Shared settings live once in [Directory.Build.props](../../src/Directory.Build.props) — target framework, implicit usings, and a deliberate `NoWarn` for NuGet vulnerability warnings (NU1901-04) with a comment explaining why. That file is the answer to "where do I change something for *every* project?"

Sharp edge: plugins build into the web app's `Plugins` folder via custom build targets — a plugin that fails to copy shows up as a runtime "plugin not found," not a build error. When a plugin change seems ignored, check the copied output under `src/Presentation/Nop.Web/Plugins`, not just the build log.

## The two build systems

C# and the frontend pipeline are independent: gulp copies/bundles assets from `node_modules` into `wwwroot/lib` ([gulpfile.js](../../src/Presentation/Nop.Web/gulpfile.js), [package.json](../../src/Presentation/Nop.Web/package.json)). CI runs only the .NET side ([dotnet.yml](../../.github/workflows/dotnet.yml)) — bundled assets are committed. Transferable lesson: always ask "which artifacts are generated, and which generated artifacts are *committed*?" Committed-generated files are a classic source of "my change disappeared" (editing the output instead of the source) — check whether a `wwwroot/lib` file you're about to edit is gulp-managed first.

## Test tooling

NUnit + one test project mirroring the library structure. The heavy lift is [BaseNopTest](../../src/Tests/Nop.Tests/BaseNopTest.cs#L98): builds a real DI container, initializes a SQLite database via the actual installation service (L109-L116), so `GetService<T>()` (L572) hands you fully wired services. Cost: first-test startup is slow (full install); benefit: tests exercise real SQL and real service interactions. Targeted runs:

```bash
# __inferred__
dotnet test src --filter "FullyQualifiedName~OrderProcessing"
dotnet test src --filter "Name~ShouldBeAvailableWhenStartDateIsNotSet"
```

## What's missing (and what that teaches)

No analyzers/StyleCop config, no formatter check, no coverage gate in CI. In interviews, don't recite "every repo needs linters" — the mature framing: this project substitutes *convention density* (20 years of extremely uniform code — look at any two services) for tooling enforcement, which works at high convention density and fails on younger teams. Knowing *why* a control is absent beats knowing the control exists.

Drill: from a cold clone, list the exact commands (in order) to get a red-green cycle on one service test. Time yourself; under 15 minutes including restore is passing.
