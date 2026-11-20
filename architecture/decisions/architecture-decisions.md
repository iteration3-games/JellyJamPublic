# Architecture decision register

[Overview](../overview.md) | [Dependencies](../dependency-graph.md)

This is a reverse-engineered register, **not a set of newly accepted ADRs**. Explicit evidence means current code comments/contracts or repository documentation state the intent; it does not establish who decided it or when. Alternatives below are comparisons, not documented historical evaluations or approved changes. Remediation belongs in the supplied [architecture review](../../../.github/generated/architecture-review.md).

## D01 - Platform-neutral contracts in one substantial Core

- **Classification/confidence:** explicitly reflected in project references and contracts; high confidence in implementation, historical motivation not established.
- **Sources:** [Core project](../../../src/JellyJam.Core/JellyJam.Core.csproj), [MAUI project](../../../src/JellyJam.MAUI/JellyJam.MAUI.csproj), [audio contract](../../../src/JellyJam.Core/Abstractions/IRadioAudioOutput.cs), [station source contract](../../../src/JellyJam.Core/Abstractions/IStationSource.cs).
- **Problem addressed:** multiple presentation/platform hosts need common models and behavior.
- **Approach:** Core contains contracts, algorithms, orchestration, HTTP integrations, and JSON stores; platform projects implement native capabilities. MAUI references Windows infrastructure only for its Windows target.
- **Advantages:** domain behavior and fake-based tests are reusable; no MAUI/native API dependency in Core.
- **Disadvantages:** Core is not domain-only; every consumer takes the Jellyfin SDK and integration/storage surface.
- **Alternatives:** separate domain/application/integration assemblies, or duplicated platform logic. Neither alternative is the current architecture.

## D02 - Separate broadcast and selected-playback orchestration

- **Classification/confidence:** strongly inferred architectural choice from two deliberately different models; high confidence in separation, medium in rationale beyond comments.
- **Sources:** [RadioTimeline.cs](../../../src/JellyJam.Core/Radio/RadioTimeline.cs), [RadioTuner.cs](../../../src/JellyJam.Core/Radio/RadioTuner.cs), [LibraryPlayer.cs](../../../src/JellyJam.Core/Library/LibraryPlayer.cs), [PlaybackCoordinator.cs](../../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs).
- **Problem addressed:** joining a continuous station and traversing a user selection have different semantics.
- **Approach:** RadioTuner/StationFeed use clock-based programmes; LibraryPlayer/coordinator use TrackSelector/history. Both reuse output/channel implementations.
- **Advantages:** no need to force live tuning into playlist history; shared decoding remains reusable.
- **Disadvantages:** ownership, transport controls, notification state, and lifecycle must integrate two engines.
- **Alternatives:** one generalized player state machine, or two entirely separate audio stacks. Neither is wired today.

## D03 - Deterministic virtual broadcast without saved station cursor

- **Classification/confidence:** explicit RadioTimeline comments and implementation; high.
- **Sources:** [RadioTimeline.cs](../../../src/JellyJam.Core/Radio/RadioTimeline.cs), [ShuffleOrder.cs](../../../src/JellyJam.Core/Radio/ShuffleOrder.cs), [RadioTimelineTests.cs](../../../tests/JellyJam.Core.Tests/RadioTimelineTests.cs).
- **Problem addressed:** tuning should join a station as if it has always been broadcasting, including after restart.
- **Approach:** validate/sort timed tracks; compute cycle from fixed epoch; derive shuffle seed from version/station/cycle; calculate exact track interval and seek.
- **Advantages:** no continuously persisted cursor; reproducible schedule when clocks and metadata agree.
- **Disadvantages:** missing/wrong durations prevent or distort scheduling; metadata/identity/seed changes can retime a station; no distributed clock synchronization.
- **Alternatives:** persist per-station progress or ask a server for synchronized broadcast position. Not used.

## D04 - Serialize selected playback, reject stale worker events

- **Classification/confidence:** explicit comments and implementation; high.
- **Sources:** [PlaybackCoordinator.cs](../../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs), [TrackSelector.cs](../../../src/JellyJam.Core/Playback/TrackSelector.cs).
- **Problem addressed:** rapid commands and cancelled loads must not advance the wrong track or leave false history.
- **Approach:** single-reader message channel, generations, previous-worker chaining, pending history confirmed only after Started.
- **Advantages:** clear command ordering and stale-result protection; pure selector is deterministic/testable with seeded randomness.
- **Disadvantages:** diagnosing work requires tracking messages and worker generations; some retry-status writes occur in workers; this does not serialize Radio/Library ownership globally.
- **Alternatives:** locking every callback or a fully synchronous playback loop. Not current approaches.

## D05 - Separate user-visible mode from sounding owner

- **Classification/confidence:** explicitly documented by PlayerShell; high.
- **Sources:** [PlayerShell.cs](../../../src/JellyJam.Core/Playback/PlayerShell.cs), [RadioViewModel.cs](../../../src/JellyJam.MAUI/ViewModels/RadioViewModel.cs), [PlayerShellTests.cs](../../../tests/JellyJam.Core.Tests/PlayerShellTests.cs).
- **Problem addressed:** browsing another screen should not interrupt music.
- **Approach:** SetMode publishes view state only; starting playback performs ordered owner handoffs; last Library request supports replay.
- **Advantages:** user can browse while Radio or Library keeps playing; mode is a simple preference.
- **Disadvantages:** UI controls must distinguish mode from owner; direct platform starts and non-atomic handoffs can violate intended single-owner behavior.
- **Alternatives:** stop on every mode change, or route all playback through a single global arbiter. Neither describes today's implementation.

## D06 - Last complete index remains readable during refresh

- **Classification/confidence:** explicit LibraryIndex/store comments and source; high.
- **Sources:** [LibraryIndex.cs](../../../src/JellyJam.Core/Library/LibraryIndex.cs), [JsonLibraryIndexStore.cs](../../../src/JellyJam.Core/Configuration/JsonLibraryIndexStore.cs), [LibraryBrowser.cs](../../../src/JellyJam.Core/Library/LibraryBrowser.cs).
- **Problem addressed:** large-library scans should not make previously indexed metadata unusable or mix pages from different scans.
- **Approach:** load account-scoped saved snapshot, build replacement separately, persist before publication, pin browser lists to one snapshot.
- **Advantages:** local browsing/search can remain useful during refresh/failure; coherent pagination; no partial index persisted as complete.
- **Disadvantages:** old and replacement metadata coexist in memory; fresh metadata is delayed until full success; account/store identity is Jellyfin-shaped.
- **Alternatives:** query remote data for every action, incrementally mutate the visible index, or use a local database. Not implemented here.

## D07 - JSON settings and OS-protected credentials

- **Classification/confidence:** explicit store/model comments and implementation; high.
- **Sources:** [JsonStationStore.cs](../../../src/JellyJam.Core/Configuration/JsonStationStore.cs), [RadioLineup.cs](../../../src/JellyJam.Core/Radio/RadioLineup.cs), [WindowsServerCredentialStore.cs](../../../src/JellyJam.Infrastructure.Windows/Security/WindowsServerCredentialStore.cs), [SecureServerCredentialStore.cs](../../../src/JellyJam.MAUI/Services/SecureServerCredentialStore.cs).
- **Problem addressed:** keep local configuration portable/inspectable without storing access secrets as ordinary JSON.
- **Approach:** separate lineup from protected token store; replace individual files using temporary writes; export lineup metadata, strip Plex endpoint/consent.
- **Advantages:** straightforward file format and testing; credentials protected by OS facilities; configuration export need not include tokens.
- **Disadvantages:** no atomic transaction across stores, compensation complexity, export still contains potentially private metadata, different corruption policies.
- **Alternatives:** one plaintext file, one encrypted settings blob, or transactional database plus key store. None is the current design.

## D08 - Bounded streaming and deferred native cleanup

- **Classification/confidence:** explicit audio contracts/comments; high.
- **Sources:** [AudioReadAhead.cs](../../../src/JellyJam.Core/Audio/AudioReadAhead.cs), [DecoderWorker.cs](../../../src/JellyJam.Core/Audio/DecoderWorker.cs), [AndroidRadioChannel.cs](../../../src/JellyJam.MAUI/Platforms/Android/AndroidRadioChannel.cs), [NAudioRadioChannel.cs](../../../src/JellyJam.Infrastructure.Windows/Audio/NAudioRadioChannel.cs).
- **Problem addressed:** avoid whole-track downloads and UI hangs when native decode calls cannot be interrupted immediately.
- **Approach:** bounded compressed/decoded buffers; decoder worker owns cleanup; Dispose waits a bounded grace period before letting cleanup finish later.
- **Advantages:** bounded routine buffering and responsive stop even when native calls stall; no concurrent disposal during native reads.
- **Disadvantages:** return from disposal is not always proof that every native resource is already gone; retained workers complicate lifecycle diagnosis.
- **Alternatives:** unbounded buffering, blocking shutdown indefinitely, or forcefully interrupting native resources. Not implemented policies.

## D09 - Linked-source MAUI tests beside real platform builds

- **Classification/confidence:** strongly inferred testing tradeoff; high in mechanics, medium in rationale.
- **Sources:** [MAUI.Tests project](../../../tests/JellyJam.MAUI.Tests/JellyJam.MAUI.Tests.csproj), [main solution](../../../JellyJam.slnx), [MAUI solution](../../../JellyJam.MAUI.slnx).
- **Problem addressed:** exercise shared presentation helpers and inspect XAML without always running a native app.
- **Approach:** plain net10.0 test project links selected production helpers and embeds markup.
- **Advantages:** fast tests independent of platform UI execution.
- **Disadvantages:** main-solution success misses this project; linked-source/structural tests do not prove full app compilation, DI, native lifecycle, or rendered behavior.
- **Alternatives:** only native UI tests, or a separately factored presentation assembly tested directly. Not the current test structure.

## D10 - Partial owner/lifecycle routing is an apparent implementation gap

- **Classification/confidence:** apparent accidental/incomplete integration, **not an intentional accepted design**; high confidence in code paths, unknown historical cause.
- **Sources:** [RadioPlaybackService.cs](../../../src/JellyJam.MAUI/Platforms/Android/RadioPlaybackService.cs), [PlaybackServiceLifetime.cs](../../../src/JellyJam.MAUI/Services/PlaybackServiceLifetime.cs), [App.cs](../../../src/JellyJam.MAUI/App.cs).
- **Problem it currently attempts to address:** keep Android playback alive and provide system controls; stop resources when windows/services end.
- **Approach observed:** lifetime/noisy/destruction consider both players, ordinary controls and metadata use Radio; Window.Destroying explicitly disposes only Radio after service Stop.
- **Advantages:** reuse of existing Radio notification path and some shared lifetime coverage; this does not establish a deliberate reason to leave Library routing incomplete.
- **Disadvantages:** wrong Library system controls/metadata, no explicit complete join/flush shutdown sequence.
- **Alternatives:** owner-aware control routing and explicit shared shutdown orchestration are review proposals, not present guarantees.

## Maintaining this register

When behavior changes, link the implementing code and verification, label whether intent is explicit, and update the relevant runtime trace. Do not infer a historical rationale from a class name, a passing structural test, or an unimplemented review recommendation.
