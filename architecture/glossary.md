# Developer glossary

[Overview](overview.md)

## Domain and playback

| Term | Meaning in JellyJam |
|---|---|
| Station | A [RadioStation](../../src/JellyJam.Core/Radio/RadioStation.cs): local identity, frequency, call sign, source-specific configuration, effect, and shuffle. Not an audio player instance. |
| Lineup | [RadioLineup](../../src/JellyJam.Core/Radio/RadioLineup.cs): saved accounts, Plex servers, stations, schema version, and last frequency. |
| RadioServer | A local Jellyfin account record in [RadioStation.cs](../../src/JellyJam.Core/Radio/RadioStation.cs). Its local GUID differs from the remote Jellyfin user ID; token is stored elsewhere. |
| PlexServer / MachineId | Local connection record plus Plex's stable server-machine identity in [RadioStation.cs](../../src/JellyJam.Core/Radio/RadioStation.cs); not a display-name lookup. |
| StationKind | Jellyfin, Podcast, Device, Internet, or Plex discriminator used by [StationSources](../../src/JellyJam.Core/Radio/StationSources.cs). |
| StreamProvider | RadioBrowser, SomaFm, or Direct within an internet station; see [RadioBrowserModels.cs](../../src/JellyJam.Core/Internet/RadioBrowserModels.cs). This is not a Library account. |
| Programme | A finite list of timed tracks aired by a [StationFeed](../../src/JellyJam.Core/Radio/StationFeed.cs), not an editable user playlist. |
| Track | Shared metadata in [Track.cs](../../src/JellyJam.Core/Models/Track.cs): identity/title/artist/album/order/duration and optional artwork URL. Not a stream handle. |
| TrackFilter | Normalized genre/artist/tag lists and cache key in [RadioStation.cs](../../src/JellyJam.Core/Radio/RadioStation.cs). Capabilities differ by source. |
| Timeline / epoch | [RadioTimeline](../../src/JellyJam.Core/Radio/RadioTimeline.cs) computes broadcast position from clock, identity, metadata, and shuffle; epoch/version are part of deterministic scheduling. |
| PlaybackPosition | Radio timeline result: track, seek position, start/end times, and global track index; defined in [RadioTimeline.cs](../../src/JellyJam.Core/Radio/RadioTimeline.cs). |
| Neighborhood | Nearest station plus adjacent stations retained by [TuningMix.Neighbourhood](../../src/JellyJam.Core/Radio/TuningMix.cs). Up to three feeds, not necessarily three audible tracks. |
| Locked station | Dial within the lock radius in [TuningMix](../../src/JellyJam.Core/Radio/TuningMix.cs); it may still be connecting or failed. |
| RadioSignal | None/Connecting/Playing/Failed in [RadioTuner.cs](../../src/JellyJam.Core/Radio/RadioTuner.cs), independent of power state. |
| IsOn | [RadioController](../../src/JellyJam.Core/Radio/RadioController.cs) has a tuner instance. Not proof of speaker output. |
| Displayed mode | [PlayerShell.Mode](../../src/JellyJam.Core/Playback/PlayerShell.cs): Radio or Library view. Does not determine audible owner. |
| Active Library | [LibrarySnapshot.IsActive](../../src/JellyJam.Core/Library/LibraryPlayer.cs): Loading or Playing. No Paused status exists. |
| Replay | [PlayerShell.ReplayLibraryAsync](../../src/JellyJam.Core/Playback/PlayerShell.cs) reuses last in-memory request and starts again; not pause/resume. |
| RandomMode / RepeatMode | Selected-playback scope in [TrackSelector.cs](../../src/JellyJam.Core/Playback/TrackSelector.cs), distinct from station ShuffleAlgorithm. |
| Pending history | Selector entered a track but has not received Started confirmation; [TrackSelector.DiscardPending](../../src/JellyJam.Core/Playback/TrackSelector.cs) prevents cancelled loads becoming false history. |
| Generation | [PlaybackCoordinator](../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs) identity attached to worker events; obsolete generations cannot advance a newer selection. |

## Library and state

| Term | Meaning |
|---|---|
| Provider | [LibraryProvider](../../src/JellyJam.Core/Library/LibraryModels.cs), currently a connected Jellyfin account exposed by LibraryCatalog. Not every StationKind is browsable. |
| Node / section / scope | [LibraryModels.cs](../../src/JellyJam.Core/Library/LibraryModels.cs): artist/album/genre/song metadata; section is a top-level category; scope narrows browsing or selected tracks. |
| LibraryPage | Nodes plus HasMore from [ILibraryClient.BrowseAsync](../../src/JellyJam.Core/Abstractions/ILibraryClient.cs). Not a MAUI page. |
| Index snapshot / back buffer | Last complete metadata scan in [LibraryIndex.cs](../../src/JellyJam.Core/Library/LibraryIndex.cs), retained while a replacement is built and persisted. No audio cached. |
| Epoch | [LibraryBrowser](../../src/JellyJam.Core/Library/LibraryBrowser.cs) request counter used to reject old navigation results; unrelated to RadioTimeline's date epoch. |
| listIndex | Browser's captured [LibraryIndexSnapshot](../../src/JellyJam.Core/Library/LibraryIndexSnapshot.cs) for consistent multi-page browsing. |
| Now Playing | Browser view with a return state, managed by [LibraryBrowser.ShowNowPlaying](../../src/JellyJam.Core/Library/LibraryBrowser.cs); not another playback engine. |
| Shelf | Local favorite/recent metadata in [LibraryShelves](../../src/JellyJam.Core/Library/LibraryShelves.cs), projected into carousels by LibraryHomeViewModel. Not a remote playlist. |
| ArtworkSlot | [UI-thread picture owner](../../src/JellyJam.MAUI/ViewModels/ArtworkSlot.cs) that cancels old waiters/rejects stale picture keys. |
| Snapshot | A read-only-style published view of state; backing collections still require ownership discipline. See [data flow](data-flow.md). |

## Audio, platform, and patterns

| Term | Meaning |
|---|---|
| PlaybackSource | [Transient audio description](../../src/JellyJam.Core/Models/PlaybackSource.cs), potentially containing credentials/session/URI/start offset/provider frames. Never ordinary settings. |
| StreamFormat.Media | Unauthenticated local/public media handled by native decoders; unlike Jellyfin's negotiated MP3 path. See [PlaybackSource.cs](../../src/JellyJam.Core/Models/PlaybackSource.cs). |
| Channel | [IRadioChannel](../../src/JellyJam.Core/Abstractions/IRadioAudioOutput.cs): one decoded/buffered input with gain, Ready/Completed, position, disposal. |
| Ready versus Completed | Ready permits playback start; Completed represents consumed end/error. Neither is the same as constructing a channel. |
| Mixer / PCM | [RadioMixer](../../src/JellyJam.Core/Audio/RadioMixer.cs) combines 44.1 kHz stereo float samples (decoded PCM) with effects/static/tone. |
| Read-ahead | [AudioReadAhead](../../src/JellyJam.Core/Audio/AudioReadAhead.cs): duration-bounded compressed memory reserve across scheduled tracks. Not offline download. |
| WASAPI / NAudio | Windows audio API and wrapper used by [NAudioRadioOutput](../../src/JellyJam.Infrastructure.Windows/Audio/NAudioRadioOutput.cs). |
| Media Foundation | Windows media decoder/prober used for supported files/public media; [MediaFoundationProbe.cs](../../src/JellyJam.Infrastructure.Windows/Audio/MediaFoundationProbe.cs). |
| MediaCodec / MediaExtractor / AudioTrack | Android decoding, media extraction, and PCM speaker facilities in [AndroidRadioChannel.cs](../../src/JellyJam.MAUI/Platforms/Android/AndroidRadioChannel.cs) and [AndroidRadioOutput.cs](../../src/JellyJam.MAUI/Platforms/Android/AndroidRadioOutput.cs). |
| Foreground playback service | [RadioPlaybackService](../../src/JellyJam.MAUI/Platforms/Android/RadioPlaybackService.cs) supplies Android notification/lifetime/media-session integration; not the decoder thread. |
| DPAPI / SecureStorage | Platform-protected [Windows](../../src/JellyJam.Infrastructure.Windows/Security/WindowsServerCredentialStore.cs) / [Android](../../src/JellyJam.MAUI/Services/SecureServerCredentialStore.cs) credential persistence. |
| UMP | Google's consent-information/form SDK used by [AndroidAdService](../../src/JellyJam.MAUI/Platforms/Android/AndroidAdService.cs); current implementation limitations are in [platform lifecycle](subsystems/platform-lifecycle.md). |
| MVVM / RelayCommand | MAUI binding architecture and CommunityToolkit generated commands/properties in [ViewModels](../../src/JellyJam.MAUI/ViewModels). |
| Composition root | [MauiProgram.CreateMauiApp](../../src/JellyJam.MAUI/MauiProgram.cs) or [Console Program](../../src/JellyJam.Console/Program.cs), where interfaces/factories are wired to implementations. |
| Single-reader channel | Serialized message consumption used differently by [PlaybackCoordinator](../../src/JellyJam.Core/Playback/PlaybackCoordinator.cs) (commands/events) and [RadioTuner](../../src/JellyJam.Core/Radio/RadioTuner.cs) (coalesced refresh signals). |
| SafeTask | [Type-only logging wrapper](../../src/JellyJam.MAUI/Services/SafeTask.cs) for callbacks without awaiting callers; not a user notification or persistence guarantee. |
| Atomic file replacement | Write temporary file then replace target in [JsonStationStore](../../src/JellyJam.Core/Configuration/JsonStationStore.cs); does not create a transaction across token/lineup/cache stores. |
| Linked-source test | [MAUI.Tests](../../tests/JellyJam.MAUI.Tests/JellyJam.MAUI.Tests.csproj) compiles selected production helpers and embeds XAML without building the full MAUI app. |
| Cocona / Spectre.Console | CLI command parsing and terminal rendering in [Console Program](../../src/JellyJam.Console/Program.cs), not MAUI dependencies. |
