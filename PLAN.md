# AniTrack - Anime Video Player & Tracker

> Local video player with dual-audio/subtitle switching and episode tracking.
> **Platform:** Android (Kotlin + Jetpack Compose)
> **Metadata Provider:** AniList (free, open-source GraphQL API)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Video Player | Media3 (ExoPlayer) |
| Database | Room (SQLite) |
| DI | Hilt |
| Image Loading | Coil |
| Preferences | Jetpack DataStore |
| Metadata API | AniList GraphQL (free) |
| Async | Kotlin Coroutines + Flow |
| Navigation | Compose Navigation |

---

## 0. Product Scope & Decisions

### MVP (must have)
- Local folder scan (SAF) for mkv/mp4 files
- Play videos with full Media3 integration
- Audio track switching (dual audio support)
- Subtitle track switching (embedded subs)
- Resume playback from last position
- Watch state tracking (Not Started / Watching / Watched)
- Progress dashboard UI
- Continue Watching section on home screen
- Manual "Mark as Watched / Unwatched"

### V1 (should have)
- AniList metadata (posters, titles, episode counts)
- Search & match shows to AniList entries
- Missing episode indicators
- Auto-play next episode with countdown
- PiP (Picture-in-Picture)

### V2 (nice to have)
- External subtitle files (.srt / .ass)
- Subtitle timing offset adjustment
- Cast support (Chromecast)
- Cloud sync / backup
- Downloads manager

### Conventions
- **Supported naming:** `[SubGroup] Anime Title - S01E01 [1080p].mkv`, `Anime Title/Season 1/01.mkv`, and similar patterns
- **Watched threshold:** episode is "watched" if >= 92% played OR player reaches STATE_ENDED
- **Default audio:** Japanese if available, else first track
- **Default subtitles:** English if available, else Off

---

## 1. Project Setup & Architecture

### 1.1 Create Android Studio project
- Kotlin, Compose, Material 3, min SDK 26

### 1.2 Package structure (single module, clean packages)
```
com.anitrack/
  data/
    local/           # Room entities, DAOs, database
    repository/      # Repository implementations
    datastore/       # Preferences DataStore
  domain/
    model/           # Domain models
    usecase/         # Use cases
  player/            # Media3 wrapper, MediaSession service
  ui/
    library/         # Library/home screen
    showdetail/      # Show detail screen
    player/          # Player screen
    progress/        # Progress/tracker screen
    settings/        # Settings screen
    components/      # Shared composables
    theme/           # Material 3 theme, colors, typography
  util/              # Filename parser, format helpers
  di/                # Hilt modules
```

### 1.3 Architecture rules
- MVVM: Screen -> ViewModel -> UseCase -> Repository -> DAO / Media3
- UI state via `StateFlow`
- Navigation: Library -> Show Detail -> Player | Progress | Settings

### 1.4 Dependencies
- Media3 (ExoPlayer, UI, Session)
- Room + KSP
- Hilt
- Coil (Compose)
- DataStore (Preferences)
- Compose Navigation
- Kotlin Coroutines + Flow

**Deliverable:** App builds, launches, shows placeholder screens with navigation.

---

## 2. Data Model & Database (Room)

### 2.1 Entities

**LibraryFolderEntity**
| Column | Type | Notes |
|---|---|---|
| id | Long (PK, auto) | |
| treeUri | String | Persisted SAF tree URI (**unique constraint**) |
| displayName | String | Human-readable folder name |
| isEnabled | Boolean | Can disable without removing |
| lastScanAt | Long? | For incremental scanning |
| scanCounter | Long | Incremented each scan cycle, default 0 |
| createdAt | Long | Timestamp |

**ShowEntity**
| Column | Type | Notes |
|---|---|---|
| id | Long (PK, auto) | |
| titleRaw | String | From filename/folder (display candidate) |
| titleKey | String | Normalized key for grouping (**indexed**) |
| titleDisplay | String? | After AniList match or user edit |
| posterUrl | String? | From AniList |
| provider | Enum (NONE, ANILIST) | |
| providerId | Int? | AniList media ID |
| createdAt | Long | Timestamp |
| updatedAt | Long | Timestamp |

> **Title normalization function** (`normalizeTitle(raw) -> titleKey`):
> 1. Lowercase
> 2. Remove punctuation (keep alphanumeric + spaces)
> 3. Collapse whitespace
> 4. Strip known tags: "1080p", "720p", "x265", "x264", "hevc", "bluray", "bdrip", etc.
> 5. Optionally remove year in parentheses
> 6. Trim
>
> `titleKey` is used for show grouping during scanning. `titleRaw` is kept as the display candidate.

**EpisodeEntity**
| Column | Type | Notes |
|---|---|---|
| id | Long (PK, auto) | |
| showId | Long (FK) | |
| libraryFolderId | Long (FK) | Which library folder this came from |
| seasonNumber | Int | Default 1 |
| episodeNumber | Int | |
| fileUri | String | Persisted SAF `content://` URI (not a file path) |
| documentId | String? | SAF document ID for stable reference |
| displayName | String | File display name from SAF |
| mimeType | String? | e.g. "video/x-matroska" |
| durationMs | Long? | Filled after first play |
| fileSize | Long | For change detection |
| lastModified | Long? | File last modified timestamp |
| stableId | String? | Hash of (treeUri + displayName + size + lastModified) |
| availability | Enum (AVAILABLE, MISSING, NEED_RELINK) | Default AVAILABLE |
| lastSeenScanId | Long | Set to folder's `scanCounter` when file seen during scan |
| parseConfidence | Float | 0.0-1.0, how confident the parser is |
| isUserOverridden | Boolean | True if user manually edited mapping |
| addedAt | Long | |

> **Unique identity priority (fallback order):**
> 1. `(libraryFolderId, documentId)` if `documentId != null` (most stable)
> 2. `(libraryFolderId, fileUri)` as fallback (some SAF providers return null documentId)
> 3. `stableId` used separately for move detection, NOT for uniqueness
>
> **Note:** `fileUri` can change after re-link, so it's only reliable for uniqueness at scan time. After re-link, update `fileUri` but keep the same episode `id`.

**WatchStateEntity**
| Column | Type | Notes |
|---|---|---|
| episodeId | Long (PK, FK) | |
| status | Enum (NOT_STARTED, WATCHING, WATCHED) | |
| startedAt | Long? | |
| finishedAt | Long? | |
| lastPositionMs | Long | Default 0 |
| lastUpdatedAt | Long | |

**TrackPreferenceEntity**
| Column | Type | Notes |
|---|---|---|
| showId | Long (PK, FK) | |
| preferredAudioLang | String? | e.g. "ja" |
| preferredAudioTrackKey | String? | Composite key (see format below) |
| preferredSubLang | String? | e.g. "en", "off" |
| preferredSubTrackKey | String? | Composite key (see format below) |

> **Track key format:** `language:label:roleFlags:channelCount:sampleMimeType:bitrate`
> Built from Media3 `Format` fields. `bitrate` included to disambiguate tracks that share lang/label/codec (e.g. stereo vs 5.1 audio, or multiple commentary tracks). Falls back to language-only matching if key doesn't match (tracks may vary across episodes).

### 2.2 DAOs
- **LibraryFolderDao:** insert/update/delete, getAllEnabled, getById, updateLastScan
- **ShowDao:** insert/update, getAllShowsWithProgress, getShowById, search, mergeShows
- **EpisodeDao:** upsert, getByShowAndSeason, getNextUnwatched, getByAvailability, getNeedsReview (parseConfidence < threshold AND !isUserOverridden)
- **WatchStateDao:** updateProgress, markWatched, markUnwatched, getContinueWatching
- **TrackPrefDao:** getForShow, saveForShow

### 2.3 Migrations
- Start at version 1, plan migration files for schema changes.

**Deliverable:** DB created; can insert mock data and query it.

---

## 3. Library Scanning (Files -> Shows/Episodes)

### 3.1 Storage access
- Use **SAF (Storage Access Framework)** - user selects library folders
- Persist URI permissions via `takePersistableUriPermission`
- Store folders in **Room** (`LibraryFolderEntity`) - enables per-folder scan time, enable/disable, UI management
- "Add Library Folder" flow: SAF folder picker -> persist permission -> insert `LibraryFolderEntity`

### 3.2 File discovery
- Scan enabled `LibraryFolderEntity` folders recursively via `DocumentsContract`
- For each file, collect SAF metadata:
  - `fileUri` (content:// URI)
  - `documentId`
  - `displayName`
  - `mimeType`
  - `fileSize`
  - `lastModified`
- Generate `stableId` = hash of (treeUri + displayName + size + lastModified) as fingerprint

### 3.3 Filename parsing (three-tier approach)

**Tier 1 - Folder context (check parent directories first):**
- If parent folder matches `Season (\d+)` or `S(\d+)` -> season inferred from folder
- If parent folder matches `OVA|Special|SP|Specials` -> `seasonNumber = 0` (specials bucket)
- If show folder name exists (grandparent) -> title inferred from folder, not filename
- Folder context takes priority over filename for title/season

**Tier 2 - Strip noise from filename:**
- Remove bracket patterns: `[SubGroup]`, `[1080p]`, `[HEVC]`, `(BD)`
- Remove known codec/quality tags

**Tier 3 - Extract metadata (try in order):**
1. `S(\d+)E(\d+)` pattern
2. `(\d+)x(\d+)` pattern
3. `Season (\d+).*Episode (\d+)` pattern
4. Trailing number as episode (folder already gave season)
5. **Fallback:** `parseConfidence = 0.0`, goes to "Needs Review" queue

**Output:** `showTitleRaw`, `seasonNumber`, `episodeNumber`, `parseConfidence` (0.0 - 1.0)

**Confidence scoring:**
- Folder context + regex match = **1.0**
- Regex match only (no folder context) = **0.8**
- Trailing number only = **0.5**
- No pattern matched = **0.0** (goes to review queue)

> Store parser version in DB so files can be re-parsed when parser improves. Skip re-parsing episodes where `isUserOverridden = true`.

### 3.4 "Needs Review" queue
- Episodes with `parseConfidence < 0.5` AND `isUserOverridden = false` appear in a review queue
- User can assign: show, season, episode number
- Once approved, set `isUserOverridden = true` (parser upgrades won't overwrite)

### 3.5 Show grouping
- Group episodes by `titleKey` (output of `normalizeTitle(showTitleRaw)`)
- Match existing `ShowEntity` by `titleKey` first; create new if no match
- Allow manual rename/merge from Show Detail screen

### 3.6 Incremental scanning (scanCounter-based)
**Scan flow:**
1. Increment `LibraryFolderEntity.scanCounter`
2. Scan folder recursively
3. For each file found: upsert episode, set `lastSeenScanId = scanCounter`
4. After scan completes: any episodes in this folder where `lastSeenScanId < scanCounter` -> set `availability = MISSING`

**Additional rules:**
- Compare `fileSize` + `lastModified` to detect content changes on existing URIs
- If SAF URI invalid but `stableId` found at new URI: set `availability = NEED_RELINK`
- If file re-appears (matched by `stableId`): set `availability = AVAILABLE`, update URI
- Update `LibraryFolderEntity.lastScanAt` on completion

> This scanCounter approach is more reliable than timestamp-only inference, because it catches files that disappeared without any timestamp change.

### 3.7 Availability states
```
AVAILABLE    -> file exists and URI is valid (lastSeenScanId = current scanCounter)
MISSING      -> file not seen in last scan (lastSeenScanId < scanCounter)
NEED_RELINK  -> stableId matched at different URI, needs user confirmation
```

**Deliverable:** Library screen shows real shows/seasons/episodes; review queue catches bad parses; availability is reliably tracked via scanCounter.

---

## 4. Player Core (Media3)

### 4.1 Media3 setup + player ownership rule

> **Critical rule:** The `ExoPlayer` instance lives in the `MediaSessionService`, NOT per-screen. The UI binds to the service and observes state. The player screen must NOT recreate the player on recomposition. This prevents: playback restarting on config changes, state desync between notification and UI, and crashes when leaving/returning to the player.

- `PlayerController` wrapper class (singleton, delegating to service player):
  - Bind to `PlayerMediaSessionService` to get/create ExoPlayer
  - Prepare MediaItem from SAF URI
  - Expose state via StateFlow: playing, buffering, duration, position, tracks
  - Player screen observes this controller, never touches ExoPlayer directly

### 4.2 MediaSession + foreground service
- Implement `MediaSessionService` for background playback
- **Subtasks:**
  - Create `PlayerMediaSessionService` extending `MediaSessionService`
  - **Audio focus management:** request/abandon audio focus, duck/pause on transient loss
  - **Noisy intent handling:** pause when headphones unplugged (`ACTION_AUDIO_BECOMING_NOISY`)
  - **Media button support:** play/pause/skip from Bluetooth headsets and lockscreen
  - **Notification channel:** create dedicated channel, foreground notification with episode-aware actions
  - **Lifecycle:** handle process death gracefully - save state before death, restore session + offer resume on relaunch (V1: best-effort, save to Room on every progress update)
  - **Release resources:** stop foreground service + release player when user explicitly stops or notification dismissed
- **Notification actions (must match player logic):**
  - Play / Pause
  - Skip to next episode (not just next track - must trigger your episode chain logic)
  - Skip to previous episode
  - Seek forward / back (10s)
  - Stop (dismiss notification + release player)
  - Notification metadata: show title, episode number, poster art

### 4.3 Player screen (Compose)
- Video surface via `AndroidView` wrapping `PlayerView`
- Controls overlay:
  - Play/pause, seek bar with timestamps
  - 10s skip forward/back buttons
  - Playback speed selector
  - Show title + S01E01 label
- Fullscreen: lock to landscape, hide system bars
- Gestures:
  - Tap to show/hide controls
  - Double-tap left/right to seek -10s/+10s
  - Swipe up/down for volume/brightness (optional V1)

### 4.4 Resume support
- On episode open: load `WatchState.lastPositionMs`, seek after prepared
- Save progress: every 5 seconds while playing + on pause/stop/background
- On first play: store `durationMs` to `EpisodeEntity`

### 4.5 Next episode
- "Next Episode in 10s" countdown overlay when current episode ends
- Auto-play next (toggle in settings)
- "Next episode" button always available in controls

**Deliverable:** MKV/MP4 playback is smooth, resume works, media notification present.

---

## 5. Audio & Subtitle Track Switching

### 5.1 Track discovery
- From Media3 `Tracks` object, extract:
  - Audio tracks: language, label, codec, channel count
  - Text tracks: language, label, forced/default flags
- Build UI models: `AudioTrackOption`, `SubtitleTrackOption`
- Handle tracks with no language tag: display as "Track 1", "Track 2"

### 5.2 Track selection (no desync)
- On switch:
  1. Store current playback position
  2. Apply new `TrackSelectionParameters`
  3. Re-seek if position drifts (safety check)
- "Off" option for subtitles: disable text renderer
- Switching must **never** restart playback from 0

### 5.3 Default & auto-apply
- On episode start, apply preferences in order:
  1. Match by `preferredAudioTrackKey` / `preferredSubTrackKey` (exact composite key match)
  2. Fallback: match by `preferredAudioLang` / `preferredSubLang` (language-only)
  3. Global fallback: audio=ja, subs=en
  4. Last resort: first available track
- When user changes tracks mid-playback: persist both language AND track key to `TrackPreferenceEntity`
- Track key = `language:label:roleFlags:channelCount:sampleMimeType:bitrate` (handles "Signs & Songs" vs "Full Subs", commentary tracks, stereo vs 5.1, etc.)

### 5.4 Track picker UI
- Bottom sheet with audio and subtitle sections
- Current selection highlighted
- Language + label + extra info displayed

**Deliverable:** Track picker works smoothly, selections persist per show, switching is seamless.

---

## 6. Watch Tracking Engine

### 6.1 State transitions
```
NOT_STARTED --[play]--> WATCHING (set startedAt)
WATCHING --[playing]--> update lastPositionMs every 5s
WATCHING --[>= 92% OR STATE_ENDED]--> WATCHED (set finishedAt)
WATCHED --[user replays]--> keep WATCHED, update lastPositionMs
                            if re-watched past threshold, update finishedAt
```

### 6.2 Manual controls
- Long-press episode -> "Mark as Watched" / "Mark as Unwatched"
- Bulk mark: "Mark season as watched"

### 6.3 Completion edge cases
- Duration unknown: rely on `Player.STATE_ENDED`
- Credits skip: still counts if threshold reached
- Rewatch: preserve WATCHED status, just update timestamps

### 6.4 Next episode logic
- Same season: episodeNumber + 1
- Season end: next season, episode 1
- File missing: show "Episode X not found - download needed"

**Deliverable:** Watch states update automatically; manual marking works; next episode logic is correct.

---

## 7. UI/UX - Premium Dark Theme

### 7.1 Design system
- Material 3 with custom dark theme (default)
- Curated color palette (not generic), subtle gradients
- Rounded corners, thin outlines, soft elevation
- Google Fonts typography (Inter or Outfit)
- Spring animations for sheets and transitions
- AnimatedVisibility for player controls

### 7.2 Screens

#### A) Library / Home Screen
- **Top section:** "Continue Watching" carousel (most important, first thing user sees)
  - Episode card with progress bar, resume button
- **Below:** Full library grid
  - Show cards with poster + overall progress ring
  - Sort: Recently watched | A-Z | Most progress
  - Search bar

#### B) Show Detail Screen
- Header: poster + title + overall status badge
- Season tabs (or dropdown if many seasons)
- Episode list with:
  - Watched/watching/unwatched icon
  - Resume time (if watching)
  - File availability indicator (AVAILABLE / MISSING / NEED_RELINK badges)
  - "Play Next" CTA button at top
- **Edit mapping tools** (accessible from overflow menu or long-press):
  - Rename show title / merge two shows into one
  - Edit episode number (fix wrong parse)
  - Re-assign episode to different show/season
  - Re-link file to an episode (when NEED_RELINK)
  - All edits set `isUserOverridden = true` to survive future parser upgrades

#### C) Player Screen
- Minimal, cinematic controls
- Glass-effect bottom overlay
- Track selector as modal bottom sheet
- Next episode countdown near end

#### D) Progress / Tracker Screen
- Sections: Continue Watching | Currently Watching | Completed
- Per show: watched X / Y episodes, next episode suggestion
- Missing episodes highlighted (need to download)

#### E) Needs Review Screen
- List all episodes with `parseConfidence < 0.5` AND `isUserOverridden = false`
- Quick-assign UI: pick show (from existing or create new), season, episode number
- Approve button sets `isUserOverridden = true`
- Badge count on navigation item so user knows items need attention
- Accessible from Library screen (banner/badge) and Settings

#### F) Settings Screen
- Library folder management (add/remove/enable/disable per folder)
- Auto-play next toggle
- Watched threshold slider (90-97%)
- Default audio/subtitle language (global fallback)
- Rescan library button
- About / version

**Deliverable:** App feels modern and premium with smooth animations.

---

## 8. AniList Metadata (V1)

### 8.1 AniList GraphQL client
- Use Ktor or OkHttp for HTTP
- Key queries:
  - Search anime by title
  - Get media details: titles (romaji/english/native), poster, episode count, status

### 8.2 Match flow
- In Show Detail: "Match to AniList" button
- Shows search results, user confirms the correct match
- Saves: provider=ANILIST, providerId, posterUrl, titleDisplay

### 8.3 Auto-suggestions (optional)
- On new show creation, suggest top AniList result
- Never auto-assign without user confirmation (titles can be ambiguous)

**Deliverable:** Shows display proper titles and poster art after matching.

---

## 9. Reliability & Edge Cases

### 9.1 File changes & availability
- SAF URI becomes invalid: set `availability = MISSING`
- File moved (stableId found at new URI): set `availability = NEED_RELINK`, prompt user
- Deleted files: set `availability = MISSING`, keep watch history, show download-needed indicator
- Re-link flow: user picks new file, update `fileUri` + `documentId`, set `availability = AVAILABLE`

### 9.2 Tricky MKV streams
- Tracks without language labels: display as "Track 1", "Track 2"
- Forced subtitles: mark clearly in picker
- Embedded ASS/SSA: works on most devices, note V2 fallback if rendering issues

### 9.3 Performance
- Library scanning on background coroutine
- Paging for long episode lists (Paging 3 if needed)
- Coil handles poster caching

### 9.4 Testing checklist (manual QA)
Test with at minimum:
- [ ] MP4 single audio, no subs
- [ ] MKV dual audio (JP + EN)
- [ ] MKV with 3+ subtitle tracks
- [ ] MKV with no language tags on tracks
- [ ] File with unusual naming pattern
- [ ] Resume after app kill
- [ ] Track switch doesn't reset position
- [ ] Watch state transitions correctly
- [ ] Next episode works across seasons

**Deliverable:** Stable behavior across common file types.

---

## 10. Release Prep

- App icon + splash screen
- Version code/name
- DB export/backup option (optional)
- Crash reporting (optional, Firebase Crashlytics)
- Build signed APK / AAB

**Deliverable:** Installable release build.

---

## Milestone Timeline

| # | Milestone | Est. Days | Dependencies |
|---|---|---|---|
| 1 | Project setup + architecture | 1-2 | None |
| 2 | DB schema + Room setup | 1-2 | M1 |
| 3 | Library scan + filename parser | 3-4 | M2 |
| 4 | Library UI | 2-3 | M3 |
| 5 | Player MVP + resume + MediaSession | 3-4 | M2 |
| 6 | Audio/subtitle switching | 2-3 | M5 |
| 7 | Watch tracking engine | 2-3 | M5, M2 |
| 8 | Progress/tracker UI | 2-3 | M7 |
| 9 | UI polish + animations | 2-3 | M4, M8 |
| 10 | AniList metadata | 2-3 | M4 |
| 11 | Edge cases + QA testing | 2-3 | All above |
| 12 | Release prep | 1 | M11 |

**Total: ~25-35 days** of focused development.

---

## Status

- [ ] M1: Project setup
- [ ] M2: Database
- [ ] M3: Library scanning
- [ ] M4: Library UI
- [ ] M5: Player
- [ ] M6: Track switching
- [ ] M7: Watch tracking
- [ ] M8: Progress UI
- [ ] M9: UI polish
- [ ] M10: AniList
- [ ] M11: Edge cases
- [ ] M12: Release
