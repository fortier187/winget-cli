# GitHub Copilot / AI Agent Instructions for winget-cli

Quick orientation
- Purpose: this repository implements the Windows Package Manager (winget) client: a mixed native/.NET/PowerShell codebase that builds a UWP/Appx client (App Installer) and supporting libraries.
- Immediate entry points: `src/WindowsPackageManager/main.cpp` (native entry), `src/AppInstallerCLICore/` (command parsing & workflows), PowerShell modules in `tools/PowerShell/`.

What an AI agent should know (high-level)
- Architecture:
  - Core CLI logic and command definitions live in `src/AppInstallerCLICore/`. Commands are small classes that call into `Workflows/` implementations.
  - Native runtime, COM/WinRT projection and packaging live in `src/WindowsPackageManager/` and related `*.vcxproj` projects (`InProc` / `OutOfProc` COM support and WinRT metadata/winmds).
  - Test layers: unit tests in `AppInstallerCLITests`, integration & E2E in `AppInstallerCLIE2ETests` (has TestData and depends on native binaries), fuzzer in `WinGetYamlFuzzing`.
  - PowerShell integration: `tools/PowerShell/*` produce PowerShell modules that call into the app's APIs; expect extra packaging and shared native binaries copying in CI.

Key developer workflows (how to build / run / test locally)
- Environment (Windows, Visual Studio 2022, Windows SDK 10.0.26100). See `doc/Developing.md` for full details.
- Quick reproduce of CI build locally:
  1. Run one of the repository configurations: `winget configure .config/configuration.winget` (choose `configuration.winget`, `.vsProfessional` or `.vsEnterprise` as applicable).
  2. Open `src\AppInstallerCLI.sln` in Visual Studio and Build (or use `msbuild`/`VSBuild` with the same MSBuild properties used in `azure-pipelines.yml`).
  3. `vcpkg integrate install` is required for vcpkg-managed deps.
- Run client locally: Deploy `AppInstallerCLIPackage` (Build > Deploy Solution) and use `wingetdev` to exercise the client; to debug, set `AppInstallerCLIPackage` Debugger type to **Native Only** and start the session, then run `wingetdev` in a terminal.
- Unit tests: build -> run `AppInstallerCLITests.exe` (test executable under `src/<Arch>/<Config>/AppInstallerCLITests`) or use Test Explorer.
- E2E tests: `AppInstallerCLIE2ETests` is a `net8.0-windows` test project (NUnit). You can run via Visual Studio Test Explorer or `dotnet test` after ensuring native artifacts are built and copied (the csproj has post-build copy steps). Use `Test.runsettings` where needed.

Project-specific patterns & conventions (do these exactly)
- Commands & flows: Add CLI commands in `src/AppInstallerCLICore/Commands/` and business logic in `src/AppInstallerCLICore/Workflows/`. Examples:
  - Add new command: create `YourCommand.h/.cpp` in `Commands/` and add `std::make_unique<YourCommand>(FullName())` to `RootCommand::GetCommands()` in `RootCommand.cpp`.
  - Completion: handled via `CompleteCommand.cpp` and `CompletionFlow.cpp`/`CompletionData.*`.
- Tests: Add unit tests to `AppInstallerCLITests` and integration tests to `AppInstallerCLIE2ETests`. E2E tests often depend on test assets under `TestData/` and native binaries copied by project PostBuild targets.
- Style & analyzers: StyleCop and Roslyn analyzers are enforced (`stylecop.json`, `CodeAnalysis.ruleset`). Release builds treat warnings as errors — keep code style and warnings clean.
- Native/managed interop: watch for CsWinRT and `winmd` interop. If you change public WinRT APIs or projection types, update related projection projects and csproj copy steps (see `AppInstallerCLIE2ETests` post-build target).

CI and packaging gotchas (where to look & what to mirror)
- CI is driven by `azure-pipelines.yml`:
  - NuGet & Dotnet restore -> `VSBuild` -> packaging steps that create Appx layout using `Execute-AppxRecipe.ps1` in `src/AppInstallerCLIPackage`.
  - CI copies native artifacts into managed PowerShell module outputs; local builds must replicate these copy rules if running PowerShell tests.
- Packaging: appx recipe and MSIX test installers are generated in the `Build` job; `Execute-AppxRecipe.ps1` is the canonical script used for layout.

Integration / external dependencies to be aware of
- Default data sources: the `winget` community repository (winget-pkgs) and `msstore` (Microsoft Store). The repo also supports custom REST-based sources (`WinGet REST source` project referenced in README).
- Telemetry & privacy: locally built clients do not enable telemetry. See `PRIVACY.md` and `README.md` for details.
- Fuzzing & security: `WinGetYamlFuzzing` uses ASan/OneFuzz; for fuzz-related work check `src/WinGetYamlFuzzing/README.md` and `SECURITY.md` for vulnerability reporting.

Useful files to inspect for most tasks
- `doc/Developing.md` — step-by-step setup and debugging notes (must-read)
- `azure-pipelines.yml` — canonical CI steps (mirrors what needs to run for PRs)
- `src/AppInstallerCLICore/` — where CLI & workflow logic lives
- `src/WindowsPackageManager/` — native entry points and COM/WinRT wiring
- `src/AppInstallerCLITests/` & `src/AppInstallerCLIE2ETests/` — unit and integration tests
- `tools/PowerShell/` — PowerShell module implementations and examples

Do NOT assume (explicit warnings)
- Don’t assume a Linux/macOS build path — the repo is Windows-first with UWP/WinRT and Visual Studio-centric tooling.
- Don’t assume managed test projects automatically find native artifacts — the csproj post-build copy steps (or CI copy steps) are required to run E2E tests successfully.

Example quick tasks (copy/paste style)
- Run unit tests locally:

  "Open `src\AppInstallerCLITests` output dir and run `AppInstallerCLITests.exe`"

- Add new CLI command (minimal set):
  1. Add `src/AppInstallerCLICore/Commands/YourCommand.h` and `.cpp` following existing command classes.
  2. Add `#include "YourCommand.h"` and `std::make_unique<YourCommand>(FullName()),` to `RootCommand::GetCommands()`.
  3. Add unit tests to `AppInstallerCLITests` and an E2E to `AppInstallerCLIE2ETests` if it requires native behavior.

Feedback & iteration
- If anything here is unclear or you want the file to include explicit runnable commands for local reproduction (msbuild examples, exact msbuild properties to mirror CI), tell me which area to expand and I’ll update the file.

---
(If you'd like, I can: add more exact msbuild command lines to replicate CI locally, add a small checklist for authors submitting PRs with native + managed changes, or produce an examples section with exact `dotnet test`/`msbuild` commands.)
