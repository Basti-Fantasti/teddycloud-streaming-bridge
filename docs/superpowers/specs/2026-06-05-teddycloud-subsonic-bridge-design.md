# TeddyCloud–Subsonic Bridge — Design

**Date:** 2026-06-05
**Status:** Approved design / pre-implementation

## Summary

A standalone Python (FastAPI) service, packaged as a Docker container, that runs
on the same host as TeddyCloud and makes audio from a Subsonic-compatible server
(e.g. Navidrome) playable on a Toniebox via TeddyCloud.

It works in two complementary ways:

1. **Playlist generation** — writes TeddyCloud playlist JSON files (the
   `"type":"tap"` format) into TeddyCloud's mounted library folder. Each track
   becomes a `files[]` entry whose `filepath` is a bridge proxy URL.
2. **Audio proxying** — serves the per-track MP3 URLs those playlists reference,
   fetching from Subsonic (transcoded to MP3 server-side) and piping the bytes
   through to TeddyCloud.

The user browses Subsonic in a small web UI, creates named mappings, and the
bridge writes the corresponding playlist file. The user then assigns that
playlist to a Tonie/NFC tag using TeddyCloud's own UI.

## Goals

- Play Subsonic albums, playlists, and artist/starred collections on the Toniebox.
- Output that TeddyCloud already understands: an MP3 file or a continuous
  web-radio-style MP3 stream.
- Native track behaviour on the box (titles, skip forward/back, resume) for
  finite content, via TeddyCloud playlists.
- Keep Subsonic credentials in one place (the bridge), not embedded in files.
- Simple deployment: one container next to TeddyCloud, sharing the library volume.

## Non-Goals (YAGNI for v1)

- ffmpeg-based gapless/normalized concatenation.
- Multiple Subsonic servers.
- Bridge authentication (assumes a trusted LAN; documented in README).
- Direct TeddyCloud tag-assignment API calls (the user assigns in TeddyCloud's UI).
- TAF (Tonie Audio Format) generation — TeddyCloud transcodes our MP3 to Opus.

## Architecture

```
┌─────────────┐    Subsonic API     ┌──────────────────────────┐
│  Subsonic   │◀────browse/stream───│   teddycloud-subsonic     │
│ (Navidrome) │    (format=mp3)     │   bridge  (FastAPI)       │
└─────────────┘                     │                           │
                                    │  • Subsonic client        │
┌─────────────┐  reads playlist     │  • Playlist generator     │
│ TeddyCloud  │◀──files (mounted)───│  • Streaming/proxy        │
│             │                     │  • Storage (SQLite)       │
│             │──GET /track/{id}───▶│  • Web UI + JSON API      │
└─────────────┘   .mp3 (proxy)     └──────────────────────────┘
       ▲                                      ▲
       │ writes *.tap into                    │ browser:
       └─ mounted library volume              └─ manage mappings
```

### Mapping modes

- **Finite** (albums, playlists): one playlist file with N named track entries,
  each pointing at a bridge proxy URL. TeddyCloud provides native track
  skip/resume/titles. The track list is **snapshotted** into the playlist file at
  creation time; Subsonic changes require a "regenerate".
- **Endless radio** (artist / starred): a playlist file with a **single** entry
  pointing at a bridge "radio" endpoint that loops (optionally shuffled) over the
  mapping's tracks and never closes the connection. Tracks are resolved **live**
  at playtime, so changes are picked up automatically.

## Components

Six focused modules, each with one clear job and a narrow interface.

### 1. `subsonic` — Subsonic API client
- Authenticated calls using token auth: `t=md5(password+salt)`, `s=salt`,
  plus `u`, `v`, `c`, `f=json`.
- Methods: `ping()`, `getArtists`/`search`, `getArtist`, `getAlbum`,
  `getAlbumList2`, `getPlaylists`, `getPlaylist`, `getStarred2`, and
  `stream(id, format="mp3", maxBitRate)` returning a streaming HTTP response.
- Returns typed dataclasses (`Artist`, `Album`, `Track`, `Playlist`) — raw JSON
  never leaks out.
- Depends on: `config`, `httpx` (async).

### 2. `storage` — persistence (SQLite)
- Stores connection settings and **mapping records**.
- Lets the UI list/edit/regenerate/delete mappings even though the playlist files
  live in TeddyCloud's folder.
- Interface: repository functions (`list_mappings`, `get`, `save`, `delete`,
  `get_setting`, `set_setting`).

**Schema (two tables):**
- `settings(key TEXT PRIMARY KEY, value TEXT)`
- `mappings(id, name, kind, source_type, source_id, shuffle, playlist_path,
  created_at, updated_at)`
  - `kind` ∈ {`finite`, `endless`}
  - `source_type` ∈ {`album`, `playlist`, `artist`, `starred`}

### 3. `playlist` — TeddyCloud playlist generator
- Turns a mapping + resolved track list into the `{"type":"tap", ...}` JSON and
  writes it atomically (temp file + rename) into the mounted library dir.
- Builds each `files[]` entry: `name` = track title, `filepath` = bridge proxy URL.
- Handles slug/filename generation and file removal on delete.
- Depends on: `storage`, `config`.

**Generated finite playlist shape:**
```json
{
  "type": "tap",
  "audio_id": 0,
  "filepath": "lib://.../<slug>.tap",
  "name": "<mapping name>",
  "files": [
    {"filepath": "http://<bridge-base>/track/<trackId>.mp3", "name": "<title>"}
  ]
}
```

**Generated endless playlist shape:** identical, but a single entry pointing at
`http://<bridge-base>/radio/<mappingId>.mp3`.

> Playlist file extension defaults to `.tap`; confirm against TeddyCloud's
> expected convention during implementation and make it configurable if needed.

### 4. `streaming` — audio proxy + radio
- `GET /track/{id}.mp3` → opens Subsonic `stream(format=mp3, maxBitRate)` and
  streams bytes through (passthrough; forward/honor Range if present). On EOF the
  track ends and TeddyCloud advances.
- `GET /radio/{mappingId}.mp3` → resolves the mapping's track set (optionally
  shuffled) and streams track after track via Subsonic MP3, looping forever,
  never closing. On a failed track fetch, skip to the next rather than killing
  the stream.
- Depends on: `subsonic`, `storage`.

### 5. `web` — UI + JSON API
- FastAPI routes: JSON API for browse/search, CRUD on mappings, "regenerate
  playlist", and a settings / test-connection endpoint. Serves the UI.
- **UI:** server-rendered Jinja2 + HTMX (no build step, small footprint).
- See the **Web UI** section for the full screen-by-screen design and visual
  language.

### 6. `config` — settings & wiring
- Loads config from env vars (bootstrap) overlaid by values saved in `storage`.
- Holds: Subsonic URL/credentials, TeddyCloud library path, bridge public base
  URL, MP3 bitrate, bind address/port.

**Cross-cutting:** logging, and a thin `models` module for shared dataclasses.

## Data Flow

### A. Creating a finite mapping (album/playlist)
1. User browses Subsonic in the UI, picks an album, clicks "Create Tonie
   playlist", names it, picks mode = finite.
2. `web` resolves the album via `subsonic.getAlbum(id)` → ordered track list.
3. `storage.save()` persists the mapping record.
4. `playlist` builds the JSON and writes it atomically into the mounted library dir.
5. UI shows success + the file location so the user can assign it to a tag in
   TeddyCloud.

### B. Playback of a finite mapping
1. Box taps tag → TeddyCloud reads the playlist file → iterates `files[]`.
2. For each entry, TeddyCloud fetches `GET /track/<id>.mp3` from the bridge.
3. `streaming` calls `subsonic.stream(...)` and pipes bytes; on EOF, TeddyCloud
   advances to the next entry.
4. Titles/skip/resume handled natively by TeddyCloud from the playlist entries.

### C. Endless radio mapping (artist/starred)
1. User picks an artist or "starred", mode = endless, optional shuffle.
2. `playlist` writes a playlist file with a single entry →
   `http://<bridge-base>/radio/<mappingId>.mp3`.
3. On play, `streaming` resolves the track set, optionally shuffles, and streams
   track after track via Subsonic MP3, looping forever.

### D. Regeneration / edit / delete
- Edit name/mode/shuffle → update record, rewrite playlist file.
- Delete → remove playlist file from the mounted dir + delete the record.
- "Regenerate" re-resolves tracks from Subsonic and rewrites the file (finite
  only; endless resolves live).

## Web UI

Server-rendered Jinja2 + HTMX, no build step. The UI has three surfaces inside a
shared app shell. Interactive mockups (reference, not committed) live under
`.superpowers/brainstorm/` from the design session.

### Visual language ("cozy")

A warm, tactile "playroom-meets-homelab" aesthetic — friendly to the Toniebox
world but still a focused admin tool.

- **Background:** warm cream "paper" (`#F6EFE3`) with a very subtle noise grain
  and faint radial gold/sage glows.
- **Surfaces:** off-white cards (`#FFFDF8`), `1px` warm borders (`#E7DBC7`),
  generous `12–18px` rounded corners, soft shadows.
- **Accent:** coral (`#E0603F`, deep `#C44A2C`) for primary actions and the
  active nav item (with a chunky `4px` bottom "press" shadow). Secondary: sage
  `#3E8C7C`, gold `#E0A43B`.
- **Ink/muted text:** `#2A2420` / `#8C7F70`.
- **Typography:** display = **Fraunces** (characterful serif, headings);
  body = **Hanken Grotesk**. Both from Google Fonts; deliberately not generic.

### App shell

- **Left sidebar (≈248px):** brand mark + name; nav with three items —
  **Library**, **Tonies**, **Settings**; a spacer; and a **connection-status
  panel** at the bottom showing Subsonic (connected) and Library (mounted) dots.
- **Main area:** page title + contextual content.

### Library screen (browse + create)

- Title, description, and a **search box** (albums / artists / playlists).
- **Type tabs** with counts: Albums · Artists · Playlists · Starred.
- **Cover grid** of cards (cover art, title, subtitle). Hovering a card lifts it
  and reveals a **＋ Tonie** button — the entry point to the create flow.

### Create Tonie drawer

Slides in from the right over a dimmed library.

- **Source summary:** cover, title, subtitle, and quick facts — track count and
  total duration (one extra Subsonic call when the drawer opens; accepted).
- **Tonie name:** prefilled from the source, editable; becomes the playlist
  `name` and the filename slug.
- **Playback mode:** two selectable cards — **Finite** (plays tracks in order,
  skip & resume) and **Endless** (loops like a radio). Selecting **Endless**
  reveals a **Shuffle** toggle.
- **Playlist file preview:** shows the exact target path
  (e.g. `/teddycloud/library/der-gruffelo.tap`).
- **Footer:** Cancel / ＋ Create Tonie.
- **Success state:** drawer flips to "✓ Playlist written to `<file>.tap`" with a
  **copy `lib://…` path** button and a reminder: *"Now assign this playlist to a
  Tonie/tag in TeddyCloud's admin UI."* (We deliberately don't call TeddyCloud's
  tag API — this handoff hint closes the loop.)

### Tonies screen (manage)

- Title + **＋ New Tonie** button (alternate entry into the create flow).
- A **list of rows**, one per mapping: cover, name, a **mode badge**
  (📖 Finite / 📻 Endless·shuffle), a summary line (source type · track count ·
  duration / "resolves live" for endless), and the `lib://…` path with a **copy**
  button.
- **Per-row actions:** ↻ **Regenerate** (re-resolve from Subsonic and rewrite the
  file — finite only), ✎ **Edit** (rename / change mode / shuffle), 🗑 **Delete**
  (removes the record *and* the `.tap` file).

### Settings screen

Grouped panels with a single Save / Reset action bar:

1. **Subsonic server:** URL, username, password, **Test connection** button with
   live status (server type/version, album count). Bootstraps from env vars if
   set; UI values persist in SQLite and take over.
2. **TeddyCloud library:** mounted library path shown **read-only** with a
   mounted/writable health check (it's a Docker mount — editing here wouldn't
   remount anything); plus the playlist file extension (default `.tap`).
3. **Bridge address:** the base URL TeddyCloud uses to reach the audio proxy —
   baked into every playlist entry.
4. **Audio & defaults:** MP3 bitrate, default mode for new Tonies, and a
   "shuffle endless radios by default" toggle.

> **Note on credentials:** the Subsonic password is stored in the bridge's
> SQLite volume (needed to compute Subsonic token auth). Acceptable under the
> trusted-LAN assumption.

## Configuration

Env bootstrap, overridable in the UI, stored in SQLite:

- `SUBSONIC_URL`, `SUBSONIC_USER`, `SUBSONIC_PASSWORD`
- `TEDDYCLOUD_LIBRARY_PATH` (mounted dir where `.tap` files are written)
- `BRIDGE_BASE_URL` (what TeddyCloud uses to reach the proxy, e.g.
  `http://bridge:8080`)
- `MP3_BITRATE` (default 192)
- `BIND_ADDR` / `PORT`

## Error Handling

- **Subsonic unreachable / auth fail:** surfaced via a "Test connection" action;
  browse calls show a clear error, never a stack trace.
- **Track fetch fails mid-playback:** logged; endless mode skips to the next
  track; finite mode ends the track early and TeddyCloud advances.
- **Library path not writable / not mounted:** startup check + clear UI banner;
  playlist creation refuses with a helpful message.
- **Bad/empty source** (e.g. album with 0 tracks): refuse to create the mapping
  with a validation message.
- **Atomic writes:** temp file + rename so TeddyCloud never reads a half-written
  playlist.

## Testing

- **Unit:** `subsonic` client (mock httpx — URL/auth-param construction, response
  parsing); `playlist` generator (exact `type:tap` JSON shape, slug, atomic
  write); `storage` repository (in-memory SQLite).
- **Integration:** FastAPI `TestClient` over the JSON API — create/list/edit/
  delete mapping → assert the right file appears in a temp "library" dir with
  correct contents; proxy endpoint streams from a stubbed Subsonic.
- **Manual/E2E checklist:** real Subsonic + real TeddyCloud — create an album
  playlist, assign to a tag, confirm playback + skip + resume; create an endless
  artist radio, confirm it loops.
- Development follows TDD: tests first, per module.

## Deployment

- `Dockerfile` (slim Python base; no ffmpeg in v1).
- `docker-compose.yml` example placing the bridge alongside TeddyCloud,
  bind-mounting TeddyCloud's library dir and a small named volume for the
  bridge's SQLite.
- README documenting the trusted-LAN assumption and configuration.

## Open Items to Confirm During Implementation

- Exact playlist file extension/location TeddyCloud expects (default `.tap`).
- Whether TeddyCloud issues HTTP Range requests to the proxy; pass-through
  handling if so.
