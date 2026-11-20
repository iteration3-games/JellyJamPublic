# Feature development using the current architecture

[Overview](overview.md) | [Dependencies](dependency-graph.md) | [Debugging/tests](debugging-guide.md)

This guide describes today's extension conventions. It does not adopt the review's proposed redesigns or claim that a generic plugin system already exists.

## First identify the owning path

| Change | Starting owner |
|---|---|
| Dial/broadcast/station policy | RadioController, RadioTuner, StationFeed, RadioTimeline |
| Selected sequence/track operation | LibraryPlayer, PlaybackCoordinator, TrackSelector |
| Library hierarchy/search | LibraryBrowser, LibraryIndexSnapshot, LibrarySearch |
| Cross-owner UI operation | PlayerShell plus host/platform call sites |
| Screen/form-only state | Relevant MAUI view model/page |
| Decode/device/focus | Platform output/channel and Core audio contract |
| Durable setting/account edit | RadioLineup/RadioController and its store |

Do not implement the same track-selection policy in a view model and a native media callback. Also do not assume the current shell has a global transition lock: see [playback](subsystems/playback.md).

## 1. Add a UI feature

Example shape: a Library action that opens a related artist/album, without changing playback.

1. **Location:** add the command backing method and observable presentation state in [LibraryViewModel.cs](../../src/JellyJam.MAUI/ViewModels/LibraryViewModel.cs), binding in [LibraryView.xaml](../../src/JellyJam.MAUI/Views/LibraryView.xaml).
2. **Reuse:** existing LibraryActions/LibraryNode relationship data and Browser.ShowAsync/OpenAsync; ExecuteAsync for busy/error handling and WaitForIndexAsync for the existing first-index UX.
3. **Modify:** the relevant visibility/can-execute projection, command parameter, and navigation/back behavior. Keep generated RelayCommand naming consistent.
4. **DI:** no new service is required for a command on an existing singleton VM. A new page follows transient registration in [MauiProgram.cs](../../src/JellyJam.MAUI/MauiProgram.cs).
5. **Flow:** UI command -> Core browser operation -> Changed -> MainThread projection. Do not mutate Browser.State or playback snapshots.
6. **Test:** browser behavior in [LibraryBrowserTests.cs](../../tests/JellyJam.Core.Tests/LibraryBrowserTests.cs); structural binding/helper checks in [MAUI.Tests](../../tests/JellyJam.MAUI.Tests/JellyJam.MAUI.Tests.csproj); build/render the actual affected MAUI target.

## 2. Add an application operation

Example shape: an additional selected-playback operation, if its behavior has been specified.

1. **Location:** LibraryPlayer for session facade, [PlaybackCoordinator.cs](../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs) for serialized track commands, [PlayerAction.cs](../../src/JellyJam.Core/Models/PlayerAction.cs) if a new command is required.
2. **Reuse:** coordinator channel, generation checks, worker chaining, source release, and TrackSelector for pure sequence changes.
3. **Modify:** action dispatch, snapshot semantics, appropriate VM command, and every applicable native/terminal route. A Radio operation instead belongs on RadioController/tuner, not this channel.
4. **DI:** existing controller/player registrations usually remain; dependencies must be passed through constructors/factories in the relevant host.
5. **Flow:** command -> owner facade -> serialized action -> stream/selector -> snapshot -> UI. Define natural completion versus explicit action before touching the worker.
6. **Test:** deterministic sequence/history, rapid superseding commands, cancellation, output cleanup, fatal versus nonfatal failures, and cross-owner effects using [PlaybackCoordinatorTests.cs](../../tests/JellyJam.Core.Tests/PlaybackCoordinatorTests.cs) and [PlayerShellTests.cs](../../tests/JellyJam.Core.Tests/PlayerShellTests.cs).

Pause/seek are not currently implemented operations. A start-position property is not a shortcut to a complete transport feature.

## 3. Add a domain concept

Example shape: a source-dependent station option.

1. **Location:** [RadioStation.cs](../../src/JellyJam.Core/Radio/RadioStation.cs) or the relevant Core model; pure calculation under Radio/Library/Audio.
2. **Reuse:** immutable-style records, WithStation validation, TrackFilter normalization, station source rules.
3. **Modify:** [RadioLineup.cs](../../src/JellyJam.Core/Radio/RadioLineup.cs) validation, consumers, editor helpers such as [StationSourceChoices.cs](../../src/JellyJam.MAUI/ViewModels/StationSourceChoices.cs), and serialization/defaults.
4. **DI:** pure value types/functions require no registration. Inject a service only when the concept has an existing service-style responsibility.
5. **Flow:** form -> model -> candidate validation -> persisted lineup -> new tuner/source snapshot after restart. Existing edits take Radio off air.
6. **Test:** defaults/invalid values, old-file loading, round trips/import/export, and deterministic timeline impact in [RadioTimelineTests.cs](../../tests/JellyJam.Core.Tests/RadioTimelineTests.cs) and related source/controller tests.

Changing station IDs, shuffle seeds, epoch, duration interpretation, or ordering can retime a broadcast; test exact positions, not only that some track plays.

## 4. Add an external integration

Example shape: another station source.

1. **Location:** provider-neutral contracts in [Core Abstractions](../../src/JellyJam.Core/Abstractions), HTTP/provider translation beside existing [Core source adapters](../../src/JellyJam.Core), native access under platform folders.
2. **Reuse:** IStationSource for programme/resolve/release, Track/PlaybackSource as transport-neutral vocabulary, IMediaProbe for missing length, provider MP3 transport when needed.
3. **Modify:** StationKind/model validation, RadioController source composition, MAUI source/editor rules, account form if needed, and CLI routing only if that host supports it. Unsupported hosts need honest behavior.
4. **DI:** add provider dependencies and the correct StationKind dictionary entry in MauiProgram/controller composition; the singleton IStationSource registration by itself is currently device music, not the entire router.
5. **Flow:** feed -> router -> provider metadata/resolve -> channel -> release. Auth material stays in protected stores; source URLs/tokens are not persisted as Track metadata.
6. **Test:** fake HTTP requests, stable IDs/durations, positive and empty programmes, timeout/cancellation, credential/origin behavior, and source release. Use [StationSourceTests.cs](../../tests/JellyJam.Core.Tests/StationSourceTests.cs) and [PlexStationSourceTests.cs](../../tests/JellyJam.Core.Tests/PlexStationSourceTests.cs) as concrete patterns.

Follow existing HTTPS/HTTP confirmation and bounded retry policies. No adapter should bypass certificate validation or download whole audio files.

## 5. Add a browsable data provider

This is more than adding a Radio source.

1. **Location:** implement [ILibraryClient.cs](../../src/JellyJam.Core/Abstractions/ILibraryClient.cs) in the provider adapter: BrowseAsync, GetScopeTracksAsync, resolve/release; optional artwork/recent-album methods as supported.
2. **Reuse:** LibraryNode/LibraryPage, browser pagination/epochs, complete metadata snapshots, and LibraryPlayer's selection flow.
3. **Modify:** [LibraryCatalog.cs](../../src/JellyJam.Core/Library/LibraryCatalog.cs) currently enumerates RadioLineup.Servers and constructs clients from PlayerConfiguration. [LibraryIndex.cs](../../src/JellyJam.Core/Library/LibraryIndex.cs) and its store key on RadioServer. Those exact discovery/factory/identity surfaces must also recognize a second provider; no existing registration makes that automatic.
4. **DI:** update the catalog factory and relevant provider services in MauiProgram. IServerProvider only supplies a MAUI account form; it does not register a Library provider.
5. **Flow:** account discovery -> catalog client -> index/browser -> selected tracks -> LibraryPlayer. Keep remote provider IDs, local account IDs, and node IDs distinct.
6. **Test:** account isolation, paging progress/deduplication, old snapshot while refreshing, index save/load identity, browse hierarchy, removed account behavior, and playback through the existing adapter.

This describes the currently necessary touch points, not an approved generic-provider refactor.

## 6. Add persistence

1. **Location:** a Core model/store contract where applicable, implementation in [Configuration](../../src/JellyJam.Core/Configuration), or platform credential adapter for secrets.
2. **Reuse:** existing JSON temporary-file replacement, cancellation, version validation, and constructor-supplied directory.
3. **Modify:** authoritative writer/reader, defaults/migration, export behavior, and current state owner. Identify whether the data is user-authored or disposable cache.
4. **DI:** register with FileSystem.AppDataDirectory for MAUI, CLI's selected directory for Console. Do not assume those hosts share state.
5. **Flow:** awaited authoritative save then publication, or explicitly deferred cache/shelf write according to the existing subsystem. File replacement does not make multiple stores transactional.
6. **Test:** missing/corrupt files, save failure before publication, cancellation, old versions, identity separation, and cleanup/compensation. [LibraryIndexPersistenceTests.cs](../../tests/JellyJam.Core.Tests/LibraryIndexPersistenceTests.cs) and [PlexControllerTests.cs](../../tests/JellyJam.Core.Tests/PlexControllerTests.cs) cover useful cases.

Current shelves have no flush contract; do not silently promise shutdown durability for another deferred writer.

## 7. Add tests

- Put pure Core logic/coordination tests in [Core.Tests](../../tests/JellyJam.Core.Tests). Use fixed clocks/seeded Random and synthetic clients/channels; [PlayerShellTests.cs](../../tests/JellyJam.Core.Tests/PlayerShellTests.cs) shows a complete fake rig and explicit disposal.
- Put NAudio/DPAPI behavior in [Infrastructure.Windows.Tests](../../tests/JellyJam.Infrastructure.Windows.Tests). Mark actual hardware needs LiveAudio.
- Add MAUI helper source links/resources to [MAUI.Tests.csproj](../../tests/JellyJam.MAUI.Tests/JellyJam.MAUI.Tests.csproj) only for the existing linked-source testing style. Such tests do not prove complete app composition.
- Use [TestSupport](../../tests/JellyJam.TestSupport) for existing shared fixtures. Keep credentials synthetic and do not require a user's live account for ordinary tests.
- Run the smallest relevant filtered suite, then the affected real platform build when production code changes. Use [debugging-guide commands](debugging-guide.md#existing-test-commands-and-limits).
- For behavior shifts, verify observable outcome and exact boundary: e.g. no two owners active under a controlled interleaving, no stale page publication, bounded bytes read, or no false history entry. A binding-name assertion alone is not a transport test.

## Keep documentation aligned

Update the owning subsystem and its end-to-end trace when call order, state ownership, native routing, or persistence changes. Update this [decision register](decisions/architecture-decisions.md) only with explicit or clearly labeled inferred choices; do not turn review recommendations into historical facts.
