# TeddyCloud Audio Bridge — Design

*(repo: `teddycloud-subsonic-bridge`)*

**Date:** 2026-06-05
**Status:** Approved design / pre-implementation

## Summary

A standalone Python (FastAPI) service, packaged as a Docker container, that runs
on the same host as TeddyCloud and makes audio from one or more **content
sources** playable on a Toniebox via TeddyCloud. Two source types are supported:

- **Subsonic** (e.g. Navidrome) — music, audiobooks.
- **Audiobookshelf** (ABS) — audiobooks / Hörspiele.

The user can connect Subsonic, Audiobookshelf, or **both**.

It works in two complementary ways:

1. **Playlist generation** — writes TeddyCloud playlist JSON files (the
   `"type":"tap"` format) into TeddyCloud's mounted library folder. Each track
   becomes a `files[]` entry whose `filepath` is a bridge proxy URL.
2. **Audio proxying** — serves the per-track MP3 URLs those playlists reference,
   fetching from the originating source and piping the bytes through to
   TeddyCloud. (Subsonic transcodes to MP3 server-side; ABS serves the already-MP3
   file directly.)

The user browses a connected source in a small web UI, creates named mappings,
and the bridge writes the corresponding playlist file. The user then assigns that
playlist to a Tonie/NFC tag using TeddyCloud's own UI.

## Goals

- Play content from Subsonic (albums, playlists, artist/starred) and
  Audiobookshelf (audiobook items) on the Toniebox.
- Output that TeddyCloud already understands: an MP3 file or a continuous
  web-radio-style MP3 stream.
- Native track behaviour on the box (titles, skip forward/back, resume) for
  finite content, via TeddyCloud playlists.
- Keep source credentials in one place (the bridge), not embedded in files.
- Simple deployment: one container next to TeddyCloud, sharing the library volume.

## Non-Goals (YAGNI for v1)

- ffmpeg-based gapless/normalized concatenation.
- Multiple instances of the same source type (one Subsonic + one ABS).
- Bridge authentication (assumes a trusted LAN; documented in README).
- Direct TeddyCloud tag-assignment API calls (the user assigns in TeddyCloud's UI).
- TAF (Tonie Audio Format) generation — TeddyCloud transcodes our MP3 to Opus.
- **Audiobookshelf:** playlists, series/collections, and podcasts (audiobook
  *items* only); ABS HLS/transcode playback path (we use the direct file
  endpoint); splitting a single-file audiobook by its embedded chapters.

## Architecture

```
┌─────────────┐   Subsonic API      ┌──────────────────────────┐
│  Subsonic   │◀──browse/stream─────│   teddycloud audio        │
│ (Navidrome) │   (format=mp3)      │   bridge  (FastAPI)       │
└─────────────┘                     │                           │
┌─────────────┐   ABS REST API      │  • ContentSource:         │
│Audiobookshelf◀──browse────────────│      – subsonic           │
│             │   /items/{id}/file  │      – audiobookshelf     │
└─────────────┘                     │  • Playlist generator     │
                                    │  • Streaming/proxy        │
┌─────────────┐  reads playlist     │  • Storage (SQLite)       │
│ TeddyCloud  │◀──files (mounted)───│  • Web UI + JSON API      │
│             │──GET /track/{src}/.▶│                           │
└─────────────┘   .mp3 (proxy)     └──────────────────────────┘
       ▲                                      ▲
       │ writes *.tap into                    │ browser:
       └─ mounted library volume              └─ manage mappings
```

The bridge talks to sources through a common **`ContentSource` interface**, so
the playlist generator, streaming proxy, storage, and UI are source-agnostic.
`subsonic` and `audiobookshelf` are two implementations of that interface.

### Mapping modes

- **Finite** (albums, playlists): one playlist file with N named track entries,
  each pointing at a bridge proxy URL. TeddyCloud provides native track
  skip/resume/titles. The track list is **snapshotted** into the playlist file at
  creation time; source-side changes require a "regenerate".
- **Endless radio** (artist / starred): a playlist file with a **single** entry
  pointing at a bridge "radio" endpoint that loops (optionally shuffled) over the
  mapping's tracks and never closes the connection. Tracks are resolved **live**
  at playtime, so changes are picked up automatically.

## Components

Focused modules, each with one clear job and a narrow interface.

### 1. `sources` — content-source abstraction

A common interface plus one implementation per source type. Everything else in
the bridge depends on this interface, never on a concrete source.

**`ContentSource` interface (async):**
- `test_connection() → SourceStatus` (ok flag, server type/version, item counts).
- `browse(kind, parent_id=None, query=None) → [BrowseEntry]` — lists the things a
  user can pick (and search). `kind` is source-specific (see below).
- `resolve_tracks(ref) → [Track]` — expands a chosen item into an ordered track
  list. `Track` carries: source id(s), `title`, `duration`, and the data needed
  to build a proxy URL.
- `open_stream(track, range=None) → StreamResponse` — opens the upstream byte
  stream for one track (status, headers incl. content-type/length, async byte
  iterator); forwards Range when the upstream supports it.
- `cover_url(ref) → str | None` — for the browse grid.

Shared typed dataclasses live in `models` (`BrowseEntry`, `Track`,
`SourceStatus`); raw upstream JSON never leaks past a source implementation.

**1a. `subsonic` implementation**
- Token auth: `t=md5(password+salt)`, `s=salt`, plus `u`, `v`, `c`, `f=json`.
- Browse kinds: `albums`, `artists`, `playlists`, `starred` (via `getAlbumList2`,
  `getArtists`/`getArtist`, `getPlaylists`/`getPlaylist`, `getStarred2`).
- `open_stream` uses `stream(id, format=mp3, maxBitRate)` (server-side transcode).
- Depends on: `config`, `httpx` (async).

**1b. `audiobookshelf` implementation**
- Auth: API token *or* username/password (`POST /login` → `user.token`); sent as
  `Authorization: Bearer` (token also accepted as `?token=`). Verified live.
- Browse kinds: `libraries` (book-type) → `items`. Uses `GET /api/libraries`,
  `GET /api/libraries/{id}/items` (paginated, with title/duration/track counts),
  `GET /api/items/{id}?expanded=1` for `media.tracks[]`.
- `resolve_tracks`: one `Track` per entry in `media.tracks[]`; each track's
  upstream path is `contentUrl = /api/items/{itemId}/file/{ino}`.
- `open_stream`: GETs that file endpoint and streams bytes through. Files are
  already MP3 (`audio/mpeg`); the endpoint returns `206`/`Accept-Ranges: bytes`
  with a real Content-Length, so Range is forwarded for clean seeking. **No
  transcode needed.**
- `cover_url`: `GET /api/items/{id}/cover`.
- Audiobook *items* only (finite). See Non-Goals for what's excluded.

### 2. `storage` — persistence (SQLite)
- Stores connection settings and **mapping records**.
- Lets the UI list/edit/regenerate/delete mappings even though the playlist files
  live in TeddyCloud's folder.
- Interface: repository functions (`list_mappings`, `get`, `save`, `delete`,
  `get_setting`, `set_setting`).

**Schema (two tables):**
- `settings(key TEXT PRIMARY KEY, value TEXT)` — includes per-source connection
  config, keyed by source (e.g. `subsonic.url`, `audiobookshelf.token`).
- `mappings(id, name, source, kind, source_type, source_id, shuffle,
  playlist_path, created_at, updated_at)`
  - `source` ∈ {`subsonic`, `audiobookshelf`}
  - `kind` ∈ {`finite`, `endless`}
  - `source_type` — source-specific (`album`/`playlist`/`artist`/`starred` for
    Subsonic; `item` for Audiobookshelf)

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
    {"filepath": "http://<bridge-base>/track/<source>/<...ids>.mp3", "name": "<title>"}
  ]
}
```

**Generated endless playlist shape:** identical, but a single entry pointing at
`http://<bridge-base>/radio/<mappingId>.mp3`.

> Playlist file extension defaults to `.tap`; confirm against TeddyCloud's
> expected convention during implementation and make it configurable if needed.

### 4. `streaming` — audio proxy + radio
- `GET /track/{source}/{...ids}.mp3` → looks up the source, calls
  `open_stream(track, range)`, and pipes bytes through (passthrough; forwards
  Range when the upstream supports it — Subsonic transcoded MP3, ABS direct file).
  On EOF the track ends and TeddyCloud advances. URL shapes:
  `/track/subsonic/{trackId}.mp3` and `/track/abs/{itemId}/{ino}.mp3`.
- `GET /radio/{mappingId}.mp3` → resolves the mapping's track set (optionally
  shuffled) and streams track after track through the mapping's source, looping
  forever, never closing. On a failed track fetch, skip to the next rather than
  killing the stream.
- Depends on: `sources`, `storage`.

### 5. `web` — UI + JSON API
- FastAPI routes: JSON API for browse/search, CRUD on mappings, "regenerate
  playlist", and a settings / test-connection endpoint. Serves the UI.
- **UI:** server-rendered Jinja2 + HTMX (no build step, small footprint).
- See the **Web UI** section for the full screen-by-screen design and visual
  language.

### 6. `config` — settings & wiring
- Loads config from env vars (bootstrap) overlaid by values saved in `storage`.
- Holds: per-source connection config (Subsonic URL/credentials; Audiobookshelf
  URL + token or username/password), TeddyCloud library path, bridge public base
  URL, MP3 bitrate, bind address/port.

**Cross-cutting:** logging, and a thin `models` module for shared dataclasses.

## Data Flow

### A. Creating a finite mapping (Subsonic album/playlist, or ABS audiobook)
1. User picks a source, browses it in the UI, picks an item, clicks "Create Tonie
   playlist", names it, picks mode = finite.
2. `web` calls `source.resolve_tracks(ref)` → ordered track list.
3. `storage.save()` persists the mapping record (incl. `source`).
4. `playlist` builds the JSON and writes it atomically into the mounted library dir.
5. UI shows success + the file location so the user can assign it to a tag in
   TeddyCloud.

### B. Playback of a finite mapping
1. Box taps tag → TeddyCloud reads the playlist file → iterates `files[]`.
2. For each entry, TeddyCloud fetches `GET /track/{source}/…​.mp3` from the bridge.
3. `streaming` resolves the source and calls `open_stream(...)`, piping bytes; on
   EOF, TeddyCloud advances to the next entry.
4. Titles/skip/resume handled natively by TeddyCloud from the playlist entries.

### C. Endless radio mapping (Subsonic artist/starred)
1. User picks an artist or "starred", mode = endless, optional shuffle.
2. `playlist` writes a playlist file with a single entry →
   `http://<bridge-base>/radio/<mappingId>.mp3`.
3. On play, `streaming` resolves the track set, optionally shuffles, and streams
   track after track through the source, looping forever.

### D. Regeneration / edit / delete
- Edit name/mode/shuffle → update record, rewrite playlist file.
- Delete → remove playlist file from the mounted dir + delete the record.
- "Regenerate" re-resolves tracks from the source and rewrites the file (finite
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
  panel** at the bottom showing a dot per connected source (Subsonic and/or
  Audiobookshelf) plus Library (mounted). Only configured sources appear.
- **Main area:** page title + contextual content.

### Library screen (browse + create)

- A **source selector** (segmented control) above the tabs when more than one
  source is connected — e.g. Subsonic / Audiobookshelf. With one source it's
  hidden.
- A **search box** and **type tabs** that adapt to the selected source:
  - Subsonic → Albums · Artists · Playlists · Starred.
  - Audiobookshelf → its book-type libraries (e.g. Hörbücher · Hörspiele).
- **Cover grid** of cards (cover art, title, subtitle). Hovering a card lifts it
  and reveals a **＋ Tonie** button — the entry point to the create flow.

### Create Tonie drawer

Slides in from the right over a dimmed library.

- **Source summary:** cover, title, subtitle, and quick facts — track count and
  total duration (one extra source call when the drawer opens; accepted).
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
  (📖 Finite / 📻 Endless·shuffle), a small **source badge** (Subsonic / ABS), a
  summary line (source type · track count · duration / "resolves live" for
  endless), and the `lib://…` path with a **copy** button.
- **Per-row actions:** ↻ **Regenerate** (re-resolve from the source and rewrite
  the file — finite only), ✎ **Edit** (rename / change mode / shuffle), 🗑 **Delete**
  (removes the record *and* the `.tap` file).

### Settings screen

Grouped panels with a single Save / Reset action bar. Each source panel works
independently — connect one or both:

1. **Subsonic server:** URL, username, password, **Test connection** button with
   live status (server type/version, album count). Bootstraps from env vars if
   set; UI values persist in SQLite and take over.
2. **Audiobookshelf server:** URL, an **API token** *or* username/password,
   **Test connection** button with live status (version, library/item counts).
   Same env-bootstrap-then-UI behaviour.
3. **TeddyCloud library:** mounted library path shown **read-only** with a
   mounted/writable health check (it's a Docker mount — editing here wouldn't
   remount anything); plus the playlist file extension (default `.tap`).
4. **Bridge address:** the base URL TeddyCloud uses to reach the audio proxy —
   baked into every playlist entry.
5. **Audio & defaults:** MP3 bitrate, default mode for new Tonies, and a
   "shuffle endless radios by default" toggle.

> **Note on credentials:** source credentials (Subsonic password; Audiobookshelf
> token or password) are stored in the bridge's SQLite volume — needed to
> authenticate upstream. Acceptable under the trusted-LAN assumption.

## Configuration

Env bootstrap, overridable in the UI, stored in SQLite. All source vars are
optional — connect whichever source(s) you use:

- `SUBSONIC_URL`, `SUBSONIC_USER`, `SUBSONIC_PASSWORD`
- `ABS_URL`, and either `ABS_TOKEN` or `ABS_USER` / `ABS_PASSWORD`
- `TEDDYCLOUD_LIBRARY_PATH` (mounted dir where `.tap` files are written)
- `BRIDGE_BASE_URL` (what TeddyCloud uses to reach the proxy, e.g.
  `http://bridge:8080`)
- `MP3_BITRATE` (default 192; Subsonic transcode target — ABS files pass through)
- `BIND_ADDR` / `PORT`

## Error Handling

- **Source unreachable / auth fail** (Subsonic or ABS): surfaced via that
  source's "Test connection" action; browse calls show a clear error, never a
  stack trace. A failure in one source doesn't affect the other.
- **Track fetch fails mid-playback:** logged; endless mode skips to the next
  track; finite mode ends the track early and TeddyCloud advances.
- **Library path not writable / not mounted:** startup check + clear UI banner;
  playlist creation refuses with a helpful message.
- **Bad/empty source** (e.g. album with 0 tracks): refuse to create the mapping
  with a validation message.
- **Atomic writes:** temp file + rename so TeddyCloud never reads a half-written
  playlist.

## Testing

- **Unit:** both source implementations against the `ContentSource` interface
  (mock httpx — Subsonic URL/auth-param construction; ABS login + browse +
  `contentUrl` resolution + Range forwarding); `playlist` generator (exact
  `type:tap` JSON shape, slug, atomic write, source-encoded proxy URLs);
  `storage` repository (in-memory SQLite).
- **Integration:** FastAPI `TestClient` over the JSON API — create/list/edit/
  delete mapping per source → assert the right file appears in a temp "library"
  dir with correct contents; proxy endpoint streams from a stubbed source.
- **Manual/E2E checklist:** real TeddyCloud with each source — Subsonic album +
  endless artist radio; ABS audiobook (multi-file) → confirm playback + skip +
  resume on the box.
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
- **ABS single-file audiobooks:** an item with one audio file (chapters embedded
  in a single M4B/MP3) becomes a single playlist entry — TeddyCloud plays it as
  one long track (resume works; no per-chapter skip). Multi-file items split into
  proper tracks. Chapter-splitting a single file is out of scope for v1.
