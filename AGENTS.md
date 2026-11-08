# serilog-dotnet — working in this repo

Extensions for Serilog — a logging baseline as one `LoggerConfiguration` call: the log context, thread, process and exception-detail enrichers, plus the entry assembly's `Version` and `Application` — shipped on NuGet as `JorgeCostaMacia.Serilog`. Moved out of **shared-net** with the 10.2 versions, keeping the package id and the version line. An independent project, not affiliated with Serilog: the README says so, and the repo name must never read as Serilog itself.

## Layout

- `src/JorgeCostaMacia.Serilog/` — the package. `test/JorgeCostaMacia.Serilog.Tests/` — its tests, mirroring the package's folders (`Infrastructure/`); a real logger writes into `Serilog.Sinks.InMemory`, so nothing external is needed. `assets/` — icons + social preview.
- **3-tier `Directory.Build.props`**: **root** (repo identity — Authors / Company / Copyright / Repository — + the single `VersionPrefix`; TFM `net10.0`; ImplicitUsings, Nullable, AnalysisLevel, EnforceCodeStyleInBuild, TreatWarningsAsErrors, the NuGet audit; `IsPackable` false by default) → **`src/`** (`IsPackable` true, package-output: icon / readme / license, SourceLink, symbols, `GenerateDocumentationFile`, pack of LICENSE/COPYRIGHT/icon/README) → **`test/`** (test settings). The `src` csproj declares **only** `Description` / `PackageTags` and its package references; everything else is inherited from the props (don't restate it).

## Targets & stack

- TFM: **`net10.0`**, single-target. **Do not reintroduce a per-TFM conditional** — if a dependency needs one, the answer is to move the whole repo, not to split it.
- Serilog: **4.x**, with the three packages the baseline needs — `Serilog.Enrichers.Thread`, `Serilog.Enrichers.Process` and `Serilog.Exceptions`. Nothing else: the sinks, and `Serilog.Settings.Configuration` for `ReadFrom.Configuration`, are the host's choice and stay with it, so this package never drags a sink into an app.
- Tests: **xUnit v4 (the `xunit.v3` package, 4.x) on Microsoft.Testing.Platform v2 (MTP)** — test projects are `OutputType=Exe`, `dotnet test` runs MTP because the root **`global.json`** opts in, and `xunit.runner.visualstudio` is referenced for IDE discovery, as `testing-net` sets up. Running them needs the **.NET 10 SDK**.
- Source is **UTF-8 without BOM** (`.editorconfig` `charset = utf-8`). camelCase locals, PascalCase types, I-prefixed interfaces. Copyright year stays **2023** (deliberate — don't bump).
- **Explicit types everywhere — spell the type out.** Never `var`, never target-typed `new()`, never collection expressions `[]`: write `new Foo(...)`, `new byte[] { ... }`, `new List<T> { ... }`, `Array.Empty<T>()`. The `.editorconfig` sets all three to explicit, but only `var` is analyzer-enforced — the analyzer never reports the implicit forms. Develop (`dotnet-build-nuget` from github-workflows) guards target-typed `new(` and implicit `new[]` with a **grep step** that fails the build, and collection expressions `[…]` with a second one (`code-style-net`'s pattern: attributes, indexers, array types and list patterns never match it). Do not introduce any of them.

## Extend the framework's options, never replace its call

The baseline is an **extension on the type Serilog already hands the host** — `LoggerConfiguration.WithDefaults()` — and never an `Add…` method that wraps the framework's own. The host keeps writing `AddSerilog(…)`, so what it composes stays readable in its `ProgramBuilder`; only the policy moves here. quartz-dotnet and `http-net` follow the same rule with their options extensions.

An `AddSerilogContext` facade that replaced `AddSerilog` was removed in 7.0.0 for breaking it: no host had adopted it, each had hand-rolled the enrichers instead. **Don't add another.** A policy belongs in an extension when Serilog cannot read it from configuration — the entry assembly's `Version` and `Application`, and the enrichers every app shares; when it can — sinks, levels and overrides — leave it in `appsettings`, where it can differ per app and per environment.

## Dependencies on the family

Nothing here depends on another `JorgeCostaMacia.*` package, and nothing in the family needs this one: `Quartz.Serilog` (quartz-dotnet) takes the third-party `Serilog`, not this baseline. That is what lets this repo release on its own. Keep it that way.

## Dependencies — Central Package Management

Third-party package versions are centralized in **`Directory.Packages.props`** (repo root, `ManagePackageVersionsCentrally=true`): add or bump them **there** as `<PackageVersion>`, and reference packages in csproj **without** a `Version`.

**The strict gates are on**: `TreatWarningsAsErrors` and the NuGet vulnerability audit on every restore (direct and transitive, every severity), so any warning — an advisory included — fails the build. The fix is a version bump or a transitive override in CPM, never `NoWarn`. The root `nuget.config` clears every inherited source and keeps nuget.org alone: this package consumes nothing else, and an inherited second source would raise `NU1507`. It is **deny-read** for agents — hand changes to it to the user.

## CI / publishing

The three workflows call the reusable ones of [github-workflows](https://github.com/JorgeCostaMacia/github-workflows) (`@v1`): `develop.yml` runs `dotnet-build-nuget` on develop and PRs (the explicit-types and collection-expression greps, build, the `dotnet format` check, test, pack — no publish); `main.yml` runs the same build on a push to `main` and then pushes the `nuget` artifact to nuget.org with Trusted Publishing (`NuGet/login`, OIDC) — that job stays here because the nuget.org policy names this repo's `main.yml`; `release.yml` creates the GitHub Release of each `v*` tag. **The (central) `VersionPrefix` is the publish gate**: `--skip-duplicate` leaves an existing version alone, so only a release that moved it publishes.

## Branching & releases — GitFlow

Use the **`gitflow` skill** for any branch/release work — never improvise.

- Feature/bugfix → `feature/`|`bugfix/<name>-<ts>` from develop → finish `--no-ff` into develop.
- Release → `release/<version>` from develop → bump the **single** `VersionPrefix` in the **root** `Directory.Build.props` (the one holding the solution identity) → Release Finish (merge develop+main, annotated tag `v<version>`, atomic push).
- Use git's **default merge message** (`--no-ff --no-edit`, never `-m`).
- Branch prefixes only: `feature` / `bugfix` / `release` / `hotfix`.

## Git etiquette

- Commit under **your own identity** — don't hardcode anyone's name/email.
- Keep history clean — **no** `Co-Authored-By` / AI-assistant trailers in commits or messages.
- Merges use git's **default** message (see *Branching & releases* above).

## Relevant skills

Skills that apply to this repo — let them trigger, or invoke explicitly. The first group is from `jorgecostamacia-agent-skills` (each `-net` skill brings the generic one it realizes); the rest from `dotnet-agent-skills` (the `dotnet/skills` marketplace).

A skill listed here but missing on this machine is one command away: `claude plugin marketplace add JorgeCostaMacia/skills` (or `dotnet/skills`) once, then `claude plugin install <skill>@<marketplace>`.

- **`logging-net`** (with **`logging`**) — **the spec this package implements**: its *Wiring* section is `WithDefaults()` — `FromLogContext`, without which no pushed property reaches the line, the thread and process ids, structured exception details, and the entry assembly's `Version` and `Application` — and its *Configuration* section is what stays in `appsettings`. A change here is a change in what that skill describes: keep the two in step.
- **`code-style-net`** — how the C# is written: explicit types, the 120-column width with declarations and constructions vertical, comments wrapped too, the gates.
- **`build-net`** — the three-tier `Directory.Build.props`, the single `VersionPrefix`, CPM, the three-day package age, the NuGet audit as a build gate, the library's packing metadata, and the `nuget.config` base.
- **`testing-net`** (with **`testing`**) — xUnit v3 on Microsoft.Testing.Platform, the test project layout, behaviour asserted through a real logger and an in-memory sink.
- **`solid`** — design review of the public surface.
- **`gitflow`** — all branch/release work.
- **`dotnet`** — C# language server + general .NET development.
- **`dotnet-msbuild`** — `Directory.Build.props`, project-file quality, CPM.
- **`dotnet-nuget`** — dependency management.
- **`dotnet-test`** / **`dotnet-test-migration`** — tests; the xUnit v4 / MTP v2 setup.
- **`dotnet-upgrade`** — target-framework migrations.

Not relevant to this repo (skip): `ddd` / `ddd-net`, `validation-net`, `clean-architecture-net` (one extension method, no modules or composition contexts), `efcore-net` / `efcore-postgres-net`, `messaging` / `bus-net` / `bus-kafka-net`, `github-actions` / `github-actions-net` (the company's CI; this repo's runs on github-workflows), `dotnet-aspnetcore` (that's http-net), `dotnet-data`, `dotnet-ai`, `dotnet-maui`, `dotnet-blazor`, `dotnet-template-engine`, `dotnet11`, `dotnet-diag`, `dotnet-advanced`.

## Build & test

```
dotnet format serilog-dotnet.slnx                  # apply .editorconfig (using order, whitespace) — run before committing
dotnet build  serilog-dotnet.slnx -c Release
dotnet test   serilog-dotnet.slnx -c Release       # MTP v2 via global.json (needs the .NET 10 SDK); --logger is VSTest-only (MTP0001)
dotnet pack   serilog-dotnet.slnx -c Release        # packs the package; the tests are IsPackable=false
```

The tests need nothing external: they build a real logger with `WithDefaults()` and read what reached an in-memory sink.

Run **`dotnet format` before committing** — it applies the `.editorconfig` (using ordering, whitespace, line endings), the CLI equivalent of Visual Studio's *Code Cleanup*, and it is the normalizer for a `.cs` a shell tool wrote with LF. Develop checks it after the build with `dotnet format --verify-no-changes --no-restore`: the solution is clean, so the check only ever fails on what the push itself brought. `.gitattributes` checks `*.cs` out as CRLF (`eol=crlf`), so the Linux runner sees what the `.editorconfig` requires.
