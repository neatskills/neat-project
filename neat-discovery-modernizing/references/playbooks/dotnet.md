# .NET Playbook

Tech stack: .NET — covers .NET Framework, .NET Core, and .NET 5+ (C# projects).

## Evidence

Additional scan targets in `legacy_path`:

- `.csproj` files — `<TargetFramework>` or `<TargetFrameworks>` values (framework version)
- `.sln` files — project count and solution structure
- `global.json` — SDK version pinning
- `packages.config` — legacy NuGet format (pre-PackageReference)
- `Startup.cs` — presence indicates pre-.NET 6 hosting model
- `appsettings*.json` — configuration pattern and environment model

Additional documents to request after standard document intake:
> "Share any available: Azure DevOps pipeline definitions, IIS / App Service configuration, deployment slot or environment configs, Azure resource group or Bicep templates."

## Evaluate

Dimension overrides — apply these in place of the generic signals for the specified dimensions.

**Dimension 4 — Technology health:**

| Signal | Red | Amber | Green |
| --- | --- | --- | --- |
| Target framework | .NET Framework ≤ 4.7.2, .NET Core ≤ 2.1, or .NET 5 (EOL) | .NET Framework 4.8 or .NET 6 (EOL Nov 2024) | .NET 8 (LTS) or .NET 9+ |
| NuGet format | `packages.config` present in any project | Mix of `packages.config` and `PackageReference` | All projects use `PackageReference` |
| Hosting model | `Startup.cs` present with no minimal-API `Program.cs` pattern | `Startup.cs` present alongside a minimal-API `Program.cs` | No `Startup.cs`; minimal API / top-level statements only |

**Dimension 2 — Changeability:**
Test harness signal: presence of `xunit`, `nunit`, or `mstest` package reference in any `.csproj`. Apply in addition to the standard test-file count heuristic.

## Derivation

Stack-specific rules — evaluated before standard Rules 1–10; first-match-wins applies across the combined list.

| Priority | Condition | Phase inserted |
| --- | --- | --- |
| P1 | Target framework is any .NET Framework version | Framework-to-Core migration — mandatory first phase; subsequent phases operate on the migrated codebase. Bundle `Startup.cs` hosting model migration and `packages.config` migration into this phase. |
| P2 | `packages.config` present and P1 did not fire | NuGet format migration — convert to `PackageReference` before other structural changes |

## Design

Additional patterns — extend the Pattern Selection table with these before determining the pattern for each phase.

| Pattern | Use when |
| --- | --- |
| Framework-to-Core migration | Target framework is .NET Framework — run `dotnet-upgrade-assistant` alongside manual compatibility shim resolution; migrate projects in dependency order (leaf projects first) |
| Minimal API migration | `Startup.cs` present after framework migration — consolidate middleware and DI registrations into `Program.cs` top-level statements |
| PackageReference migration | `packages.config` present — convert using `dotnet migrate-packages` or manual edit; verify no transitive version conflicts after conversion |
