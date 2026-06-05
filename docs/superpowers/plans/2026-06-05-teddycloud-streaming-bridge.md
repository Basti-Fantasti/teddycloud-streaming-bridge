# TeddyCloud Streaming Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone FastAPI service that makes audio from Subsonic and Audiobookshelf playable on a Toniebox by writing TeddyCloud playlist files and proxying per-track MP3 audio.

**Architecture:** A `ContentSource` interface with two implementations (`subsonic`, `audiobookshelf`) is consumed by source-agnostic modules: a SQLite `storage` repository, a `playlist` generator that writes TeddyCloud `.tap` files into a mounted volume, a `streaming` proxy that pipes per-track audio, and a Jinja2 + HTMX web UI. Sources talk to upstream servers over async `httpx`; storage and playlist I/O are synchronous.

**Tech Stack:** Python 3.11+, FastAPI, Uvicorn, httpx (async), Jinja2, HTMX, SQLite (stdlib `sqlite3`), pytest + pytest-asyncio, Docker.

**Reference:** The approved spec is `docs/superpowers/specs/2026-06-05-teddycloud-subsonic-bridge-design.md`. The approved UI mockups (cozy visual language, exact CSS/fonts/colors) are in the workspace at `.superpowers/brainstorm/197-1780688864/content/` (`library.html`, `settings.html`, `create-tonie.html`, `tonies.html`) — they are the authoritative styling reference for the template tasks.

---

## File Structure

```
teddycloud-streaming-bridge/
├── pyproject.toml                 # deps + pytest config
├── README.md                      # setup, trusted-LAN note, config table
├── Dockerfile                     # slim python image
├── docker-compose.example.yml     # bridge + teddycloud volumes
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app factory, route wiring, startup checks
│   ├── models.py                  # dataclasses: Track, BrowseEntry, SourceStatus, Mapping, StreamResponse
│   ├── storage.py                 # SQLite repo: settings + mappings (sync)
│   ├── config.py                  # ConfigService: storage values overlaid on env
│   ├── playlist.py                # slugify, build/write/delete .tap files (atomic)
│   ├── streaming.py               # /track/* and /radio/* proxy router
│   ├── sources/
│   │   ├── __init__.py
│   │   ├── base.py                # ContentSource ABC
│   │   ├── subsonic.py            # Subsonic implementation
│   │   ├── audiobookshelf.py      # Audiobookshelf implementation
│   │   └── registry.py            # build configured sources from ConfigService
│   └── web/
│       ├── __init__.py
│       ├── api.py                 # JSON API router
│       ├── views.py               # HTML page router (Jinja2)
│       ├── templates/
│       │   ├── base.html
│       │   ├── library.html
│       │   ├── tonies.html
│       │   ├── settings.html
│       │   └── partials/
│       │       ├── grid.html
│       │       └── tonie_row.html
│       └── static/
│           ├── app.css
│           ├── app.js
│           └── htmx.min.js
└── tests/
    ├── conftest.py
    ├── test_models.py
    ├── test_storage.py
    ├── test_config.py
    ├── test_subsonic.py
    ├── test_audiobookshelf.py
    ├── test_registry.py
    ├── test_playlist.py
    ├── test_streaming.py
    ├── test_api.py
    └── test_views.py
```

**Key design decisions locked in here:**
- `sources` is its own package; each implementation is one focused file.
- `storage` and `playlist` are synchronous (SQLite + file I/O are fast and local); FastAPI runs sync calls in its threadpool. Only `sources` (network) are async.
- Sources accept an injected `httpx.AsyncClient` so tests use `httpx.MockTransport` with no real network.

---

## Task 1: Project scaffolding

**Files:**
- Create: `pyproject.toml`
- Create: `app/__init__.py` (empty)
- Create: `app/sources/__init__.py` (empty)
- Create: `app/web/__init__.py` (empty)
- Create: `tests/__init__.py` (empty)
- Create: `tests/conftest.py`
- Create: `tests/test_smoke.py`

- [ ] **Step 1: Create `pyproject.toml`**

```toml
[project]
name = "teddycloud-streaming-bridge"
version = "0.1.0"
description = "Bridge Subsonic and Audiobookshelf audio to TeddyCloud playlists"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.29",
    "httpx>=0.27",
    "jinja2>=3.1",
    "python-multipart>=0.0.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.setuptools.packages.find]
include = ["app*"]
```

- [ ] **Step 2: Create the empty package files**

Create `app/__init__.py`, `app/sources/__init__.py`, `app/web/__init__.py`, `tests/__init__.py` — all empty.

- [ ] **Step 3: Create `tests/conftest.py`** (shared fixtures used by later tasks)

```python
import sqlite3
import pytest


@pytest.fixture
def db() -> sqlite3.Connection:
    """In-memory SQLite connection with schema applied."""
    from app.storage import Storage
    conn = sqlite3.connect(":memory:")
    conn.row_factory = sqlite3.Row
    Storage.init_schema(conn)
    return conn


@pytest.fixture
def storage(db):
    from app.storage import Storage
    return Storage(db)
```

- [ ] **Step 4: Write a smoke test** in `tests/test_smoke.py`

```python
def test_app_package_imports():
    import app
    assert app is not None
```

- [ ] **Step 5: Install and run**

Run: `python -m pip install -e ".[dev]"`
Then: `python -m pytest tests/test_smoke.py -v`
Expected: PASS (1 passed)

- [ ] **Step 6: Create `.gitignore` additions if missing** — ensure these lines exist: `__pycache__/`, `*.db`, `.superpowers/`, `data/` (the repo already has a `.gitignore`; only add missing entries).

- [ ] **Step 7: Commit**

```bash
git add pyproject.toml app tests .gitignore
git commit -m "chore: project scaffolding and pytest setup"
```

---

## Task 2: Domain models

**Files:**
- Create: `app/models.py`
- Test: `tests/test_models.py`

- [ ] **Step 1: Write the failing test**

```python
from app.models import Track, BrowseEntry, SourceStatus, Mapping


def test_track_proxy_path_subsonic():
    t = Track(source="subsonic", ids=["123"], title="Song", duration=200.0)
    assert t.proxy_path == "subsonic/123"


def test_track_proxy_path_abs_multi_id():
    t = Track(source="audiobookshelf", ids=["itm1", "19388"], title="Ch1", duration=26.6)
    assert t.proxy_path == "audiobookshelf/itm1/19388"


def test_browse_entry_defaults():
    e = BrowseEntry(id="al1", kind="album", title="Album")
    assert e.subtitle is None and e.cover_url is None


def test_source_status_ok():
    s = SourceStatus(ok=True, name="subsonic", server="Navidrome 0.52", detail="128 albums")
    assert s.ok and s.error is None


def test_mapping_roundtrip_fields():
    m = Mapping(id="m1", name="Grüffelo", source="audiobookshelf", kind="finite",
                source_type="item", source_id="itm1", shuffle=False,
                playlist_path="/lib/gruffelo.tap", created_at="t", updated_at="t")
    assert m.source == "audiobookshelf" and m.kind == "finite"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_models.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.models'`

- [ ] **Step 3: Implement `app/models.py`**

```python
from __future__ import annotations

from collections.abc import AsyncIterator
from dataclasses import dataclass, field


@dataclass
class Track:
    """One playable track resolved from a source."""
    source: str            # "subsonic" | "audiobookshelf"
    ids: list[str]         # source-specific id components, e.g. ["trackId"] or ["itemId", "ino"]
    title: str
    duration: float | None = None

    @property
    def proxy_path(self) -> str:
        """Path used inside the bridge proxy URL, without the .mp3 suffix."""
        return "/".join([self.source, *self.ids])


@dataclass
class BrowseEntry:
    """One pickable/navigable thing returned by ContentSource.browse()."""
    id: str
    kind: str                       # source-specific (album/artist/playlist/starred/library/item)
    title: str
    subtitle: str | None = None
    cover_url: str | None = None
    is_container: bool = False      # True if it can be drilled into (artist, library)


@dataclass
class SourceStatus:
    ok: bool
    name: str
    server: str | None = None
    detail: str | None = None
    error: str | None = None


@dataclass
class StreamResponse:
    """Upstream byte stream for one track, ready to relay to the client."""
    status_code: int
    media_type: str
    headers: dict[str, str]
    body: AsyncIterator[bytes]


@dataclass
class Mapping:
    id: str
    name: str
    source: str
    kind: str               # "finite" | "endless"
    source_type: str
    source_id: str
    shuffle: bool
    playlist_path: str
    created_at: str
    updated_at: str
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_models.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add app/models.py tests/test_models.py
git commit -m "feat: domain models (Track, BrowseEntry, SourceStatus, Mapping, StreamResponse)"
```

---

## Task 3: SQLite storage repository

**Files:**
- Create: `app/storage.py`
- Test: `tests/test_storage.py`

- [ ] **Step 1: Write the failing test**

```python
from app.models import Mapping


def _mapping(**kw):
    base = dict(id="m1", name="Grüffelo", source="subsonic", kind="finite",
                source_type="album", source_id="al1", shuffle=False,
                playlist_path="/lib/gruffelo.tap", created_at="2026-01-01",
                updated_at="2026-01-01")
    base.update(kw)
    return Mapping(**base)


def test_settings_set_get(storage):
    assert storage.get_setting("subsonic.url") is None
    storage.set_setting("subsonic.url", "https://music")
    assert storage.get_setting("subsonic.url") == "https://music"


def test_settings_overwrite(storage):
    storage.set_setting("k", "a")
    storage.set_setting("k", "b")
    assert storage.get_setting("k") == "b"


def test_mapping_save_and_get(storage):
    storage.save_mapping(_mapping())
    got = storage.get_mapping("m1")
    assert got is not None and got.name == "Grüffelo" and got.source == "subsonic"


def test_mapping_list_and_delete(storage):
    storage.save_mapping(_mapping(id="m1"))
    storage.save_mapping(_mapping(id="m2", name="Maus"))
    assert {m.id for m in storage.list_mappings()} == {"m1", "m2"}
    storage.delete_mapping("m1")
    assert storage.get_mapping("m1") is None
    assert {m.id for m in storage.list_mappings()} == {"m2"}


def test_mapping_save_is_upsert(storage):
    storage.save_mapping(_mapping(name="old"))
    storage.save_mapping(_mapping(name="new"))
    assert storage.get_mapping("m1").name == "new"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_storage.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.storage'`

- [ ] **Step 3: Implement `app/storage.py`**

```python
from __future__ import annotations

import sqlite3
from dataclasses import asdict

from app.models import Mapping

_SCHEMA = """
CREATE TABLE IF NOT EXISTS settings (
    key   TEXT PRIMARY KEY,
    value TEXT
);
CREATE TABLE IF NOT EXISTS mappings (
    id            TEXT PRIMARY KEY,
    name          TEXT NOT NULL,
    source        TEXT NOT NULL,
    kind          TEXT NOT NULL,
    source_type   TEXT NOT NULL,
    source_id     TEXT NOT NULL,
    shuffle       INTEGER NOT NULL DEFAULT 0,
    playlist_path TEXT NOT NULL,
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);
"""

_MAPPING_COLUMNS = [
    "id", "name", "source", "kind", "source_type", "source_id",
    "shuffle", "playlist_path", "created_at", "updated_at",
]


class Storage:
    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn
        self.conn.row_factory = sqlite3.Row

    @staticmethod
    def init_schema(conn: sqlite3.Connection) -> None:
        conn.executescript(_SCHEMA)
        conn.commit()

    @classmethod
    def open(cls, path: str) -> "Storage":
        conn = sqlite3.connect(path, check_same_thread=False)
        cls.init_schema(conn)
        return cls(conn)

    # --- settings ---
    def get_setting(self, key: str) -> str | None:
        row = self.conn.execute(
            "SELECT value FROM settings WHERE key = ?", (key,)
        ).fetchone()
        return row["value"] if row else None

    def set_setting(self, key: str, value: str) -> None:
        self.conn.execute(
            "INSERT INTO settings(key, value) VALUES(?, ?) "
            "ON CONFLICT(key) DO UPDATE SET value = excluded.value",
            (key, value),
        )
        self.conn.commit()

    # --- mappings ---
    def save_mapping(self, m: Mapping) -> None:
        data = asdict(m)
        data["shuffle"] = 1 if m.shuffle else 0
        placeholders = ", ".join("?" for _ in _MAPPING_COLUMNS)
        updates = ", ".join(f"{c}=excluded.{c}" for c in _MAPPING_COLUMNS if c != "id")
        self.conn.execute(
            f"INSERT INTO mappings ({', '.join(_MAPPING_COLUMNS)}) "
            f"VALUES ({placeholders}) "
            f"ON CONFLICT(id) DO UPDATE SET {updates}",
            [data[c] for c in _MAPPING_COLUMNS],
        )
        self.conn.commit()

    def get_mapping(self, mapping_id: str) -> Mapping | None:
        row = self.conn.execute(
            "SELECT * FROM mappings WHERE id = ?", (mapping_id,)
        ).fetchone()
        return self._row_to_mapping(row) if row else None

    def list_mappings(self) -> list[Mapping]:
        rows = self.conn.execute(
            "SELECT * FROM mappings ORDER BY created_at, id"
        ).fetchall()
        return [self._row_to_mapping(r) for r in rows]

    def delete_mapping(self, mapping_id: str) -> None:
        self.conn.execute("DELETE FROM mappings WHERE id = ?", (mapping_id,))
        self.conn.commit()

    @staticmethod
    def _row_to_mapping(row: sqlite3.Row) -> Mapping:
        return Mapping(
            id=row["id"], name=row["name"], source=row["source"], kind=row["kind"],
            source_type=row["source_type"], source_id=row["source_id"],
            shuffle=bool(row["shuffle"]), playlist_path=row["playlist_path"],
            created_at=row["created_at"], updated_at=row["updated_at"],
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_storage.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add app/storage.py tests/test_storage.py tests/conftest.py
git commit -m "feat: SQLite storage repository for settings and mappings"
```

---

## Task 4: ConfigService (env overlaid by storage)

**Files:**
- Create: `app/config.py`
- Test: `tests/test_config.py`

Stored settings win over env vars. Source config is returned as a dict or `None` when not configured (missing required keys).

- [ ] **Step 1: Write the failing test**

```python
import pytest
from app.config import ConfigService


def test_value_prefers_storage_over_env(storage, monkeypatch):
    monkeypatch.setenv("MP3_BITRATE", "128")
    cfg = ConfigService(storage)
    assert cfg.value("MP3_BITRATE", "mp3_bitrate") == "128"
    storage.set_setting("mp3_bitrate", "256")
    assert cfg.value("MP3_BITRATE", "mp3_bitrate") == "256"


def test_subsonic_config_none_when_unset(storage, monkeypatch):
    monkeypatch.delenv("SUBSONIC_URL", raising=False)
    cfg = ConfigService(storage)
    assert cfg.subsonic_config() is None


def test_subsonic_config_from_env(storage, monkeypatch):
    monkeypatch.setenv("SUBSONIC_URL", "https://music")
    monkeypatch.setenv("SUBSONIC_USER", "teddy")
    monkeypatch.setenv("SUBSONIC_PASSWORD", "secret")
    cfg = ConfigService(storage)
    assert cfg.subsonic_config() == {
        "url": "https://music", "user": "teddy", "password": "secret"
    }


def test_abs_config_requires_url_and_auth(storage, monkeypatch):
    for k in ["ABS_URL", "ABS_TOKEN", "ABS_USER", "ABS_PASSWORD"]:
        monkeypatch.delenv(k, raising=False)
    cfg = ConfigService(storage)
    assert cfg.abs_config() is None
    storage.set_setting("audiobookshelf.url", "https://abs")
    storage.set_setting("audiobookshelf.token", "tok")
    assert cfg.abs_config() == {"url": "https://abs", "token": "tok",
                                "user": None, "password": None}


def test_library_path_default(storage, monkeypatch):
    monkeypatch.delenv("TEDDYCLOUD_LIBRARY_PATH", raising=False)
    cfg = ConfigService(storage)
    assert cfg.library_path() == "/teddycloud/library"


def test_mp3_bitrate_default_int(storage, monkeypatch):
    monkeypatch.delenv("MP3_BITRATE", raising=False)
    cfg = ConfigService(storage)
    assert cfg.mp3_bitrate() == 192
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.config'`

- [ ] **Step 3: Implement `app/config.py`**

```python
from __future__ import annotations

import os

from app.storage import Storage

_DEFAULT_LIBRARY_PATH = "/teddycloud/library"
_DEFAULT_BRIDGE_BASE_URL = "http://localhost:8080"
_DEFAULT_BITRATE = 192
_DEFAULT_EXTENSION = ".tap"


class ConfigService:
    """Reads config: a stored setting (UI) overrides the env var (bootstrap)."""

    def __init__(self, storage: Storage):
        self.storage = storage

    def value(self, env_key: str, setting_key: str, default: str | None = None) -> str | None:
        stored = self.storage.get_setting(setting_key)
        if stored is not None and stored != "":
            return stored
        return os.environ.get(env_key, default)

    def subsonic_config(self) -> dict | None:
        url = self.value("SUBSONIC_URL", "subsonic.url")
        user = self.value("SUBSONIC_USER", "subsonic.user")
        password = self.value("SUBSONIC_PASSWORD", "subsonic.password")
        if not (url and user and password):
            return None
        return {"url": url, "user": user, "password": password}

    def abs_config(self) -> dict | None:
        url = self.value("ABS_URL", "audiobookshelf.url")
        token = self.value("ABS_TOKEN", "audiobookshelf.token")
        user = self.value("ABS_USER", "audiobookshelf.user")
        password = self.value("ABS_PASSWORD", "audiobookshelf.password")
        if not url or not (token or (user and password)):
            return None
        return {"url": url, "token": token, "user": user, "password": password}

    def library_path(self) -> str:
        return self.value("TEDDYCLOUD_LIBRARY_PATH", "library_path", _DEFAULT_LIBRARY_PATH)

    def bridge_base_url(self) -> str:
        return self.value("BRIDGE_BASE_URL", "bridge_base_url", _DEFAULT_BRIDGE_BASE_URL).rstrip("/")

    def playlist_extension(self) -> str:
        return self.value("PLAYLIST_EXTENSION", "playlist_extension", _DEFAULT_EXTENSION)

    def mp3_bitrate(self) -> int:
        return int(self.value("MP3_BITRATE", "mp3_bitrate", str(_DEFAULT_BITRATE)))

    def default_shuffle(self) -> bool:
        return self.value("DEFAULT_SHUFFLE", "default_shuffle", "false").lower() == "true"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_config.py -v`
Expected: PASS (6 passed)

- [ ] **Step 5: Commit**

```bash
git add app/config.py tests/test_config.py
git commit -m "feat: ConfigService overlaying stored settings on env vars"
```

---

## Task 5: ContentSource interface

**Files:**
- Create: `app/sources/base.py`
- Test: `tests/test_sources_base.py`

The interface is an abstract base class. The test defines a tiny concrete subclass to prove the contract is usable.

- [ ] **Step 1: Write the failing test**

```python
import pytest
from app.models import BrowseEntry, SourceStatus, Track, StreamResponse
from app.sources.base import ContentSource


class DummySource(ContentSource):
    name = "dummy"

    async def test_connection(self):
        return SourceStatus(ok=True, name=self.name)

    async def browse(self, kind=None, parent_id=None, query=None):
        return [BrowseEntry(id="x", kind="album", title="X")]

    async def resolve_tracks(self, source_type, source_id):
        return [Track(source=self.name, ids=["1"], title="t")]

    async def open_stream(self, ids, range_header=None):
        async def gen():
            yield b"data"
        return StreamResponse(200, "audio/mpeg", {}, gen())

    def cover_url(self, source_type, source_id):
        return None


async def test_dummy_source_satisfies_interface():
    s = DummySource()
    assert (await s.test_connection()).ok is True
    assert (await s.browse())[0].id == "x"
    assert (await s.resolve_tracks("album", "1"))[0].title == "t"
    sr = await s.open_stream(["1"])
    chunks = [c async for c in sr.body]
    assert chunks == [b"data"]


def test_cannot_instantiate_abstract():
    with pytest.raises(TypeError):
        ContentSource()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_sources_base.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.sources.base'`

- [ ] **Step 3: Implement `app/sources/base.py`**

```python
from __future__ import annotations

from abc import ABC, abstractmethod

from app.models import BrowseEntry, SourceStatus, StreamResponse, Track


class ContentSource(ABC):
    """Common interface every content source implements."""

    name: str = "base"

    @abstractmethod
    async def test_connection(self) -> SourceStatus: ...

    @abstractmethod
    async def browse(
        self,
        kind: str | None = None,
        parent_id: str | None = None,
        query: str | None = None,
    ) -> list[BrowseEntry]: ...

    @abstractmethod
    async def resolve_tracks(self, source_type: str, source_id: str) -> list[Track]: ...

    @abstractmethod
    async def open_stream(self, ids: list[str], range_header: str | None = None) -> StreamResponse: ...

    @abstractmethod
    def cover_url(self, source_type: str, source_id: str) -> str | None: ...
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_sources_base.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add app/sources/base.py tests/test_sources_base.py
git commit -m "feat: ContentSource abstract interface"
```

---

## Task 6: Subsonic source implementation

**Files:**
- Create: `app/sources/subsonic.py`
- Test: `tests/test_subsonic.py`

Subsonic token auth: `t = md5(password + salt)`, plus `s=salt`, `u`, `v=1.16.1`, `c=teddybridge`, `f=json`. Tests inject an `httpx.MockTransport` so no network is used. The salt is fixed via an injectable callable for determinism.

- [ ] **Step 1: Write the failing test**

```python
import hashlib
import httpx
import pytest
from app.sources.subsonic import SubsonicSource


def _client(handler):
    return httpx.AsyncClient(transport=httpx.MockTransport(handler))


def _ok(body):
    return {"subsonic-response": {"status": "ok", "version": "1.16.1", **body}}


async def test_auth_params_use_token(monkeypatch):
    seen = {}

    def handler(request):
        seen.update(dict(request.url.params))
        seen["path"] = request.url.path
        return httpx.Response(200, json=_ok({"albumList2": {"album": []}}))

    src = SubsonicSource({"url": "https://music", "user": "teddy", "password": "secret"},
                         client=_client(handler), salt="abc")
    await src.browse(kind="albums")
    assert seen["u"] == "teddy"
    assert seen["s"] == "abc"
    assert seen["t"] == hashlib.md5(b"secretabc").hexdigest()
    assert seen["f"] == "json"
    assert seen["path"].endswith("/rest/getAlbumList2")


async def test_browse_albums_maps_entries():
    def handler(request):
        return httpx.Response(200, json=_ok({"albumList2": {"album": [
            {"id": "al1", "name": "Grüffelo", "artist": "Donaldson", "coverArt": "al1"},
        ]}}))

    src = SubsonicSource({"url": "https://m", "user": "u", "password": "p"},
                         client=_client(handler), salt="s")
    entries = await src.browse(kind="albums")
    assert entries[0].id == "al1"
    assert entries[0].title == "Grüffelo"
    assert entries[0].subtitle == "Donaldson"
    assert entries[0].kind == "album"


async def test_resolve_tracks_from_album():
    def handler(request):
        return httpx.Response(200, json=_ok({"album": {"song": [
            {"id": "s1", "title": "One", "duration": 100},
            {"id": "s2", "title": "Two", "duration": 200},
        ]}}))

    src = SubsonicSource({"url": "https://m", "user": "u", "password": "p"},
                         client=_client(handler), salt="s")
    tracks = await src.resolve_tracks("album", "al1")
    assert [t.ids for t in tracks] == [["s1"], ["s2"]]
    assert tracks[0].source == "subsonic"
    assert tracks[1].title == "Two"


async def test_open_stream_passes_range_and_format():
    seen = {}

    def handler(request):
        seen["params"] = dict(request.url.params)
        seen["range"] = request.headers.get("range")
        return httpx.Response(206, content=b"audiobytes",
                              headers={"Content-Type": "audio/mpeg"})

    src = SubsonicSource({"url": "https://m", "user": "u", "password": "p"},
                         client=_client(handler), salt="s")
    sr = await src.open_stream(["s1"], range_header="bytes=0-9")
    assert sr.status_code == 206
    assert sr.media_type == "audio/mpeg"
    assert seen["params"]["id"] == "s1"
    assert seen["params"]["format"] == "mp3"
    assert seen["range"] == "bytes=0-9"
    chunks = [c async for c in sr.body]
    assert b"".join(chunks) == b"audiobytes"


async def test_test_connection_ok():
    def handler(request):
        return httpx.Response(200, json=_ok({"albumList2": {"album": [{"id": "a"}]}}))

    src = SubsonicSource({"url": "https://m", "user": "u", "password": "p"},
                         client=_client(handler), salt="s")
    status = await src.test_connection()
    assert status.ok is True and status.name == "subsonic"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_subsonic.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.sources.subsonic'`

- [ ] **Step 3: Implement `app/sources/subsonic.py`**

```python
from __future__ import annotations

import hashlib
import secrets

import httpx

from app.models import BrowseEntry, SourceStatus, StreamResponse, Track
from app.sources.base import ContentSource

_API_VERSION = "1.16.1"
_CLIENT_NAME = "teddybridge"


class SubsonicSource(ContentSource):
    name = "subsonic"

    def __init__(self, config: dict, client: httpx.AsyncClient | None = None,
                 salt: str | None = None, bitrate: int = 192):
        self.url = config["url"].rstrip("/")
        self.user = config["user"]
        self.password = config["password"]
        self.bitrate = bitrate
        self._client = client or httpx.AsyncClient(timeout=30.0)
        self._salt = salt

    def _auth_params(self) -> dict:
        salt = self._salt or secrets.token_hex(8)
        token = hashlib.md5(f"{self.password}{salt}".encode()).hexdigest()
        return {"u": self.user, "t": token, "s": salt,
                "v": _API_VERSION, "c": _CLIENT_NAME, "f": "json"}

    async def _get(self, method: str, params: dict | None = None) -> dict:
        url = f"{self.url}/rest/{method}"
        all_params = {**self._auth_params(), **(params or {})}
        resp = await self._client.get(url, params=all_params)
        resp.raise_for_status()
        return resp.json()["subsonic-response"]

    async def test_connection(self) -> SourceStatus:
        try:
            data = await self._get("getAlbumList2", {"type": "newest", "size": 1})
            version = data.get("version")
            return SourceStatus(ok=True, name=self.name,
                                server=f"Subsonic {version}" if version else "Subsonic",
                                detail="connected")
        except Exception as exc:  # noqa: BLE001 - surfaced to UI
            return SourceStatus(ok=False, name=self.name, error=str(exc))

    async def browse(self, kind=None, parent_id=None, query=None):
        kind = kind or "albums"
        if query:
            data = await self._get("search3", {"query": query, "songCount": 0})
            result = data.get("searchResult3", {})
            albums = result.get("album", [])
            return [self._album_entry(a) for a in albums]
        if kind == "albums":
            data = await self._get("getAlbumList2", {"type": "alphabeticalByName", "size": 500})
            return [self._album_entry(a) for a in data.get("albumList2", {}).get("album", [])]
        if kind == "artists":
            data = await self._get("getArtists")
            out: list[BrowseEntry] = []
            for index in data.get("artists", {}).get("index", []):
                for artist in index.get("artist", []):
                    out.append(BrowseEntry(id=artist["id"], kind="artist",
                                           title=artist.get("name", ""), is_container=True))
            return out
        if kind == "playlists":
            data = await self._get("getPlaylists")
            return [BrowseEntry(id=p["id"], kind="playlist", title=p.get("name", ""))
                    for p in data.get("playlists", {}).get("playlist", [])]
        if kind == "starred":
            data = await self._get("getStarred2")
            songs = data.get("starred2", {}).get("song", [])
            return [BrowseEntry(id="starred", kind="starred",
                                title="Starred", subtitle=f"{len(songs)} tracks")]
        return []

    def _album_entry(self, a: dict) -> BrowseEntry:
        return BrowseEntry(id=a["id"], kind="album", title=a.get("name", ""),
                           subtitle=a.get("artist"),
                           cover_url=self.cover_url("album", a["id"]) if a.get("coverArt") else None)

    async def resolve_tracks(self, source_type, source_id):
        if source_type == "album":
            data = await self._get("getAlbum", {"id": source_id})
            songs = data.get("album", {}).get("song", [])
        elif source_type == "playlist":
            data = await self._get("getPlaylist", {"id": source_id})
            songs = data.get("playlist", {}).get("entry", [])
        elif source_type == "artist":
            songs = await self._artist_songs(source_id)
        elif source_type == "starred":
            data = await self._get("getStarred2")
            songs = data.get("starred2", {}).get("song", [])
        else:
            songs = []
        return [Track(source=self.name, ids=[s["id"]], title=s.get("title", ""),
                      duration=s.get("duration")) for s in songs]

    async def _artist_songs(self, artist_id: str) -> list[dict]:
        data = await self._get("getArtist", {"id": artist_id})
        songs: list[dict] = []
        for album in data.get("artist", {}).get("album", []):
            adata = await self._get("getAlbum", {"id": album["id"]})
            songs.extend(adata.get("album", {}).get("song", []))
        return songs

    async def open_stream(self, ids, range_header=None):
        params = {**self._auth_params(), "id": ids[0],
                  "format": "mp3", "maxBitRate": str(self.bitrate)}
        headers = {"Range": range_header} if range_header else {}
        req = self._client.build_request("GET", f"{self.url}/rest/stream",
                                         params=params, headers=headers)
        resp = await self._client.send(req, stream=True)
        return StreamResponse(
            status_code=resp.status_code,
            media_type=resp.headers.get("Content-Type", "audio/mpeg"),
            headers=_relay_headers(resp.headers),
            body=_aiter_closing(resp),
        )

    def cover_url(self, source_type, source_id):
        return None  # covers are proxied lazily; UI can call Subsonic getCoverArt later if needed


def _relay_headers(headers: httpx.Headers) -> dict[str, str]:
    out = {}
    for key in ("Content-Length", "Content-Range", "Accept-Ranges"):
        if key in headers:
            out[key] = headers[key]
    return out


async def _aiter_closing(resp: httpx.Response):
    try:
        async for chunk in resp.aiter_bytes():
            yield chunk
    finally:
        await resp.aclose()
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_subsonic.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add app/sources/subsonic.py tests/test_subsonic.py
git commit -m "feat: Subsonic content source (browse, resolve, stream)"
```

---

## Task 7: Audiobookshelf source implementation

**Files:**
- Create: `app/sources/audiobookshelf.py`
- Test: `tests/test_audiobookshelf.py`

Auth: bearer token (or login with user/password → `user.token`). Browse: `libraries` (book-type) then `items`. resolve_tracks: `GET /api/items/{id}?expanded=1` → `media.tracks[]`, each with `contentUrl=/api/items/{id}/file/{ino}` → `Track(ids=[itemId, ino])`. open_stream: GET that file path, forward Range.

- [ ] **Step 1: Write the failing test**

```python
import httpx
from app.sources.audiobookshelf import AudiobookshelfSource


def _client(handler):
    return httpx.AsyncClient(transport=httpx.MockTransport(handler))


async def test_browse_libraries_filters_book_type():
    def handler(request):
        assert request.headers["authorization"] == "Bearer tok"
        return httpx.Response(200, json={"libraries": [
            {"id": "lib1", "name": "Hörbücher", "mediaType": "book"},
            {"id": "lib2", "name": "Podcasts", "mediaType": "podcast"},
        ]})

    src = AudiobookshelfSource({"url": "https://abs", "token": "tok",
                               "user": None, "password": None}, client=_client(handler))
    entries = await src.browse(kind="libraries")
    assert [(e.id, e.title) for e in entries] == [("lib1", "Hörbücher")]
    assert entries[0].is_container is True


async def test_browse_items_in_library():
    def handler(request):
        assert request.url.path == "/api/libraries/lib1/items"
        return httpx.Response(200, json={"results": [
            {"id": "itm1", "media": {"metadata": {"title": "Pumuckl", "authorName": "X"},
                                     "numAudioFiles": 25, "duration": 100.0}},
        ], "total": 1})

    src = AudiobookshelfSource({"url": "https://abs", "token": "tok",
                               "user": None, "password": None}, client=_client(handler))
    entries = await src.browse(kind="library", parent_id="lib1")
    assert entries[0].id == "itm1"
    assert entries[0].kind == "item"
    assert entries[0].title == "Pumuckl"
    assert "25" in (entries[0].subtitle or "")


async def test_resolve_tracks_uses_contenturl_ino():
    def handler(request):
        assert "expanded" in request.url.params
        return httpx.Response(200, json={"media": {"tracks": [
            {"ino": "100", "title": "Ch1", "duration": 10.0,
             "contentUrl": "/api/items/itm1/file/100"},
            {"ino": "101", "title": "Ch2", "duration": 20.0,
             "contentUrl": "/api/items/itm1/file/101"},
        ]}})

    src = AudiobookshelfSource({"url": "https://abs", "token": "tok",
                               "user": None, "password": None}, client=_client(handler))
    tracks = await src.resolve_tracks("item", "itm1")
    assert [t.ids for t in tracks] == [["itm1", "100"], ["itm1", "101"]]
    assert tracks[0].source == "audiobookshelf"
    assert tracks[1].title == "Ch2"


async def test_open_stream_hits_file_endpoint_with_range():
    seen = {}

    def handler(request):
        seen["path"] = request.url.path
        seen["range"] = request.headers.get("range")
        return httpx.Response(206, content=b"mp3", headers={"Content-Type": "audio/mpeg",
                                                            "Accept-Ranges": "bytes"})

    src = AudiobookshelfSource({"url": "https://abs", "token": "tok",
                               "user": None, "password": None}, client=_client(handler))
    sr = await src.open_stream(["itm1", "100"], range_header="bytes=0-2")
    assert seen["path"] == "/api/items/itm1/file/100"
    assert seen["range"] == "bytes=0-2"
    assert sr.status_code == 206
    assert b"".join([c async for c in sr.body]) == b"mp3"


async def test_login_when_no_token():
    calls = {"login": 0}

    def handler(request):
        if request.url.path == "/login":
            calls["login"] += 1
            return httpx.Response(200, json={"user": {"token": "fresh"}})
        assert request.headers["authorization"] == "Bearer fresh"
        return httpx.Response(200, json={"libraries": []})

    src = AudiobookshelfSource({"url": "https://abs", "token": None,
                               "user": "u", "password": "p"}, client=_client(handler))
    await src.browse(kind="libraries")
    assert calls["login"] == 1
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_audiobookshelf.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.sources.audiobookshelf'`

- [ ] **Step 3: Implement `app/sources/audiobookshelf.py`**

```python
from __future__ import annotations

import httpx

from app.models import BrowseEntry, SourceStatus, StreamResponse, Track
from app.sources.base import ContentSource
from app.sources.subsonic import _aiter_closing, _relay_headers


class AudiobookshelfSource(ContentSource):
    name = "audiobookshelf"

    def __init__(self, config: dict, client: httpx.AsyncClient | None = None):
        self.url = config["url"].rstrip("/")
        self._token = config.get("token")
        self._user = config.get("user")
        self._password = config.get("password")
        self._client = client or httpx.AsyncClient(timeout=30.0)

    async def _ensure_token(self) -> str:
        if self._token:
            return self._token
        resp = await self._client.post(
            f"{self.url}/login",
            json={"username": self._user, "password": self._password},
        )
        resp.raise_for_status()
        self._token = resp.json()["user"]["token"]
        return self._token

    async def _auth_headers(self) -> dict:
        return {"Authorization": f"Bearer {await self._ensure_token()}"}

    async def _get(self, path: str, params: dict | None = None) -> dict:
        resp = await self._client.get(f"{self.url}{path}",
                                      params=params, headers=await self._auth_headers())
        resp.raise_for_status()
        return resp.json()

    async def test_connection(self) -> SourceStatus:
        try:
            data = await self._get("/api/libraries")
            libs = [l for l in data.get("libraries", []) if l.get("mediaType") == "book"]
            return SourceStatus(ok=True, name=self.name, server="Audiobookshelf",
                                detail=f"{len(libs)} book libraries")
        except Exception as exc:  # noqa: BLE001 - surfaced to UI
            return SourceStatus(ok=False, name=self.name, error=str(exc))

    async def browse(self, kind=None, parent_id=None, query=None):
        kind = kind or "libraries"
        if kind == "libraries":
            data = await self._get("/api/libraries")
            return [BrowseEntry(id=l["id"], kind="library", title=l.get("name", ""),
                                is_container=True)
                    for l in data.get("libraries", []) if l.get("mediaType") == "book"]
        if kind == "library":
            params = {"limit": 200}
            if query:
                params["q"] = query
            data = await self._get(f"/api/libraries/{parent_id}/items", params=params)
            return [self._item_entry(it) for it in data.get("results", [])]
        return []

    def _item_entry(self, it: dict) -> BrowseEntry:
        media = it.get("media", {})
        md = media.get("metadata", {})
        n = media.get("numAudioFiles") or media.get("numTracks") or 0
        return BrowseEntry(
            id=it["id"], kind="item", title=md.get("title", ""),
            subtitle=f"{md.get('authorName', '')} · {n} files".strip(" ·"),
            cover_url=self.cover_url("item", it["id"]),
        )

    async def resolve_tracks(self, source_type, source_id):
        data = await self._get(f"/api/items/{source_id}", params={"expanded": 1})
        tracks = data.get("media", {}).get("tracks", [])
        out: list[Track] = []
        for t in tracks:
            ino = str(t["ino"])
            out.append(Track(source=self.name, ids=[source_id, ino],
                             title=t.get("title") or t.get("metadata", {}).get("filename", ino),
                             duration=t.get("duration")))
        return out

    async def open_stream(self, ids, range_header=None):
        item_id, ino = ids[0], ids[1]
        headers = await self._auth_headers()
        if range_header:
            headers["Range"] = range_header
        req = self._client.build_request(
            "GET", f"{self.url}/api/items/{item_id}/file/{ino}", headers=headers)
        resp = await self._client.send(req, stream=True)
        return StreamResponse(
            status_code=resp.status_code,
            media_type=resp.headers.get("Content-Type", "audio/mpeg"),
            headers=_relay_headers(resp.headers),
            body=_aiter_closing(resp),
        )

    def cover_url(self, source_type, source_id):
        return f"{self.url}/api/items/{source_id}/cover"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_audiobookshelf.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add app/sources/audiobookshelf.py tests/test_audiobookshelf.py
git commit -m "feat: Audiobookshelf content source (browse, resolve, stream)"
```

---

## Task 8: Source registry

**Files:**
- Create: `app/sources/registry.py`
- Test: `tests/test_registry.py`

Builds configured sources from `ConfigService`. Only sources with valid config appear.

- [ ] **Step 1: Write the failing test**

```python
from app.sources.registry import SourceRegistry
from app.sources.subsonic import SubsonicSource
from app.sources.audiobookshelf import AudiobookshelfSource
from app.config import ConfigService


def test_registry_lists_only_configured(storage, monkeypatch):
    for k in ["SUBSONIC_URL", "SUBSONIC_USER", "SUBSONIC_PASSWORD",
              "ABS_URL", "ABS_TOKEN", "ABS_USER", "ABS_PASSWORD"]:
        monkeypatch.delenv(k, raising=False)
    storage.set_setting("subsonic.url", "https://m")
    storage.set_setting("subsonic.user", "u")
    storage.set_setting("subsonic.password", "p")
    reg = SourceRegistry(ConfigService(storage))
    assert reg.names() == ["subsonic"]
    assert isinstance(reg.get("subsonic"), SubsonicSource)
    assert reg.get("audiobookshelf") is None


def test_registry_both_sources(storage, monkeypatch):
    storage.set_setting("subsonic.url", "https://m")
    storage.set_setting("subsonic.user", "u")
    storage.set_setting("subsonic.password", "p")
    storage.set_setting("audiobookshelf.url", "https://abs")
    storage.set_setting("audiobookshelf.token", "tok")
    reg = SourceRegistry(ConfigService(storage))
    assert set(reg.names()) == {"subsonic", "audiobookshelf"}
    assert isinstance(reg.get("audiobookshelf"), AudiobookshelfSource)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_registry.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.sources.registry'`

- [ ] **Step 3: Implement `app/sources/registry.py`**

```python
from __future__ import annotations

from app.config import ConfigService
from app.sources.audiobookshelf import AudiobookshelfSource
from app.sources.base import ContentSource
from app.sources.subsonic import SubsonicSource


class SourceRegistry:
    """Builds configured ContentSource instances on demand from current config."""

    def __init__(self, config: ConfigService):
        self.config = config

    def get(self, name: str) -> ContentSource | None:
        if name == "subsonic":
            cfg = self.config.subsonic_config()
            if cfg:
                return SubsonicSource(cfg, bitrate=self.config.mp3_bitrate())
        elif name == "audiobookshelf":
            cfg = self.config.abs_config()
            if cfg:
                return AudiobookshelfSource(cfg)
        return None

    def names(self) -> list[str]:
        return [n for n in ("subsonic", "audiobookshelf") if self.get(n) is not None]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_registry.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add app/sources/registry.py tests/test_registry.py
git commit -m "feat: source registry building configured sources"
```

---

## Task 9: Playlist generator

**Files:**
- Create: `app/playlist.py`
- Test: `tests/test_playlist.py`

Generates the TeddyCloud `.tap` JSON, writes it atomically (temp file + `os.replace`), and deletes it. Finite mappings list one entry per track; endless mappings list a single `/radio/{id}.mp3` entry.

- [ ] **Step 1: Write the failing test**

```python
import json
import os
from app.models import Mapping, Track
from app.playlist import slugify, build_playlist, write_playlist, delete_playlist


def _mapping(**kw):
    base = dict(id="m1", name="Der Grüffelo!", source="subsonic", kind="finite",
                source_type="album", source_id="al1", shuffle=False,
                playlist_path="", created_at="t", updated_at="t")
    base.update(kw)
    return Mapping(**base)


def test_slugify_basic():
    assert slugify("Der Grüffelo!") == "der-gruffelo"
    assert slugify("Die  Maus / Folge 3") == "die-maus-folge-3"


def test_build_finite_playlist_shape():
    tracks = [Track(source="subsonic", ids=["s1"], title="One"),
              Track(source="subsonic", ids=["s2"], title="Two")]
    pl = build_playlist(_mapping(), tracks, "http://bridge:8080", "der-gruffelo", ".tap")
    assert pl["type"] == "tap"
    assert pl["name"] == "Der Grüffelo!"
    assert pl["filepath"] == "lib://der-gruffelo.tap"
    assert pl["files"] == [
        {"filepath": "http://bridge:8080/track/subsonic/s1.mp3", "name": "One"},
        {"filepath": "http://bridge:8080/track/subsonic/s2.mp3", "name": "Two"},
    ]


def test_build_endless_playlist_single_entry():
    pl = build_playlist(_mapping(kind="endless", source_type="artist"),
                        [Track(source="subsonic", ids=["s1"], title="x")],
                        "http://bridge:8080", "radio", ".tap")
    assert pl["files"] == [
        {"filepath": "http://bridge:8080/radio/m1.mp3", "name": "Der Grüffelo!"}
    ]


def test_write_playlist_atomic_and_readable(tmp_path):
    tracks = [Track(source="subsonic", ids=["s1"], title="One")]
    path = write_playlist(_mapping(), tracks, "http://b", str(tmp_path), ".tap")
    assert path == os.path.join(str(tmp_path), "der-gruffelo.tap")
    data = json.loads(open(path, encoding="utf-8").read())
    assert data["type"] == "tap"
    # no leftover temp files
    assert [p for p in os.listdir(tmp_path) if p.endswith(".tmp")] == []


def test_delete_playlist_removes_file(tmp_path):
    p = tmp_path / "x.tap"
    p.write_text("{}")
    delete_playlist(str(p))
    assert not p.exists()
    delete_playlist(str(p))  # idempotent, no error
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_playlist.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.playlist'`

- [ ] **Step 3: Implement `app/playlist.py`**

```python
from __future__ import annotations

import json
import os
import re
import unicodedata

from app.models import Mapping, Track

_TRANSLIT = {"ä": "a", "ö": "o", "ü": "u", "ß": "ss", "Ä": "a", "Ö": "o", "Ü": "u"}


def slugify(name: str) -> str:
    text = "".join(_TRANSLIT.get(ch, ch) for ch in name)
    text = unicodedata.normalize("NFKD", text).encode("ascii", "ignore").decode()
    text = text.lower()
    text = re.sub(r"[^a-z0-9]+", "-", text)
    return text.strip("-")


def build_playlist(mapping: Mapping, tracks: list[Track], bridge_base_url: str,
                   slug: str, extension: str) -> dict:
    base = bridge_base_url.rstrip("/")
    if mapping.kind == "endless":
        files = [{"filepath": f"{base}/radio/{mapping.id}.mp3", "name": mapping.name}]
    else:
        files = [{"filepath": f"{base}/track/{t.proxy_path}.mp3", "name": t.title}
                 for t in tracks]
    return {
        "type": "tap",
        "audio_id": 0,
        "filepath": f"lib://{slug}{extension}",
        "name": mapping.name,
        "files": files,
    }


def write_playlist(mapping: Mapping, tracks: list[Track], bridge_base_url: str,
                   library_path: str, extension: str) -> str:
    slug = slugify(mapping.name) or mapping.id
    playlist = build_playlist(mapping, tracks, bridge_base_url, slug, extension)
    final_path = os.path.join(library_path, f"{slug}{extension}")
    tmp_path = f"{final_path}.tmp"
    with open(tmp_path, "w", encoding="utf-8") as fh:
        json.dump(playlist, fh, ensure_ascii=False, indent=2)
    os.replace(tmp_path, final_path)
    return final_path


def delete_playlist(path: str) -> None:
    try:
        os.remove(path)
    except FileNotFoundError:
        pass
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_playlist.py -v`
Expected: PASS (6 passed)

- [ ] **Step 5: Commit**

```bash
git add app/playlist.py tests/test_playlist.py
git commit -m "feat: TeddyCloud .tap playlist generator with atomic writes"
```

---

## Task 10: Streaming proxy router

**Files:**
- Create: `app/streaming.py`
- Test: `tests/test_streaming.py`

A FastAPI router with `/track/{source}/{rest:path}` and `/radio/{mapping_id}.mp3`. It depends on a registry + storage provided via `app.state`. Tests build a minimal FastAPI app with a fake source.

- [ ] **Step 1: Write the failing test**

```python
import httpx
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from app.models import SourceStatus, StreamResponse, Track, BrowseEntry, Mapping
from app.sources.base import ContentSource
from app.streaming import router as streaming_router


class FakeSource(ContentSource):
    name = "fake"

    def __init__(self):
        self.opened = []

    async def test_connection(self): return SourceStatus(ok=True, name=self.name)
    async def browse(self, kind=None, parent_id=None, query=None): return []
    async def resolve_tracks(self, source_type, source_id):
        return [Track(source="fake", ids=["a"], title="A"),
                Track(source="fake", ids=["b"], title="B")]
    def cover_url(self, source_type, source_id): return None

    async def open_stream(self, ids, range_header=None):
        self.opened.append((tuple(ids), range_header))
        async def gen():
            yield b"AUDIO:" + "/".join(ids).encode()
        return StreamResponse(200, "audio/mpeg", {"Content-Length": "11"}, gen())


class FakeRegistry:
    def __init__(self, source): self._s = source
    def get(self, name): return self._s if name == "fake" else None
    def names(self): return ["fake"]


class FakeStorage:
    def __init__(self, mapping=None): self._m = mapping
    def get_mapping(self, mid): return self._m if (self._m and self._m.id == mid) else None


@pytest.fixture
def client_and_source():
    src = FakeSource()
    app = FastAPI()
    app.state.registry = FakeRegistry(src)
    app.state.storage = FakeStorage(
        Mapping(id="m1", name="R", source="fake", kind="endless",
                source_type="artist", source_id="ar1", shuffle=False,
                playlist_path="/p", created_at="t", updated_at="t"))
    app.include_router(streaming_router)
    return TestClient(app), src


def test_track_proxy_streams_bytes(client_and_source):
    client, src = client_and_source
    resp = client.get("/track/fake/xyz.mp3")
    assert resp.status_code == 200
    assert resp.content == b"AUDIO:xyz"
    assert src.opened[0][0] == ("xyz",)


def test_track_proxy_multi_id_path(client_and_source):
    client, src = client_and_source
    resp = client.get("/track/fake/itm1/100.mp3")
    assert resp.status_code == 200
    assert src.opened[0][0] == ("itm1", "100")


def test_track_proxy_forwards_range(client_and_source):
    client, src = client_and_source
    client.get("/track/fake/xyz.mp3", headers={"Range": "bytes=0-5"})
    assert src.opened[0][1] == "bytes=0-5"


def test_track_unknown_source_404(client_and_source):
    client, _ = client_and_source
    assert client.get("/track/nope/x.mp3").status_code == 404


def test_radio_streams_concatenated(client_and_source):
    client, src = client_and_source
    # endless radio loops forever; request a small byte range and stop early
    with client.stream("GET", "/radio/m1.mp3") as resp:
        assert resp.status_code == 200
        first = next(resp.iter_bytes())
        assert first.startswith(b"AUDIO:")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_streaming.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.streaming'`

- [ ] **Step 3: Implement `app/streaming.py`**

```python
from __future__ import annotations

import random

from fastapi import APIRouter, Header, HTTPException, Request
from fastapi.responses import StreamingResponse

router = APIRouter()


@router.get("/track/{source}/{rest:path}")
async def proxy_track(source: str, rest: str, request: Request,
                      range: str | None = Header(default=None)):
    registry = request.app.state.registry
    src = registry.get(source)
    if src is None:
        raise HTTPException(status_code=404, detail=f"unknown source: {source}")
    if rest.endswith(".mp3"):
        rest = rest[: -len(".mp3")]
    ids = rest.split("/")
    stream = await src.open_stream(ids, range_header=range)
    return StreamingResponse(stream.body, status_code=stream.status_code,
                             media_type=stream.media_type, headers=stream.headers)


@router.get("/radio/{mapping_id}.mp3")
async def proxy_radio(mapping_id: str, request: Request):
    storage = request.app.state.storage
    registry = request.app.state.registry
    mapping = storage.get_mapping(mapping_id)
    if mapping is None:
        raise HTTPException(status_code=404, detail="unknown mapping")
    src = registry.get(mapping.source)
    if src is None:
        raise HTTPException(status_code=404, detail="source not configured")

    tracks = await src.resolve_tracks(mapping.source_type, mapping.source_id)
    if mapping.shuffle:
        random.shuffle(tracks)

    async def generate():
        if not tracks:
            return
        while True:  # endless loop
            for track in tracks:
                try:
                    stream = await src.open_stream(track.ids)
                except Exception:  # noqa: BLE001 - skip a failing track, keep radio alive
                    continue
                async for chunk in stream.body:
                    yield chunk

    return StreamingResponse(generate(), media_type="audio/mpeg")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_streaming.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add app/streaming.py tests/test_streaming.py
git commit -m "feat: streaming proxy for per-track audio and endless radio"
```

---

## Task 11: JSON API router

**Files:**
- Create: `app/web/api.py`
- Test: `tests/test_api.py`

Endpoints: `GET /api/sources`, `GET /api/browse`, `POST /api/mappings`, `GET /api/mappings`, `PATCH /api/mappings/{id}`, `POST /api/mappings/{id}/regenerate`, `DELETE /api/mappings/{id}`, `GET/POST /api/settings`, `POST /api/settings/test`. Mapping creation resolves tracks, persists, and writes the playlist. Uses a deterministic id/clock injected via `app.state` for testability.

- [ ] **Step 1: Write the failing test**

```python
import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from app.storage import Storage
from app.config import ConfigService
from app.web.api import router as api_router
from app.models import SourceStatus, Track, BrowseEntry
from app.sources.base import ContentSource


class FakeSource(ContentSource):
    name = "fake"
    async def test_connection(self): return SourceStatus(ok=True, name="fake", detail="up")
    async def browse(self, kind=None, parent_id=None, query=None):
        return [BrowseEntry(id="al1", kind="album", title="Album", subtitle="Artist")]
    async def resolve_tracks(self, source_type, source_id):
        return [Track(source="fake", ids=["s1"], title="One"),
                Track(source="fake", ids=["s2"], title="Two")]
    async def open_stream(self, ids, range_header=None): ...
    def cover_url(self, source_type, source_id): return None


class FakeRegistry:
    def __init__(self): self._s = FakeSource()
    def get(self, name): return self._s if name == "fake" else None
    def names(self): return ["fake"]


@pytest.fixture
def client(tmp_path):
    storage = Storage.open(":memory:")
    app = FastAPI()
    app.state.storage = storage
    app.state.registry = FakeRegistry()
    app.state.config = ConfigService(storage)
    app.state.library_path = str(tmp_path)
    app.state.bridge_base_url = "http://bridge:8080"
    app.state.extension = ".tap"
    # deterministic id + timestamp
    counter = {"n": 0}
    def next_id():
        counter["n"] += 1
        return f"map{counter['n']}"
    app.state.new_id = next_id
    app.state.now = lambda: "2026-06-05T00:00:00"
    app.include_router(api_router)
    return TestClient(app), storage, str(tmp_path)


def test_sources_lists_configured(client):
    c, _, _ = client
    assert c.get("/api/sources").json() == ["fake"]


def test_browse_returns_entries(client):
    c, _, _ = client
    data = c.get("/api/browse", params={"source": "fake", "kind": "albums"}).json()
    assert data[0]["id"] == "al1" and data[0]["title"] == "Album"


def test_create_mapping_writes_playlist_and_record(client):
    c, storage, libdir = client
    import os, json
    resp = c.post("/api/mappings", json={
        "source": "fake", "source_type": "album", "source_id": "al1",
        "name": "My Album", "kind": "finite", "shuffle": False})
    assert resp.status_code == 201
    body = resp.json()
    assert body["id"] == "map1"
    assert body["lib_path"] == "lib://my-album.tap"
    # record persisted
    assert storage.get_mapping("map1").name == "My Album"
    # file written with two track entries
    written = json.loads(open(os.path.join(libdir, "my-album.tap"), encoding="utf-8").read())
    assert len(written["files"]) == 2


def test_list_and_delete_mapping(client):
    c, storage, libdir = client
    import os
    c.post("/api/mappings", json={"source": "fake", "source_type": "album",
           "source_id": "al1", "name": "X", "kind": "finite", "shuffle": False})
    assert len(c.get("/api/mappings").json()) == 1
    c.delete("/api/mappings/map1")
    assert c.get("/api/mappings").json() == []
    assert not os.path.exists(os.path.join(libdir, "x.tap"))


def test_settings_save_and_test(client):
    c, storage, _ = client
    c.post("/api/settings", json={"subsonic.url": "https://m"})
    assert storage.get_setting("subsonic.url") == "https://m"
    status = c.post("/api/settings/test", params={"source": "fake"}).json()
    assert status["ok"] is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_api.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.web.api'`

- [ ] **Step 3: Implement `app/web/api.py`**

```python
from __future__ import annotations

from dataclasses import asdict

from fastapi import APIRouter, Body, HTTPException, Request
from fastapi.responses import JSONResponse

from app.models import Mapping
from app.playlist import delete_playlist, slugify, write_playlist

router = APIRouter(prefix="/api")


def _state(request: Request):
    return request.app.state


@router.get("/sources")
async def list_sources(request: Request):
    return _state(request).registry.names()


@router.get("/browse")
async def browse(request: Request, source: str, kind: str | None = None,
                 parent: str | None = None, query: str | None = None):
    src = _state(request).registry.get(source)
    if src is None:
        raise HTTPException(404, f"unknown source: {source}")
    entries = await src.browse(kind=kind, parent_id=parent, query=query)
    return [asdict(e) for e in entries]


@router.get("/mappings")
async def list_mappings(request: Request):
    return [asdict(m) for m in _state(request).storage.list_mappings()]


async def _generate_playlist(state, mapping: Mapping) -> str:
    src = state.registry.get(mapping.source)
    if src is None:
        raise HTTPException(400, "source not configured")
    tracks = [] if mapping.kind == "endless" else await src.resolve_tracks(
        mapping.source_type, mapping.source_id)
    if mapping.kind == "finite" and not tracks:
        raise HTTPException(400, "source has no tracks")
    return write_playlist(mapping, tracks, state.bridge_base_url,
                          state.library_path, state.extension)


@router.post("/mappings", status_code=201)
async def create_mapping(request: Request, payload: dict = Body(...)):
    state = _state(request)
    now = state.now()
    mapping = Mapping(
        id=state.new_id(), name=payload["name"], source=payload["source"],
        kind=payload.get("kind", "finite"), source_type=payload["source_type"],
        source_id=payload["source_id"], shuffle=bool(payload.get("shuffle", False)),
        playlist_path="", created_at=now, updated_at=now,
    )
    mapping.playlist_path = await _generate_playlist(state, mapping)
    state.storage.save_mapping(mapping)
    return _mapping_response(mapping, state.extension)


@router.patch("/mappings/{mapping_id}")
async def update_mapping(request: Request, mapping_id: str, payload: dict = Body(...)):
    state = _state(request)
    mapping = state.storage.get_mapping(mapping_id)
    if mapping is None:
        raise HTTPException(404, "unknown mapping")
    old_path = mapping.playlist_path
    mapping.name = payload.get("name", mapping.name)
    mapping.kind = payload.get("kind", mapping.kind)
    mapping.shuffle = bool(payload.get("shuffle", mapping.shuffle))
    mapping.updated_at = state.now()
    new_path = await _generate_playlist(state, mapping)
    if old_path and old_path != new_path:
        delete_playlist(old_path)
    mapping.playlist_path = new_path
    state.storage.save_mapping(mapping)
    return _mapping_response(mapping, state.extension)


@router.post("/mappings/{mapping_id}/regenerate")
async def regenerate_mapping(request: Request, mapping_id: str):
    state = _state(request)
    mapping = state.storage.get_mapping(mapping_id)
    if mapping is None:
        raise HTTPException(404, "unknown mapping")
    mapping.playlist_path = await _generate_playlist(state, mapping)
    mapping.updated_at = state.now()
    state.storage.save_mapping(mapping)
    return _mapping_response(mapping, state.extension)


@router.delete("/mappings/{mapping_id}", status_code=204)
async def remove_mapping(request: Request, mapping_id: str):
    state = _state(request)
    mapping = state.storage.get_mapping(mapping_id)
    if mapping is None:
        raise HTTPException(404, "unknown mapping")
    delete_playlist(mapping.playlist_path)
    state.storage.delete_mapping(mapping_id)
    return JSONResponse(status_code=204, content=None)


@router.get("/settings")
async def get_settings(request: Request):
    keys = ["subsonic.url", "subsonic.user", "audiobookshelf.url",
            "library_path", "bridge_base_url", "mp3_bitrate", "playlist_extension",
            "default_shuffle"]
    storage = _state(request).storage
    return {k: storage.get_setting(k) for k in keys}


@router.post("/settings")
async def save_settings(request: Request, payload: dict = Body(...)):
    storage = _state(request).storage
    for key, value in payload.items():
        storage.set_setting(key, "" if value is None else str(value))
    return {"saved": True}


@router.post("/settings/test")
async def test_source(request: Request, source: str):
    src = _state(request).registry.get(source)
    if src is None:
        return {"ok": False, "name": source, "error": "not configured"}
    return asdict(await src.test_connection())


def _mapping_response(mapping: Mapping, extension: str) -> dict:
    slug = slugify(mapping.name) or mapping.id
    return {**asdict(mapping), "lib_path": f"lib://{slug}{extension}"}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_api.py -v`
Expected: PASS (6 passed)

- [ ] **Step 5: Commit**

```bash
git add app/web/api.py tests/test_api.py
git commit -m "feat: JSON API for browse, mappings CRUD, and settings"
```

---

## Task 12: App factory, startup checks, and HTML views

**Files:**
- Create: `app/main.py`
- Create: `app/web/views.py`
- Create: `app/web/templates/base.html`
- Create: `app/web/templates/library.html`
- Create: `app/web/templates/tonies.html`
- Create: `app/web/templates/settings.html`
- Create: `app/web/templates/partials/grid.html`
- Create: `app/web/templates/partials/tonie_row.html`
- Create: `app/web/static/app.css`
- Create: `app/web/static/app.js`
- Create: `app/web/static/htmx.min.js`
- Test: `tests/test_views.py`

The views render the cozy UI. **Adapt the approved mockups** at `.superpowers/brainstorm/197-1780688864/content/{library,tonies,settings}.html` into Jinja2 templates: lift the `<style>` blocks into `app/web/static/app.css` (shared), keep the markup, and replace the hard-coded demo cards/rows/panels with Jinja loops over real data. The mockups are the exact visual spec (cream `#F6EFE3`, Fraunces + Hanken Grotesk, coral `#E0603F`). The create-tonie drawer (`create-tonie.html`) becomes a client-side drawer in `app.js` driven by the JSON API.

- [ ] **Step 1: Write the failing test (server-side rendering smoke tests)**

```python
import pytest
from fastapi.testclient import TestClient
from app.main import create_app
from app.storage import Storage


@pytest.fixture
def client(tmp_path, monkeypatch):
    for k in ["SUBSONIC_URL", "ABS_URL"]:
        monkeypatch.delenv(k, raising=False)
    monkeypatch.setenv("TEDDYCLOUD_LIBRARY_PATH", str(tmp_path))
    app = create_app(db_path=":memory:")
    return TestClient(app)


def test_library_page_renders(client):
    resp = client.get("/")
    assert resp.status_code == 200
    assert "Library" in resp.text
    assert "Tonie Bridge" in resp.text  # brand from base.html


def test_tonies_page_renders(client):
    assert client.get("/tonies").status_code == 200


def test_settings_page_renders(client):
    resp = client.get("/settings")
    assert resp.status_code == 200
    assert "Audiobookshelf" in resp.text  # ABS panel present


def test_static_css_served(client):
    resp = client.get("/static/app.css")
    assert resp.status_code == 200
    assert "F6EFE3" in resp.text  # cozy paper background present


def test_healthcheck(client):
    assert client.get("/healthz").json() == {"status": "ok"}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_views.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'app.main'`

- [ ] **Step 3: Download HTMX into static**

Run: `curl -L -o app/web/static/htmx.min.js https://unpkg.com/htmx.org@1.9.12/dist/htmx.min.js`
Expected: a non-empty JS file (~40 KB). (Vendored so the LAN-only deployment needs no CDN.)

- [ ] **Step 4: Create `app/web/static/app.css`** — copy the `<style>` contents shared across the three mockups (the `:root` variables, sidebar, cards, panels, buttons). It MUST contain the `--paper:#F6EFE3` variable and the Fraunces/Hanken Grotesk `@import` or `<link>` is placed in `base.html`. Keep selectors identical to the mockups so the look matches exactly.

- [ ] **Step 5: Create `app/web/templates/base.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Tonie Bridge — {% block title %}{% endblock %}</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,600;9..144,700&family=Hanken+Grotesk:wght@400;500;600;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="/static/app.css">
  <script src="/static/htmx.min.js" defer></script>
  <script src="/static/app.js" defer></script>
</head>
<body>
<div class="app">
  <aside class="side">
    <div class="brand"><div class="mark">🧸</div><div><b>Tonie Bridge</b><span>streaming → teddycloud</span></div></div>
    <nav>
      <div class="nav-label">Menu</div>
      <a href="/" class="{{ 'active' if active=='library' }}"><span class="ic">🎵</span> Library</a>
      <a href="/tonies" class="{{ 'active' if active=='tonies' }}"><span class="ic">🏷️</span> Tonies</a>
      <a href="/settings" class="{{ 'active' if active=='settings' }}"><span class="ic">⚙️</span> Settings</a>
    </nav>
    <div class="spacer"></div>
    <div class="status">
      {% for s in sources %}<div class="row"><span class="dot ok"></span> {{ s }} <span class="muted">connected</span></div>{% endfor %}
      <div class="row"><span class="dot {{ 'ok' if library_ok else '' }}"></span> Library <span class="muted">{{ 'mounted' if library_ok else 'missing' }}</span></div>
    </div>
  </aside>
  <main class="main">{% block main %}{% endblock %}</main>
</div>
</body>
</html>
```

- [ ] **Step 6: Create `app/web/templates/library.html`, `tonies.html`, `settings.html`, and the two partials** — adapt the mockups. Concrete dynamic structures to use:

`library.html` (extends base, `active='library'`): render the source selector and tabs, then include the grid partial loaded via HTMX from `/api/browse`. Minimum dynamic block:

```html
{% block main %}
<div class="topbar"><div><h1>Library</h1><p>Browse your sources and turn anything into a Tonie.</p></div>
  <div class="search"><input id="q" placeholder="Search…" oninput="bridge.search(this.value)"></div></div>
{% if sources|length > 1 %}<div class="source-select">
  {% for s in sources %}<button class="seg {{ 'on' if loop.first }}" onclick="bridge.setSource('{{ s }}')">{{ s }}</button>{% endfor %}
</div>{% endif %}
<div id="tabs" class="tabs"></div>
<div id="grid" class="grid"></div>
<script>bridge.initLibrary({{ sources|tojson }});</script>
{% endblock %}
```

`partials/grid.html` (rendered by JS from JSON, OR server-side if you prefer) — the card markup per entry, reusing `.card`/`.cover`/`.mk` classes from the mockup, with a hover `＋ Tonie` button calling `bridge.openCreate(entry)`.

`tonies.html` (`active='tonies'`): server-render the list with a loop:

```html
{% block main %}
<div class="topbar"><div><h1>Tonies</h1><p>Playlists you've created.</p></div></div>
<div class="list">
  {% for m in mappings %}{% include "partials/tonie_row.html" %}{% endfor %}
</div>
{% endblock %}
```

`partials/tonie_row.html` — one row using `.item`/`.cov`/`.badge`/`.acts` classes; show `m.name`, a mode badge from `m.kind`, a source badge from `m.source`, the `lib://{{ m|slug }}` path with a copy button, and action buttons that `hx-post`/`hx-delete` to `/api/mappings/{{ m.id }}/...`.

`settings.html` (`active='settings'`): adapt the settings mockup’s five panels (Subsonic, Audiobookshelf, TeddyCloud library, Bridge address, Audio & defaults) into a `<form>` that POSTs JSON to `/api/settings` via `app.js`. Each source panel has a “Test connection” button calling `/api/settings/test?source=`.

- [ ] **Step 7: Create `app/web/static/app.js`** — a small `bridge` object: `initLibrary`, `setSource`, `search` (calls `/api/browse`, renders cards into `#grid`), `openCreate`/`submitCreate` (the drawer; POSTs `/api/mappings`, shows success state with copy button), `copy(text)`, and settings save/test helpers. No framework — plain `fetch` + DOM. Use the drawer markup/classes from `create-tonie.html`.

- [ ] **Step 8: Create `app/web/views.py`**

```python
from __future__ import annotations

import os

from fastapi import APIRouter, Request
from fastapi.templating import Jinja2Templates

templates = Jinja2Templates(directory="app/web/templates")
router = APIRouter()


def _common(request: Request) -> dict:
    state = request.app.state
    return {
        "request": request,
        "sources": state.registry.names(),
        "library_ok": os.path.isdir(state.library_path) and os.access(state.library_path, os.W_OK),
    }


@router.get("/")
async def library(request: Request):
    return templates.TemplateResponse("library.html", {**_common(request), "active": "library"})


@router.get("/tonies")
async def tonies(request: Request):
    ctx = _common(request)
    ctx["mappings"] = request.app.state.storage.list_mappings()
    ctx["active"] = "tonies"
    return templates.TemplateResponse("tonies.html", ctx)


@router.get("/settings")
async def settings(request: Request):
    ctx = _common(request)
    ctx["values"] = request.app.state.storage  # template reads via get_setting
    ctx["active"] = "settings"
    return templates.TemplateResponse("settings.html", ctx)
```

- [ ] **Step 9: Implement `app/main.py` (app factory + wiring + startup)**

```python
from __future__ import annotations

import uuid
from datetime import datetime, timezone

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from app.config import ConfigService
from app.sources.registry import SourceRegistry
from app.storage import Storage
from app.streaming import router as streaming_router
from app.web.api import router as api_router
from app.web.views import router as views_router


def create_app(db_path: str = "data/bridge.db") -> FastAPI:
    app = FastAPI(title="TeddyCloud Streaming Bridge")
    storage = Storage.open(db_path)
    config = ConfigService(storage)

    app.state.storage = storage
    app.state.config = config
    app.state.registry = SourceRegistry(config)
    app.state.library_path = config.library_path()
    app.state.bridge_base_url = config.bridge_base_url()
    app.state.extension = config.playlist_extension()
    app.state.new_id = lambda: uuid.uuid4().hex[:12]
    app.state.now = lambda: datetime.now(timezone.utc).isoformat()

    app.mount("/static", StaticFiles(directory="app/web/static"), name="static")
    app.include_router(api_router)
    app.include_router(streaming_router)
    app.include_router(views_router)

    @app.get("/healthz")
    async def healthz():
        return {"status": "ok"}

    return app


app = create_app()
```

- [ ] **Step 10: Run the view tests**

Run: `python -m pytest tests/test_views.py -v`
Expected: PASS (5 passed). If a template references data not in context, fix the template/context until green.

- [ ] **Step 11: Manual smoke run**

Run: `python -m uvicorn app.main:app --port 8080`
Open `http://localhost:8080` — confirm the cozy Library, Tonies, and Settings pages render with the correct fonts/colors. Stop with Ctrl+C.

- [ ] **Step 12: Commit**

```bash
git add app/main.py app/web tests/test_views.py
git commit -m "feat: app factory, HTML views, and cozy UI templates"
```

---

## Task 13: Full test run + Docker packaging

**Files:**
- Create: `Dockerfile`
- Create: `docker-compose.example.yml`
- Create: `README.md`

- [ ] **Step 1: Run the full suite**

Run: `python -m pytest -v`
Expected: ALL pass. Fix any cross-module breakage before continuing.

- [ ] **Step 2: Create `Dockerfile`**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml ./
RUN pip install --no-cache-dir .
COPY app ./app
ENV PORT=8080 BIND_ADDR=0.0.0.0
EXPOSE 8080
CMD ["sh", "-c", "uvicorn app.main:app --host ${BIND_ADDR} --port ${PORT}"]
```

- [ ] **Step 3: Create `docker-compose.example.yml`**

```yaml
services:
  bridge:
    build: .
    container_name: teddycloud-streaming-bridge
    ports:
      - "8080:8080"
    environment:
      BRIDGE_BASE_URL: "http://teddycloud-streaming-bridge:8080"
      TEDDYCLOUD_LIBRARY_PATH: "/teddycloud/library"
      # Optional bootstrap (otherwise configure in the UI):
      # SUBSONIC_URL: "http://navidrome:4533"
      # SUBSONIC_USER: "teddy"
      # SUBSONIC_PASSWORD: "secret"
      # ABS_URL: "http://audiobookshelf:13378"
      # ABS_TOKEN: "..."
    volumes:
      - /path/to/teddycloud/library:/teddycloud/library
      - bridge-data:/app/data
volumes:
  bridge-data:
```

- [ ] **Step 4: Build the image**

Run: `docker build -t teddycloud-streaming-bridge .`
Expected: image builds successfully.

- [ ] **Step 5: Create `README.md`** — include: what it does, the trusted-LAN security note (no auth; the Subsonic/ABS credentials are stored in the SQLite volume), the configuration table (all env vars from the spec's Configuration section), how to run via docker-compose, and the workflow (configure source → browse → create Tonie → assign the `.tap` in TeddyCloud's UI).

- [ ] **Step 6: Commit**

```bash
git add Dockerfile docker-compose.example.yml README.md
git commit -m "chore: Docker packaging and README"
```

---

## Self-Review (completed by plan author)

**Spec coverage check:**
- ContentSource abstraction + subsonic + audiobookshelf → Tasks 5, 6, 7 ✅
- Storage (settings + mappings incl. `source`) → Task 3 ✅
- Config overlay (env + UI), ABS vars → Task 4 ✅
- Playlist `.tap` generation, atomic, finite + endless → Task 9 ✅
- Streaming proxy (source-encoded URLs, Range, radio loop/shuffle/skip) → Task 10 ✅
- JSON API (browse, CRUD, regenerate, settings, test) → Task 11 ✅
- Web UI (cozy language, app shell, Library/Tonies/Settings, source selector, ABS panel, create drawer, copy path) → Task 12 ✅
- Error handling (source fail surfaced, skip failing radio track, library writable check, empty-source refusal, atomic writes) → Tasks 9, 10, 11, 12 ✅
- Testing strategy (unit per module, integration via TestClient) → every task ✅
- Deployment (Dockerfile, compose, README, no ffmpeg) → Task 13 ✅

**Open items carried into implementation (from spec):** playlist extension default `.tap` (configurable via `PLAYLIST_EXTENSION`); confirm TeddyCloud Range behaviour during the manual E2E; ABS single-file audiobooks become one entry (documented, no code needed).

**Type consistency:** `Track(source, ids, title, duration)` + `proxy_path`, `BrowseEntry`, `SourceStatus`, `StreamResponse`, `Mapping` are defined in Task 2 and used unchanged everywhere. `ContentSource` method signatures (`test_connection`, `browse`, `resolve_tracks`, `open_stream(ids, range_header)`, `cover_url`) are identical across base, both implementations, and all fakes.

**Manual-only steps flagged:** the template adaptation (Task 12 steps 4–7) and README (Task 13 step 5) are descriptive because they transcribe the already-approved mockups and spec; all logic-bearing code has complete listings and tests.
