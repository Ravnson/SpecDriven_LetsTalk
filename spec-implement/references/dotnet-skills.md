# dotnet/skills routing for `/spec-implement`

Source: <https://github.com/dotnet/skills> — the .NET team's official agent skills,
published as the `dotnet-agent-skills` plugin marketplace.

**Rule:** when a situation below is covered by a skill, invoke that skill via the `Skill`
tool instead of improvising. These skills encode the current .NET tooling behaviour, which
is more reliable than recall.

---

## Availability check (Step 6)

**Filesystem only — this check never goes online.** Look for installed dotnet plugins:

```bash
ls ~/.claude/plugins/marketplaces/*/plugins 2>/dev/null | grep -i dotnet
```

and cross-check against the available-skills listing in the session context. Plugin skills
are invoked as `plugin:skill`, e.g. `dotnet-test:run-tests`.

If nothing is installed, mention it **once** and continue with the plain `dotnet` CLI. Do
not block, and do not attempt the install — it is an online action and therefore the
user's, not the skill's. Offer the commands only if the user asks how:

> ```
> /plugin marketplace add dotnet/skills
> /plugin install dotnet@dotnet-agent-skills
> /plugin install dotnet-test@dotnet-agent-skills
> /plugin install dotnet-msbuild@dotnet-agent-skills
> ```
> Restart Claude Code afterwards. Updates: `/plugin update <plugin>@dotnet-agent-skills`.

`/plugin` opens an interactive panel and is only available from an interactive `claude`
terminal session.

---

## Routing table

| Situation during implementation | Plugin | Skill |
|---|---|---|
| Navigate, rename, find references, read diagnostics in C# | `dotnet` | C# language-server (LSP) integration — use it instead of text-only grep for symbol work |
| Local SDK/TFM mismatch, `global.json` trouble | `dotnet` | `setup-local-sdk` |
| Run the suite, pick a subset, tight red/green loop | `dotnet-test` | `run-tests`, `filter-syntax`, `platform-detection` |
| Write new tests (MSTest repo) | `dotnet-test` | `writing-mstest-tests` |
| No test project exists yet for the area | `dotnet-test` | `scaffold-dotnet-test-project` |
| Review the tests you just wrote | `dotnet-test` | `test-anti-patterns`, `test-smell-detection`, `assertion-quality`, `grade-tests` |
| Coverage delta and untested new code | `dotnet-test` | `coverage-analysis`, `find-untested-sources`, `test-gap-analysis`, `crap-score` |
| Existing code is untestable (statics, sealed externals, `DateTime.Now`) | `dotnet-test` | `testability-obstacle`, `detect-static-dependencies`, `generate-testability-wrappers`, `migrate-static-to-wrapper` |
| Organising/tagging a growing suite | `dotnet-test` | `test-tagging`, `code-testing-extensions`, `test-analysis-extensions` |
| Hot-reload driven test loop | `dotnet-test` | `mtp-hot-reload` |
| Build fails and the message is not self-explanatory | `dotnet-msbuild` | `binlog-generation`, `binlog-failure-analysis` |
| Build is slow after your change | `dotnet-msbuild` | `build-perf-diagnostics`, `build-perf-baseline`, `incremental-build`, `build-parallelism`, `eval-performance`, `msbuild-server` |
| Touching `.csproj` / `Directory.Build.props` / `Directory.Packages.props` | `dotnet-msbuild` | `directory-build-organization`, `property-patterns`, `item-management`, `msbuild-antipatterns`, `msbuild-modernization` |
| Content/generated files not landing in output | `dotnet-msbuild` | `copy-to-output-directory`, `including-generated-files`, `check-bin-obj-clash` |
| Adding a custom target or project reference wiring | `dotnet-msbuild` | `target-authoring`, `extension-points`, `resolve-project-references` |
| Add, pin, or update a package; central package management | `dotnet-nuget` | dependency-management and modernization skills |
| ASP.NET Core endpoints, middleware, API shape | `dotnet-aspnetcore` | `dotnet-webapi`, `minimal-api-file-upload` |
| Telemetry / observability is in the AC | `dotnet-aspnetcore` | `configuring-opentelemetry-dotnet` |
| EF Core queries, migrations, data access | `dotnet-data` | `optimizing-ef-core-queries`, `create-datadriven-aspnetcore` |
| Blazor component or UI change | `dotnet-blazor` | `plan-ui-change`, `author-component`, `coordinate-components`, `collect-user-input`, `fetch-and-send-data`, `configure-auth`, `support-prerendering`, `use-js-interop`, `create-blazor-project` |
| A performance NFR must be proven, not asserted | `dotnet-diag` | `microbenchmarking`, `analyzing-dotnet-performance` |
| Investigating a hang, leak, or crash during implementation | `dotnet-diag` | `dotnet-trace-collect`, `dump-collect`, `clr-activation-debugging` |
| MAUI app work | `dotnet-maui` | environment setup, diagnostics, navigation, binding, DI, layout, theming |
| An LLM/AI feature is part of the PBI | `dotnet-ai` | `technology-selection` |
| Scaffolding a new project or authoring a template | `dotnet-template-engine` | template discovery, scaffolding, authoring |
| The PBI requires a TFM or framework upgrade | `dotnet-upgrade` | migration/upgrade skills — **stop first**: an upgrade is almost never in a feature PBI's scope. Raise it as a follow-up or a separate enabler |
| Test framework/platform migration (xUnit→MSTest, VSTest→MTP) | `dotnet-test-migration` | migration skills + orchestrator agent — same scope warning as above |
| P/Invoke, C# scripts, NuGet trusted publishing | `dotnet-advanced` | `dotnet-pinvoke`, `csharp-scripts`, `nuget-trusted-publishing` |
| Using a .NET 11 API or language feature | `dotnet11` | new API/feature skills |

---

## Scope guard

Several of these skills (upgrade, test migration, msbuild modernization, build perf) can
cascade far beyond one PBI. Invoke them for **diagnosis** freely; before applying a
sweeping change, check it against the PBI's acceptance criteria. If it is not required by
an AC, it belongs in `implementation.md` Follow-ups and, if the user wants it, a separate
ADO work item — not in this branch.

---

## Fallback commands (no plugins installed)

```bash
dotnet build -warnaserror
```

```bash
dotnet test --filter "FullyQualifiedName~<Namespace>"
```

```bash
dotnet format --verify-no-changes
```

```bash
dotnet build -bl
```

(`-bl` writes `msbuild.binlog` for failure analysis.)

Always prefer the commands `CLAUDE.md` specifies over these defaults.
