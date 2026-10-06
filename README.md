# Omega WinUI

**Omega for Windows desktop** — a WinUI 3 / Windows App SDK music app with **no login and no account**, talking **directly to JioSaavn's upstream API** (the same design as the Android Omega app). Favourites, downloads, history and playlists will live only on the local PC (later phases).

> **WinUI 3 builds only on Windows.** This repo is developed CLI-first with **WinApp CLI 0.7+** and **Visual Studio 2026** as the IDE shell (editor + debugger around the same pipeline). Nothing in this project can be compiled on macOS/Linux.

---

## Project layout

```
OmegaWinUI.sln
global.json                  # .NET SDK pin (10.0.100, rollForward latestFeature)
Directory.Build.props        # Nullable / LangVersion / central package mgmt
Directory.Packages.props     # the single version truth (all projects)
src/
  OmegaWinUI/                # WinUI 3 app — net10.0-windows10.0.26100.0, PACKAGED (MSIX)
    Package.appxmanifest     # never delete; identity + internetClient capability
    App.xaml(.cs)            # composition root (explicit DI), dark-first theme
    MainWindow.xaml(.cs)     # shell: command strip + SelectorBar + floating Now Playing bar, Mica
    Views/  ViewModels/  Strings/en-us/Resources.resw
  OmegaWinUI.Core/           # class library — PLAIN net10.0 (no Windows TFM)
    Models/                  # Song, Album, Artist, Playlist, search/home models (partial)
    Upstream/                # JioSaavnClient (api.php __call), DTOs + mappers,
                             # MediaDecryptor (DES-ECB + ladders), LyricsHelper,
                             # UpstreamJsonContext (STJ source-generated)
tests/
  OmegaWinUI.Core.Tests/     # xUnit, net10.0 — runs on any OS
docs/DESIGN_NOTES.md         # upstream design summary + validation corrections
```

Core targets plain `net10.0` on purpose: HttpClient + System.Text.Json + DES are cross-platform BCL, so the whole data layer is unit-testable without a Windows machine, and no WinRT surface leaks into trim/AOT analysis.

## Build & run

### One-time machine setup

| Tool | Requirement |
|---|---|
| Windows | Developer Mode **enabled** (required for packaged deploy/run) |
| .NET SDK | **10.0** — `winget install Microsoft.DotNet.SDK.10` |
| WinApp CLI | **≥ 0.7.0, released build** — `winget install Microsoft.WinAppCli` |
| Visual Studio 2026 | "Windows application development" workload (WinUI) — VS is the IDE shell; the CLI flow below is the source of truth |
| MSVC C++ tools | "Desktop development with C++" workload — **required for Native AOT publish** (not for JIT iteration) |

### Inner loop (CLI-first)

```powershell
winapp run . --detach --json                                # JIT inner loop (packaged activation — never launch the exe directly)
winapp run . --aot -c Release --arch x64 --detach --json    # AOT checkpoint: publishes the native artifact, then runs it
winapp run --debug-output                                   # crash triage (stowed exceptions)
```

In **Visual Studio 2026**: open `OmegaWinUI.sln`, press **F5** — that should be the same packaged deploy/debug as `winapp run`. If the two ever disagree, the CLI commands win.

### Native AOT publish

AOT is a day-one architectural constraint here, not a release switch (see "AOT posture" below).

```powershell
dotnet publish src\OmegaWinUI\OmegaWinUI.csproj -c Release -p:PublishProfile=win-x64
dotnet publish src\OmegaWinUI\OmegaWinUI.csproj -c Release -p:PublishProfile=win-ARM64
winapp run . --aot -c Release --arch x64 --detach --json   # validate by RUNNING the published artifact
```

> **A JIT run is not AOT evidence.** Validation means running the published native artifact. AOT/trim analyzers also run in normal builds (see below), so violations surface early — fix them; warnings are never suppressed in this repo.

## Version choices — verify on first open in VS 2026

The guidance source (`microsoft/win-dev-skills` v0.7.1) is CLI-first and version-silent about Visual Studio itself, so these pins come from its floors + latest stable NuGet releases at scaffold time. **Each one is a "verify on first open" item** — if VS 2026's WinUI template disagrees, prefer the template and update `Directory.Packages.props`.

| Item | Chosen | Floor / note |
|---|---|---|
| .NET SDK | 10.0 (`global.json`: 10.0.100, rollForward latestFeature) | SDK 8 cannot build a net10.0 TFM — the TFM decides the real floor |
| App TFM | `net10.0-windows10.0.26100.0`, min version `10.0.17763.0` | Template-typical per the skills' workflow example |
| Core / tests TFM | plain `net10.0` | Cross-platform by design |
| `Microsoft.WindowsAppSDK` | **2.5.1** | ≥ 2.1.3 required (below that: XAML compiler `MSB3073` bug) |
| `Microsoft.Windows.SDK.BuildTools` | 10.0.26100.4654 | Matches the 26100 target platform |
| `Microsoft.Windows.SDK.BuildTools.WinUIAnalyzer` | 0.7.1 (`PrivateAssets="all"`) | Added by hand — the CLI does not inject it |
| `CommunityToolkit.Mvvm` | 8.4.2 | ≥ 8.4 required for partial-property `[ObservableProperty]` |
| `Microsoft.Extensions.DependencyInjection` | 10.0.0 | Explicit registrations only |
| `Microsoft.NET.Test.Sdk` / `xunit` / `xunit.runner.visualstudio` | 18.10.1 / 2.9.3 / 3.1.5 | Tests |
| Platforms | **x64, ARM64 only** | Never AnyCPU (0x8007000B) |

### VS 2026 machine-verification checklist (record results here after first session)

From the design doc §2.3 — win-dev-skills contains **no** VS-version guidance, so these are genuinely open:

- [ ] The template-built project opens/builds/debugs in VS 2026 identically to the CLI flow (F5 = packaged deploy).
- [ ] The workload VS 2026 shows for WinUI ("Windows application development" or its 2026 name) — record the exact name shown by the installer.
- [ ] `dotnet --list-sdks` — VS 2026's bundled SDK band vs the `global.json` pin.
- [ ] MSVC C++ tools from the VS 2026 install successfully publish AOT (`winapp run --aot`).

## AOT posture (exactly what is set, and where)

**App project (`src/OmegaWinUI/OmegaWinUI.csproj`)** — the win-dev-skills "day-one" block, so AOT/trim analysis runs during development, not just at publish:

```xml
<PublishAot>true</PublishAot>
<SuppressTrimAnalysisWarnings>false</SuppressTrimAnalysisWarnings>
<TrimmerSingleWarn>false</TrimmerSingleWarn>
<CsWinRTAotWarningLevel>2</CsWinRTAotWarningLevel>
```

`CsWinRTAotOptimizerEnabled` stays at its default (`True`). **Release publish profiles** (`Properties/PublishProfiles/win-x64.pubxml`, `win-ARM64.pubxml`) additionally carry `PublishAot=true`, matching RID/Platform and `SelfContained=true` for folder publishes. F5/JIT debugging is unaffected — `winapp run` without `--aot` runs the JIT build.

**Core (`src/OmegaWinUI.Core/OmegaWinUI.Core.csproj`)**:

```xml
<IsAotCompatible>true</IsAotCompatible>
<EnableTrimAnalyzer>true</EnableTrimAnalyzer>
<EnableAotAnalyzer>true</EnableAotAnalyzer>
<EnableSingleFileAnalyzer>true</EnableSingleFileAnalyzer>
```

**Code rules that follow from this:** all JSON goes through the System.Text.Json **source-generated** `UpstreamJsonContext` (no reflection serialization anywhere); the polymorphic browse-modules payload is walked as `JsonElement` with typed DTOs per item; MVVM uses `[ObservableProperty]` **partial properties** only (the field form is error MVVMTK0045 under AOT); DI registrations are explicit (no assembly scanning); ABI-crossing types are `partial`; no `dynamic`/`Activator`/reflection navigation.

## Data layer (Core)

Direct-upstream JioSaavn client — full design + the live-validation corrections are in [docs/DESIGN_NOTES.md](docs/DESIGN_NOTES.md):

- One endpoint: `GET https://www.jiosaavn.com/api.php?__call=<call>&_format=json&_marker=0&api_version=4&ctx=web6dot0…`, random browser User-Agent per request, **no cookies/auth**, no hosted wrapper anywhere.
- Media URLs: **DES-ECB/PKCS7** decryption (key in the upstream spec) + a synthesised quality ladder (`_12/_48/_96/_160/_320`, `_320` only when the `320kbps` flag allows) + image ladder (50/150/500).
- **Lyrics rule (validated):** call `lyrics.getLyrics` with `lyrics_id` = **the song's own id**; never gate on `song.getDetails`' `has_lyrics` (it lies) — the *search* payload's flag is the accurate one.
- Paging derives from `total`/`list_count` + accumulated counts — upstream's `start` is unreliable (observed negative).
- Radio/suggestions are **broken upstream** (validated): the client implements create/fetch behind an empty-on-error contract and no feature depends on it.

## Status

Phase 0/1 scaffold: shell (SelectorBar navigation across Home/Search/Library/Settings placeholders), floating Now Playing bar placeholder, full Core data layer with xUnit tests. Playback (MediaPlayer + SMTC), persistence (Microsoft.Data.Sqlite — chosen for the persistence phase; EF Core / sqlite-net / Dapper were rejected under AOT), downloads, and detail pages land in later phases per `docs/DESIGN_NOTES.md`.
