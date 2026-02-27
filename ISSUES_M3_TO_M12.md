# AniTrack Issues - M3 to M12

Copy these into GitHub Issues via the UI. Set the Milestone, Priority, Area, and Type labels for each.

---

## M3: Scanning + Parsing

### 11. [M3] SAF folder picker + permission persistence
**Labels:** P0, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] "Add Library Folder" button launches SAF folder picker
- [ ] `takePersistableUriPermission` on selected URI
- [ ] `LibraryFolderEntity` inserted into Room
- [ ] Folder list UI shows added folders with enable/disable toggle
- [ ] Remove folder removes permission + entity
- [ ] Persisted URIs survive app restart

---

### 12. [M3] Recursive file discovery via DocumentsContract
**Labels:** P0, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] Scans enabled `LibraryFolderEntity` folders recursively
- [ ] Filters for `.mkv` and `.mp4` files only
- [ ] Collects SAF metadata: fileUri, documentId, displayName, mimeType, fileSize, lastModified
- [ ] Generates `stableId` = hash(treeUri + displayName + size + lastModified)
- [ ] Works even when `documentId == null` (uses `(folderId, fileUri)` fallback for uniqueness)
- [ ] Runs on background coroutine, does not block UI

---

### 13. [M3] Three-tier filename parser
**Labels:** P0, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] **Tier 1:** Folder context - `Season X` / `SX` infers season, `OVA|Special|SP` maps to season 0, grandparent folder infers show title
- [ ] **Tier 2:** Strips bracket noise `[SubGroup]` `[1080p]` `(BD)` and codec tags
- [ ] **Tier 3:** Regex extraction in order: `S01E01` > `01x01` > `Season/Episode` text > trailing number > fallback
- [ ] Outputs: showTitleRaw, seasonNumber, episodeNumber, parseConfidence (0.0-1.0)
- [ ] Stores parser version in DB
- [ ] Skips re-parsing where `isUserOverridden = true`

**Confidence rules:** folder+regex=1.0, regex-only=0.8, trailing-number=0.5, no-match=0.0

---

### 14. [M3] Show grouping with titleKey normalization
**Labels:** P1, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] `normalizeTitle()` function: lowercase, strip punctuation, collapse whitespace, remove quality tags
- [ ] Episodes grouped by `titleKey`
- [ ] Matches existing `ShowEntity` by `titleKey` first, creates new if no match
- [ ] Deterministic: same input always produces same `titleKey`

---

### 15. [M3] scanCounter-based incremental scanning
**Labels:** P0, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] Increments `scanCounter` on each scan
- [ ] Each file found sets `lastSeenScanId = scanCounter`
- [ ] After scan: episodes where `lastSeenScanId < scanCounter` -> `availability = MISSING`
- [ ] `stableId` match at new URI -> `availability = NEED_RELINK`
- [ ] If `stableId` matches **multiple** files, mark `NEED_RELINK` and require user pick (no auto-relink) - handles duplicate copies (1080p vs 720p)
- [ ] `stableId` re-match (single) -> `availability = AVAILABLE`, URI updated
- [ ] `lastScanAt` updated on completion

---

### 16. [M3] Needs Review queue logic
**Labels:** P1, area:scanner, type:feature
**Acceptance Criteria:**
- [ ] DAO query: `parseConfidence < 0.5 AND isUserOverridden = false`
- [ ] Count available for badge display
- [ ] Approving an item sets `isUserOverridden = true`

---

## M4: Library UI

### 17. [M4] Library / Home screen with Continue Watching
**Labels:** P0, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Top section: "Continue Watching" carousel with episode cards
- [ ] Each card shows: poster/placeholder, show title, episode label, progress bar, resume button
- [ ] Below: full library grid with show cards (poster + progress ring)
- [ ] Sort options: Recently watched, A-Z, Most progress
- [ ] Search bar filters shows
- [ ] Badge count for Needs Review items visible

---

### 18. [M4] Show Detail screen
**Labels:** P0, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Header: poster + title + overall status badge (Watching/Completed/Not Started)
- [ ] Season tabs or dropdown
- [ ] Episode list with: watch status icon, resume time, availability badge
- [ ] "Play Next" CTA button at top
- [ ] Edit mapping tools via overflow/long-press: rename show, edit episode number, re-assign, re-link, merge shows
- [ ] All edits set `isUserOverridden = true`

---

### 19. [M4] Needs Review screen
**Labels:** P1, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Lists episodes with low parseConfidence
- [ ] Quick-assign UI: pick show (existing or new), season, episode number
- [ ] Approve button sets `isUserOverridden = true`
- [ ] Badge count on navigation item
- [ ] Accessible from Library screen banner and Settings

---

### 20. [M4] Search + sort controls
**Labels:** P2, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Search bar filters library by show title
- [ ] Sort toggle: Recently watched / A-Z / Most progress
- [ ] Sort preference remembered across sessions (DataStore)

---

## M5: Player + Service

### 21. [M5] PlayerMediaSessionService (ExoPlayer ownership)
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] `PlayerMediaSessionService` extends `MediaSessionService`
- [ ] ExoPlayer instance created and owned by service, NOT per-screen
- [ ] Audio focus: request/abandon, duck/pause on transient loss
- [ ] Noisy intent: pause on headphone unplug (`ACTION_AUDIO_BECOMING_NOISY`)
- [ ] Media button support (Bluetooth headset, lockscreen)
- [ ] Lifecycle: save state to Room before death, release resources on stop

---

### 22. [M5] PlayerController singleton + state model
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] Singleton that binds to `PlayerMediaSessionService`
- [ ] Exposes StateFlow: isPlaying, isBuffering, duration, position, currentTracks
- [ ] Prepares MediaItem from SAF URI
- [ ] Player screen observes this controller only, never touches ExoPlayer directly
- [ ] Survives recomposition without recreating player

---

### 23. [M5] Player screen UI
**Labels:** P0, area:player, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Video surface via `AndroidView` wrapping `PlayerView`
- [ ] Controls overlay: play/pause, seek bar with timestamps, 10s skip buttons, speed selector
- [ ] Show title + S01E01 label displayed
- [ ] Fullscreen: lock to landscape, hide system bars
- [ ] Tap to show/hide controls
- [ ] Double-tap left/right to seek -10s/+10s
- [ ] AnimatedVisibility for controls

---

### 24. [M5] Resume save/restore
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] On episode open: load `lastPositionMs` from WatchState, seek after prepared
- [ ] Save progress every 5s while playing
- [ ] Save on pause, stop, and app background
- [ ] On first play: store `durationMs` to EpisodeEntity
- [ ] After app kill: last saved position is in Room, restored on relaunch

---

### 25. [M5] Notification with episode-aware actions
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] Dedicated notification channel created
- [ ] Foreground notification with: play/pause, next/prev episode, seek 10s, stop
- [ ] Next/prev triggers episode chain logic (not just track skip)
- [ ] Notification metadata: show title, episode number, poster art
- [ ] "Stop" action releases player + stops foreground service
- [ ] Swiping notification (if allowed by OS) behaves like Stop

---

### 26. [M5] Landscape fullscreen handling
**Labels:** P1, area:player, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Lock to landscape when player opens
- [ ] Hide system bars (immersive mode)
- [ ] Restore orientation when leaving player
- [ ] Handle configuration changes without losing player state

---

### 27. [M5] Next episode in player
**Labels:** P1, area:player, type:feature
**Acceptance Criteria:**
- [ ] "Next Episode in 10s" countdown overlay when current episode ends (with cancel option)
- [ ] Auto-play next (respects Settings toggle)
- [ ] "Next episode" button always visible in controls
- [ ] Skip next/prev from notification triggers same logic

---

### 28. [M5] PiP (Picture-in-Picture)
**Labels:** P2, area:player, type:feature
**Acceptance Criteria:**
- [ ] Enter PiP on home button press or swipe
- [ ] Playback continues in PiP window
- [ ] Basic play/pause controls in PiP

---

## M6: Track Switching

### 29. [M6] Track discovery from Media3
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] Extracts audio tracks: language, label, codec, channels, bitrate
- [ ] Extracts text tracks: language, label, forced/default flags, roleFlags
- [ ] Builds `AudioTrackOption` and `SubtitleTrackOption` UI models
- [ ] Tracks without language tags display as "Track 1", "Track 2"
- [ ] Builds composite track key: `lang:label:roleFlags:channels:codec:bitrate`

---

### 30. [M6] Track selection with no desync
**Labels:** P0, area:player, type:feature
**Acceptance Criteria:**
- [ ] On switch: store position -> apply TrackSelectionParameters -> re-seek if drift
- [ ] "Off" subtitles disables text renderer
- [ ] Switching NEVER restarts playback from 0
- [ ] Auto-apply on episode start: trackKey match -> lang match -> global fallback -> first track
- [ ] Persists both language AND track key on user change

---

### 31. [M6] Track picker bottom sheet
**Labels:** P1, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Modal bottom sheet with Audio and Subtitle sections
- [ ] Current selection highlighted
- [ ] Shows: language + label + extra info (channels, codec)
- [ ] "Off" option for subtitles
- [ ] Smooth open/close animation

---

## M7: Watch Tracking

### 32. [M7] Watch state machine
**Labels:** P0, area:tracking, type:feature
**Acceptance Criteria:**
- [ ] NOT_STARTED -> play -> WATCHING (sets startedAt)
- [ ] WATCHING updates lastPositionMs every 5s
- [ ] WATCHING -> >= 92% OR STATE_ENDED -> WATCHED (sets finishedAt)
- [ ] If duration < 5 minutes (short OVA/specials): threshold = 95% instead of 92%
- [ ] If duration unknown: don't apply the 5-min rule, fall back to STATE_ENDED only
- [ ] WATCHED -> replay -> keeps WATCHED, updates lastPositionMs
- [ ] Credits skip: counts if threshold reached

---

### 33. [M7] Manual mark watched/unwatched + bulk
**Labels:** P1, area:tracking, type:feature
**Acceptance Criteria:**
- [ ] Long-press episode: "Mark as Watched" / "Mark as Unwatched"
- [ ] "Mark season as watched" bulk action
- [ ] Updates startedAt/finishedAt/lastUpdatedAt appropriately

---

### 34. [M7] Next episode selection logic
**Labels:** P1, area:tracking, type:feature
**Acceptance Criteria:**
- [ ] Same season: episodeNumber + 1
- [ ] Season end: next season, episode 1
- [ ] If file missing: shows "Episode X not found - download needed"
- [ ] Feeds into auto-play and Continue Watching section

---

## M8: Progress UI

### 35. [M8] Progress / Tracker screen
**Labels:** P0, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Sections: Continue Watching, Currently Watching, Completed
- [ ] Per show: watched X / Y episodes, next episode suggestion
- [ ] Missing episodes highlighted with download-needed indicator
- [ ] Tap show navigates to Show Detail

---

### 36. [M8] Settings screen
**Labels:** P1, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Library folder management (add/remove/enable/disable per folder)
- [ ] Auto-play next toggle
- [ ] Watched threshold slider (90-97%)
- [ ] Default audio/subtitle language (global fallback)
- [ ] Rescan library button
- [ ] About / version info

---

## M9: UI Polish

### 37. [M9] Material 3 dark theme + design system
**Labels:** P1, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Custom dark theme with curated color palette (not generic Material defaults)
- [ ] Google Fonts (Inter or Outfit)
- [ ] Rounded corners, thin outlines, soft elevation
- [ ] Subtle gradients where appropriate
- [ ] Consistent across all screens

---

### 38. [M9] Animations + glass overlay
**Labels:** P1, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Spring animations for sheets and transitions
- [ ] Glass-effect bottom overlay on player
- [ ] Smooth shared element transitions (poster -> show detail)
- [ ] AnimatedVisibility on player controls

---

### 39. [M9] Double-tap seek gesture
**Labels:** P1, area:player, area:ui, type:feature
**Acceptance Criteria:**
- [ ] Double-tap left = -10s with ripple animation
- [ ] Double-tap right = +10s with ripple animation
- [ ] Visual feedback (seconds indicator, e.g. "-10s" text)

> **Note:** Next episode countdown overlay is in #27 (M5), not here. This issue is gesture polish only.

---

## M10: AniList

### 40. [M10] AniList GraphQL client
**Labels:** P1, area:anilist, type:feature
**Acceptance Criteria:**
- [ ] HTTP client (Ktor or OkHttp) with GraphQL queries
- [ ] Search anime by title query
- [ ] Get media details: titles (romaji/english/native), poster, episode count, status
- [ ] Handles rate limiting gracefully
- [ ] Error states shown in UI

---

### 41. [M10] Match flow UI
**Labels:** P1, area:anilist, area:ui, type:feature
**Acceptance Criteria:**
- [ ] "Match to AniList" button in Show Detail screen
- [ ] Launches search with pre-filled show title
- [ ] Shows search results with poster thumbnails
- [ ] User confirms correct match
- [ ] Saves: provider=ANILIST, providerId, posterUrl, titleDisplay

---

### 42. [M10] Auto-suggestion (optional)
**Labels:** P2, area:anilist, type:feature
**Acceptance Criteria:**
- [ ] On new show creation, suggests top AniList result
- [ ] Never auto-assigns without user confirmation
- [ ] Can be dismissed / ignored

---

## M11: QA / Edge Cases

### 43. [M11] Re-link flow UI
**Labels:** P1, area:scanner, area:ui, type:feature
**Acceptance Criteria:**
- [ ] NEED_RELINK episodes show re-link prompt
- [ ] User picks new file via SAF
- [ ] Updates fileUri + documentId, sets availability = AVAILABLE
- [ ] MISSING episodes show "file not found" with option to re-link

---

### 44. [M11] Tricky MKV stream handling
**Labels:** P1, area:player, type:feature
**Acceptance Criteria:**
- [ ] Tracks without language labels: "Track 1", "Track 2"
- [ ] Forced subtitles marked clearly in picker
- [ ] ASS/SSA: best-effort rendering, note device dependency
- [ ] No crashes on unusual stream configurations

---

### 45. [M11] Manual QA checklist
**Labels:** P0, type:docs
**Body:**
- [ ] MP4 single audio, no subs
- [ ] MKV dual audio (JP + EN)
- [ ] MKV with 3+ subtitle tracks
- [ ] MKV with no language tags
- [ ] Unusual naming pattern (fansub style)
- [ ] Resume after app kill
- [ ] Track switch doesn't reset position
- [ ] Watch state transitions correctly
- [ ] Next episode across seasons
- [ ] Needs Review flow end-to-end
- [ ] Re-link flow end-to-end
- [ ] Notification actions work correctly
- [ ] Orientation change during playback does not reset position
- [ ] Background playback survives app switch (audio continues)
- [ ] PiP works (if implemented)

---

## M12: Release

### 46. [M12] Release preparation
**Labels:** P1, area:infra, type:feature
**Acceptance Criteria:**
- [ ] App icon designed and added
- [ ] Splash screen implemented
- [ ] Version code/name set
- [ ] Signed APK / AAB built successfully
- [ ] DB export/backup option (optional)
- [ ] Crash reporting setup (optional, Firebase Crashlytics)

---

## Bonus: Add to M1 (if not already created)

### [M1] Setup GitHub Actions: assembleDebug + unit tests
**Labels:** P1, area:infra, type:techdebt
**Acceptance Criteria:**
- [ ] `.github/workflows/ci.yml` runs on push/PR to `develop` and `main`
- [ ] Runs `./gradlew assembleDebug`
- [ ] Runs `./gradlew test` (unit tests)
- [ ] Fails PR if build or tests fail

> Add this to M1 on GitHub if not already there. Saves you from broken builds later.
