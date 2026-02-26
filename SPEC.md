# AniTrack - Product Spec

## Overview
Local anime video player and tracker for Android. Plays downloaded MKV/MP4 files with dual audio and subtitle switching, tracks watch progress per episode.

## Storage & File Handling
- **Storage Access Framework (SAF)** for all file access (no `MANAGE_EXTERNAL_STORAGE`)
- Files referenced by `content://` URIs (not file paths)
- Library folders managed in Room DB (not DataStore) for per-folder scan control
- Files fingerprinted via `stableId` = hash(treeUri + displayName + size + lastModified)
- Deletion detection via `scanCounter` per folder + `lastSeenScanId` per episode
- Three availability states: `AVAILABLE`, `MISSING`, `NEED_RELINK`

## Naming Convention Support (three-tier parsing)
**Tier 1 - Folder context (priority):**
- Parent folder `Season X` / `SX` infers season
- Grandparent folder infers show title

**Tier 2 - Filename patterns (after noise stripping):**
1. `S01E01` pattern
2. `01x01` pattern
3. `Season 1 Episode 01` pattern
4. Trailing number (if folder gave season)
5. Unmatched -> "Needs Review" queue (confidence < 0.5)

**Confidence:** 1.0 (folder+regex) / 0.8 (regex only) / 0.5 (trailing number) / 0.0 (no match)
**User overrides** persist across parser upgrades. Shows grouped by deterministic `titleKey` (normalized, lowercased, stripped).

## Defaults
- **Watched threshold:** 92% of episode duration OR player STATE_ENDED
- **Default audio:** Japanese if available, else first track
- **Default subtitles:** English if available, else Off
- **Track matching:** Composite key (`lang:label:roleFlags:channels:codec:bitrate`) first, then language-only fallback
- **Theme:** Dark mode

## Metadata Provider
- **AniList** (free, open-source GraphQL API)
- User-initiated matching only (no auto-assign)
- Data retrieved: titles, poster, episode count, airing status

## Supported Formats
- Containers: MKV, MP4
- Audio: all codecs supported by Media3/ExoPlayer
- Subtitles: embedded VTT/SRT (target support); ASS/SSA (best-effort, device/renderer dependent); external subs in V2
