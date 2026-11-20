# Debugging and navigation guide

[Overview](overview.md) | [Execution traces](overview.md#end-to-end-investigations)

## Run the correct host

The maintained UI is MAUI, not the terminal application. Commands below are run from the repository root in PowerShell. They are documented entry points, **not commands executed during this documentation task**.

### Windows MAUI

Use the existing **JellyJam: Windows (debug)** configuration in [launch.json](../../.vscode/launch.json). Its prelaunch task is in [tasks.json](../../.vscode/tasks.json).

```powershell
dotnet build .\src\JellyJam.MAUI\JellyJam.MAUI.csproj -c Debug -f net10.0-windows10.0.19041.0
```

For a published build:

```powershell
.\publish.ps1 Windows
& .\artifacts\windows\JellyJam.MAUI.exe
```

[publish.ps1](../../publish.ps1) assumes the repository-local .NET toolchain. The F5 configuration points to Debug output under the Windows target/win-x64 directory; if absent, check the selected target/runtime and actual build result rather than launching an older published executable.

### Android MAUI

The existing launch/task configurations invoke [scripts/android-emulator.ps1](../../scripts/android-emulator.ps1):

```powershell
.\scripts\android-emulator.ps1 -StartOnly
.\scripts\android-emulator.ps1
```

The second command is the repository's build/install/launch workflow, not simply a C# debugger attach. Inspect the script's prerequisites before running it on a shared emulator. Direct x64 emulator build:

```powershell
dotnet build .\src\JellyJam.MAUI\JellyJam.MAUI.csproj -c Debug -f net10.0-android -r android-x64
```

Android ARM64 publishing is `.\publish.ps1 Android`; it is distinct from the x64 emulator build. MAUI/Android workloads and SDK/JDK must exist. [Directory.Build.props](../../Directory.Build.props) uses repository-local Android/JDK directories when present.

### Legacy terminal host

```powershell
dotnet build .\JellyJam.slnx -c Release
dotnet run --project .\src\JellyJam.Console -c Release -- configure
dotnet run --project .\src\JellyJam.Console -c Release -- play
```

The CLI reads `JELLYJAM_HOME` or LocalApplicationData/JellyJam; MAUI uses FileSystem.AppDataDirectory. Changing the CLI variable does not redirect MAUI settings. `NO_COLOR` affects terminal rendering.

## Find configuration without exposing secrets

[Data flow](data-flow.md#persistence-and-restoration) lists active filenames/preferences and restore rules. At a breakpoint inspect FileSystem.AppDataDirectory to identify MAUI's actual directory. Do not assume CLI and MAUI share it.

Inspect station IDs, server identity, version, filters, and source kind. Never dump passwords, token stores, raw PlaybackSource.Uri/AccessToken, HTTP headers, or authenticated exception messages into logs or shared reports. An export is token-free but still may contain private account/server/folder metadata.

Missing configuration and corrupt authoritative configuration intentionally behave differently. Back up user-authored data before any repair; do not delete credentials/caches as a first diagnostic action.

## Breakpoint ladder

| Layer | Useful location | Inspect |
|---|---|---|
| Startup/composition | [MauiProgram.cs](../../src/JellyJam.MAUI/MauiProgram.cs): CreateMauiApp; [App.cs](../../src/JellyJam.MAUI/App.cs): CreateWindow | Target-specific registrations, resources before page resolution, output not yet opened. |
| UI action | [RadioViewModel.cs](../../src/JellyJam.MAUI/ViewModels/RadioViewModel.cs): TogglePowerAsync; [LibraryViewModel.cs](../../src/JellyJam.MAUI/ViewModels/LibraryViewModel.cs): PlayAsync/Next/Stop | Busy state, selected provider, ControlsRadio versus displayed mode. |
| Owner transition | [PlayerShell.cs](../../src/JellyJam.Core/Playback/PlayerShell.cs): PlayLibraryAsync/StartRadioAsync/OnRadioChanged | Radio.IsOn, Library.IsActive, callback/publication interleaving. |
| Radio | [RadioTuner.cs](../../src/JellyJam.Core/Radio/RadioTuner.cs): Refresh/SyncFeeds; [StationFeed.cs](../../src/JellyJam.Core/Radio/StationFeed.cs): AirAsync/BroadcastAsync | Latest dial, neighborhood, cached programme, readiness/gain/failure. |
| Library sequence | [PlaybackCoordinator.cs](../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs): Handle/Start/RunTrackAsync | Message action/generation, worker chain, load token, selector pending/history. |
| Metadata | [LibraryBrowser.cs](../../src/JellyJam.Core/Library/LibraryBrowser.cs): Navigate/ReadPageAsync/Apply; [LibraryIndex.cs](../../src/JellyJam.Core/Library/LibraryIndex.cs): RunAsync | Provider, parent, epoch, captured listIndex, progress, saved snapshot. |
| Integration | [JellyfinClient.cs](../../src/JellyJam.Core/Jellyfin/JellyfinClient.cs): ResolveAsync; [StationSources.cs](../../src/JellyJam.Core/Radio/StationSources.cs): For | Kind/account routing and metadata response status, not credential values. |
| Speaker | [NAudioRadioChannel.cs](../../src/JellyJam.Infrastructure.Windows/Audio/NAudioRadioChannel.cs) or [AndroidRadioChannel.cs](../../src/JellyJam.MAUI/Platforms/Android/AndroidRadioChannel.cs): DecodeAsync/Read | Ready/Completed tasks, buffer, samples consumed, gain. |
| Persistence | [RadioController.cs](../../src/JellyJam.Core/Radio/RadioController.cs): EditAsync; [JsonStationStore.cs](../../src/JellyJam.Core/Configuration/JsonStationStore.cs): SaveAsync | Candidate versus current, whether Stop has run, temp replacement and compensation. |

## Follow asynchronous and event-driven paths

- Use task/parallel-stack views and conditional breakpoints on station/provider/generation IDs. Do not pause every audio sample callback in a running real-device session.
- A RelayCommand's public name is generated; break in the attributed backing method.
- When a result disappears, compare request identity: browser epoch, coordinator generation, ArtworkSlot key, theme artworkVersion, Plex attempt number.
- Core Changed can be emitted off-thread. Follow the view model's MainThread callback to see when it actually renders.
- A completed Play command means the session was established, not that channel Ready or playback completion has occurred.
- A cancelled waiter is not necessarily a cancelled shared fetch. Catalog, artwork, podcast, and index work have different lifetimes.
- When shutdown returns slowly, distinguish awaited coordinator/tuner work from bounded native cleanup and unjoined shelf/index tasks.

## Troubleshooting

| Symptom | Where to Investigate | Important Components | Relevant Execution Path |
|---|---|---|---|
| Startup says stations invalid | Load result and loaded flag, not audio | JsonStationStore, RadioController, RadioViewModel | [Launch/restore](investigations/launch-and-restore.md) |
| Radio plays static at configured station | Programme/ready/failure before gain | StationFeed, RadioTimeline, TuningMix | [Radio tuning](investigations/radio-tuning.md) |
| Playing state but silence | Gain/buffer, then native feeder | Platform channel, RadioMixer, AndroidRadioOutput.Feed | [Failure](investigations/errors-and-lifecycle.md) |
| Two owners active | beforeStart/publication window and direct Radio callers | PlayerShell, LibraryPlayer, RadioController | [Owner switching](investigations/library-playback.md) |
| Android Stop does not stop Library | Notification and MediaSession routing | RadioPlaybackService.Controls/HandleCommand | [Native triggers](investigations/errors-and-lifecycle.md#native-triggers-are-distinct) |
| Index progress never yields new data | Save failure/repeated pages/token failure | LibraryIndex, JsonLibraryIndexStore, JellyfinClient | [Browse/search](investigations/library-browse-search.md) |
| Search empty but remote browsing works | Missing index versus query cancellation | LibrarySearchViewModel, LibrarySearch | [Browse/search](investigations/library-browse-search.md) |
| Old album results overwrite a newer page | Epoch and return state | LibraryBrowser.Apply/listIndex/playbackReturn | [Browse/search](investigations/library-browse-search.md) |
| Wrong song after Previous/rapid Next | Pending versus confirmed history, stale generation | TrackSelector, PlaybackCoordinator | [Transport](investigations/library-playback.md) |
| Missing or stale cover/color | Identify browser versus now-playing loader | LibraryBrowser, ArtworkSlot, NowPlayingArtwork, ThemeService | [Artwork](investigations/artwork-and-shelves.md) |
| Favorite disappears after close | Deferred write and lack of flush | LibraryShelves, JsonLibraryShelfStore | [Shelves](investigations/artwork-and-shelves.md) |
| Import fails after music stops | Failure point relative to Stop/Save | RadioController.EditAsync, DataTransfer, credentials | [Accounts/import](investigations/accounts-and-stations.md) |
| Windows cannot grow to 824 | Current maximum is 412, old expectation is stale | DesktopLayoutMetrics, PhoneWindow | [Platform lifecycle](subsystems/platform-lifecycle.md) |
| Ads appear after consent update failure | Eligibility and actual request boundary | AndroidAdService, AdBannerViewHandler | [Ads behavior](subsystems/platform-lifecycle.md#ads-visibility-is-not-request-authorization) |

## Diagnostics

Windows TraceLoggerBridge forwards sanitized Trace into ILogger/LogTail. SafeTask writes type/caller to stderr; some Core paths use Trace, stderr, or intentionally suppress errors. Android LogTail/native logs are another surface. A clean visible log panel is not evidence that every background operation succeeded.

Do not add raw exception/HTTP dumps to compensate for missing logs. ErrorMessageMapper exists to map connection/auth/device/format failures into user-facing messages.

## Existing test commands and limits

The projects use xUnit with Microsoft.NET.Test.Sdk/VSTest. Start with a focused suite:

```powershell
dotnet test .\tests\JellyJam.Core.Tests\JellyJam.Core.Tests.csproj -c Release --filter "FullyQualifiedName~PlayerShellTests|FullyQualifiedName~LibraryPlayerTests|FullyQualifiedName~PlaybackCoordinatorTests"
dotnet test .\tests\JellyJam.MAUI.Tests\JellyJam.MAUI.Tests.csproj -c Release
```

Broader repository commands:

```powershell
dotnet test .\JellyJam.slnx -c Release --filter "Category!=LiveAudio"
dotnet test .\JellyJam.slnx -c Release --filter "Category=LiveAudio"
```

The latter plays audio. The former can include environment-dependent integration/E2E tests; inspect skip results. MAUI.Tests is outside the main solution and only links selected files. A real MAUI build is a separate validation step.

Windows E2E discovers the published executable or `JELLYJAM_WINDOWS_EXE`. Android tests use [AndroidEmulatorFixture.cs](../../tests/JellyJam.TestSupport/AndroidEmulatorFixture.cs) and can skip if prerequisites are absent. The supplied review records earlier selected-test results; they were not rerun here, and neither those results nor Mermaid validation constitute live playback verification.
