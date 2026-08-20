# Phase 0 — Foundation & Skeleton

Part of [LazyBoy Master Design](../LazyBoy-Design.md).

---

## Goal

Produce the load-bearing skeleton every later phase bolts onto: a single `uvx lazyboy` command that starts one Uvicorn process on `127.0.0.1`, serves the built React SPA, exposes a resumable SSE event stream, and can already start/cancel a (fake) run end-to-end. No ADO, no Azure, no Claude — but every seam those phases need is cut and tested.

The two things that must be *right* here, because retrofitting them is painful:

1. **`RunEvent.seq` + SSE replay.** Resumability (P5) is one mechanism, and it's this one. Every later phase just emits events.
2. **The RunManager task/cancellation contract.** Phases 3–8 are all "a stage running inside a run". If the lifecycle is sloppy, every stage inherits the sloppiness.

## Definition of Done

| # | Criterion | How it's verified |
|---|---|---|
| D1 | `uv run lazyboy` picks a free port (7717 preferred), prints the URL, opens the browser at `http://127.0.0.1:<port>/?t=<token>` | manual + `test_launcher.py` |
| D2 | `GET /` serves the built SPA; `GET /runs/abc` (unknown path, `Accept: text/html`) also serves `index.html`; `GET /api/nope` returns JSON 404 | `test_spa_fallback.py` |
| D3 | `~/.lazyboy/` is created on first launch with a commented `config.yaml`; config loads via pydantic-settings with env override `LAZYBOY__SERVER__PORT=8000` | `test_config.py` |
| D4 | SQLite created at `~/.lazyboy/lazyboy.db`, `schema_version` = 1, re-launch is a no-op | `test_migrations.py` |
| D5 | `POST /api/runs` creates a demo run; `GET /api/runs/{id}/events?after=0` streams its events; disconnect + reconnect with `?after=<last seq>` yields no gaps and no duplicates | `test_sse_replay.py` |
| D6 | `POST /api/runs/{id}/cancel` stops the asyncio task within 2 s and the run ends in `cancelled` | `test_run_manager.py` |
| D7 | A run left in a non-terminal state by a hard kill is marked `failed` (reason `orphaned_by_restart`) on next startup | `test_crash_recovery.py` |
| D8 | A request without the session token (and without the cookie) gets 401; a request with `Origin: https://evil.com` gets 403 | `test_security.py` |
| D9 | `logger.info("token=%s", "ghp_xxx")` renders as `token=***REDACTED***` | `test_redaction.py` |
| D10 | `make dev` runs Uvicorn `--reload` on 7717 and Vite on 5173 with `/api` proxied; `make build` emits `src/lazyboy/static/index.html` | manual |

Explicitly **not** in Phase 0: any credential, any network call to ADO/Azure, any Claude SDK usage, the state machine's real transitions (Phase 0 has `created → running → done/cancelled/failed` only).

---

## Design

### 1. Repo scaffold

```
lazyboy/
├── pyproject.toml
├── Makefile
├── README.md
├── .python-version              # 3.12
├── src/lazyboy/
│   ├── __init__.py              # __version__
│   ├── main.py                  # CLI launcher (typer)
│   ├── app.py                   # create_app() factory
│   ├── config.py                # Settings (pydantic-settings)
│   ├── logging.py               # setup_logging() + RedactingFilter
│   ├── security.py              # session token + Origin middleware
│   ├── paths.py                 # ~/.lazyboy resolution, XDG-aware
│   ├── db/
│   │   ├── __init__.py          # engine, session factory
│   │   ├── models.py            # SQLModel: Run, RunEvent, Gate (stubs for later)
│   │   └── migrations.py        # schema_version ladder
│   ├── core/
│   │   ├── events.py            # RunEvent envelope + EventBus
│   │   ├── run_manager.py       # asyncio task registry
│   │   └── errors.py            # LazyBoyError hierarchy → RFC7807 responses
│   ├── api/
│   │   ├── __init__.py          # api_router aggregation
│   │   ├── health.py            # /api/health, /api/meta
│   │   ├── runs.py              # /api/runs*
│   │   └── events.py            # /api/runs/{id}/events (SSE)
│   └── static/                  # built SPA — gitignored, produced by `make build`
├── web/
│   ├── index.html
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   ├── package.json
│   └── src/
│       ├── main.tsx  App.tsx  api/client.ts  api/queries.ts
│       ├── hooks/useRunStream.ts
│       ├── components/{AppShell,RunEventList,StatusPill}.tsx
│       └── routes/{Inbox,RunView,Settings}.tsx
└── tests/
```

`src/lazyboy/static/` is a *generated* directory checked into the wheel but not into git. `hatchling` is configured with `artifacts` so `uv build` includes it; `make build` must run before `uv build` (enforced by a build hook in Phase 9).

### 2. pyproject.toml

```toml
[project]
name = "lazyboy"
version = "0.1.0"
description = "Agentic bug-fixing cockpit for Azure DevOps"
requires-python = ">=3.12,<3.14"
authors = [{ name = "Francesco Colombo", email = "francesco.colombo@zealitconsultants.ai" }]
readme = "README.md"
license = { text = "Proprietary" }

dependencies = [
  # web
  "fastapi>=0.115,<0.117",
  "uvicorn[standard]>=0.34,<0.36",
  "sse-starlette>=2.1,<3",
  "python-multipart>=0.0.20",
  # config / cli
  "pydantic>=2.9,<3",
  "pydantic-settings>=2.6,<3",
  "pyyaml>=6.0.2",
  "typer>=0.15,<0.20",
  "rich>=13.9",
  # persistence
  "sqlmodel>=0.0.22,<0.1",
  # agent engine (wired in Phase 5; pinned now so the env is stable)
  "claude-agent-sdk>=0.1.0",
  # connectors (Phases 1-4)
  "httpx[http2]>=0.28,<0.29",
  "tenacity>=9.0,<10",
  "azure-identity>=1.19,<2",
  "azure-monitor-query>=1.4,<2",
  "keyring>=25.5,<26",
  "python-dateutil>=2.9",
]

[project.optional-dependencies]
dev = [
  "pytest>=8.3", "pytest-asyncio>=0.24", "pytest-cov>=6.0",
  "httpx-sse>=0.4",            # SSE client for tests
  "respx>=0.22",               # httpx mocking (Phase 1+)
  "ruff>=0.8", "mypy>=1.13",
  "types-PyYAML", "types-python-dateutil",
]

[project.scripts]
lazyboy = "lazyboy.main:cli"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/lazyboy"]
artifacts = ["src/lazyboy/static/**"]

[tool.uv]
package = true

[tool.ruff]
line-length = 100
target-version = "py312"
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC", "S", "RUF"]
ignore = ["S101"]                     # asserts are fine in tests

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
filterwarnings = ["error::DeprecationWarning:lazyboy.*"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
```

Install/run: `uv sync --all-extras`, `uv run lazyboy`. End-user path: `uvx lazyboy` (uv builds an ephemeral venv from the published wheel, which already contains `static/`).

### 3. App factory & router layout

`create_app()` takes an explicit `Settings` so tests can build isolated apps against a tmp home. No module-level globals except the logger; everything else hangs off `app.state`.

```python
# src/lazyboy/app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings: Settings = app.state.settings
    app.state.engine = init_engine(settings.db_path)
    run_migrations(app.state.engine)
    app.state.bus = EventBus(app.state.engine)
    app.state.runs = RunManager(app.state.engine, app.state.bus)
    await app.state.runs.recover_orphans()          # D7
    try:
        yield
    finally:
        await app.state.runs.shutdown(grace=5.0)
        await app.state.bus.aclose()
        app.state.engine.dispose()

def create_app(settings: Settings) -> FastAPI:
    app = FastAPI(title="LazyBoy", version=__version__, lifespan=lifespan,
                  docs_url="/api/docs", openapi_url="/api/openapi.json")
    app.state.settings = settings
    app.add_middleware(OriginGuardMiddleware, allowed=settings.server.allowed_origins)
    app.add_middleware(SessionTokenMiddleware, token=settings.runtime.session_token)
    app.include_router(api_router, prefix="/api")
    app.add_exception_handler(LazyBoyError, problem_detail_handler)
    mount_spa(app, settings)                         # must be last: it owns "/{path:path}"
    return app
```

| Prefix | Router | Phase |
|---|---|---|
| `/api/health`, `/api/meta` | `api.health` | 0 |
| `/api/runs` | `api.runs` | 0 (extended 3–8) |
| `/api/runs/{id}/events` | `api.events` | 0 |
| `/api/auth` | `api.auth` | 1 |
| `/api/workitems` | `api.workitems` | 2 |
| `/api/repos` | `api.repos` | 4 |
| `/api/gates` | `api.gates` | 5 |

Errors are RFC 7807 problem documents so the frontend has one shape to parse:

```json
{"type":"lazyboy/run-not-found","title":"Run not found","status":404,"detail":"run 01J... does not exist","instance":"/api/runs/01J..."}
```

### 4. CLI launcher

Responsibilities, in order: resolve home → load/seed config → set up logging → mint a per-launch session token → bind a socket (preferred port, else ephemeral) → hand the *already-bound* socket to Uvicorn → open the browser → serve until SIGINT → graceful shutdown.

Binding the socket ourselves before Uvicorn starts removes the classic race where we print/open a URL for a port that Uvicorn then fails to claim.

```python
# src/lazyboy/main.py
import asyncio, contextlib, secrets, signal, socket, sys, threading, webbrowser
import typer, uvicorn
from rich.console import Console

cli = typer.Typer(add_completion=False, help="LazyBoy — agentic bug-fixing cockpit")
console = Console()

def _bind(host: str, preferred: int) -> socket.socket:
    """Bind preferred port, else an ephemeral one. Returns a listening socket."""
    for port in (preferred, 0):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        try:
            s.bind((host, port))
            s.listen(128)
            return s
        except OSError:
            s.close()
    raise RuntimeError("could not bind a port on 127.0.0.1")

@cli.command()
def serve(
    port: int = typer.Option(7717, help="Preferred port; falls back to an ephemeral one."),
    open_browser: bool = typer.Option(True, "--open/--no-open"),
    log_level: str = typer.Option("info"),
    config_path: str | None = typer.Option(None, "--config"),
) -> None:
    settings = load_settings(config_path)
    settings.runtime.session_token = secrets.token_urlsafe(32)
    setup_logging(settings.paths.log_dir, log_level)

    sock = _bind(settings.server.host, port)
    actual = sock.getsockname()[1]
    settings.server.port = actual
    settings.server.allowed_origins = [f"http://127.0.0.1:{actual}", f"http://localhost:{actual}"]
    url = f"http://127.0.0.1:{actual}/?t={settings.runtime.session_token}"

    console.print(f"[bold green]LazyBoy[/] {__version__}  →  [link={url}]{url}[/]")
    console.print(f"[dim]home: {settings.paths.home}   db: {settings.db_path}[/]")

    config = uvicorn.Config(create_app(settings), log_config=None, log_level=log_level,
                            lifespan="on", timeout_graceful_shutdown=10, access_log=False)
    server = uvicorn.Server(config)

    if open_browser:
        # delay so the browser hits a server that is already serving /
        threading.Timer(0.4, lambda: webbrowser.open_new_tab(url)).start()

    try:
        server.run(sockets=[sock])          # installs its own SIGINT/SIGTERM handlers
    finally:
        with contextlib.suppress(OSError):
            sock.close()
        console.print("[dim]LazyBoy stopped.[/]")

def main() -> None:          # `lazyboy` with no args == `lazyboy serve`
    if len(sys.argv) == 1:
        sys.argv.append("serve")
    cli()
```

Graceful shutdown chain: SIGINT → Uvicorn stops accepting → open SSE generators receive `asyncio.CancelledError` and close → lifespan `finally` → `RunManager.shutdown(grace=5)` cancels every live run task, awaits with `asyncio.wait(..., timeout=grace)`, and marks survivors `cancelled` in SQLite. Anything still alive after the grace period is abandoned; the next startup's `recover_orphans()` reconciles it.

**Second-instance behaviour:** if 7717 is already bound, we don't fight it — we probe `GET http://127.0.0.1:7717/api/meta`. If it answers with `{"app":"lazyboy"}` we print "LazyBoy is already running" and just open the browser (without a token — the existing instance's cookie is already set). Otherwise we take an ephemeral port.

### 5. Serving the SPA + fallback routing

```python
# src/lazyboy/api/spa.py
from pathlib import Path
from fastapi import FastAPI, Request
from fastapi.responses import FileResponse, JSONResponse
from fastapi.staticfiles import StaticFiles

STATIC = Path(__file__).resolve().parent.parent / "static"

def mount_spa(app: FastAPI, settings: Settings) -> None:
    if not (STATIC / "index.html").exists():
        @app.get("/{path:path}", include_in_schema=False)
        async def _missing(path: str):                       # dev convenience
            return JSONResponse({"detail": "SPA not built — run `make build` or use Vite on :5173"}, 501)
        return

    # hashed assets: immutable, year-long cache
    app.mount("/assets", StaticFiles(directory=STATIC / "assets"), name="assets")

    @app.get("/{path:path}", include_in_schema=False)
    async def spa(request: Request, path: str):
        if path.startswith("api/"):
            return JSONResponse({"detail": "Not Found"}, status_code=404)
        candidate = (STATIC / path).resolve()
        if path and candidate.is_file() and candidate.is_relative_to(STATIC):
            return FileResponse(candidate)
        return FileResponse(STATIC / "index.html",
                            headers={"Cache-Control": "no-store"})
    return
```

Rules: `/api/*` never falls through to the SPA; `index.html` is `no-store` (so a new build is picked up immediately); `/assets/*` is content-hashed by Vite and served `immutable`; the `is_relative_to` check kills `../` traversal.

### 6. Config system

`~/.lazyboy/config.yaml` is the user-editable surface; `pydantic-settings` layers env vars over it (delimiter `__`), and CLI flags over that. Precedence: **CLI > env > yaml > defaults**. Home is `$LAZYBOY_HOME` if set, else `~/.lazyboy`.

```python
# src/lazyboy/config.py
class ServerSettings(BaseModel):
    host: str = "127.0.0.1"          # never configurable to anything else in v1
    port: int = 7717
    allowed_origins: list[str] = []

class AgentSettings(BaseModel):
    investigator_model: str = "claude-opus-4-6"
    fixer_model: str = "claude-sonnet-4-6"
    # Budgets are turns and tokens, not dollars: LazyBoy runs on a Claude subscription
    # (`claude login`), so there is no per-run cost to cap. See master doc §5.4.
    max_turns_investigate: int = 60
    max_turns_fix: int = 80
    max_turns_per_run: int = 180            # ceiling across all stages of one run
    task_budget_investigate: int = 8        # subagent fan-out ceiling
    task_budget_fix: int = 12
    effort_investigate: str = "high"
    effort_fix: str = "medium"
    rate_limit_max_wait_s: int = 1800       # a 429 pauses and resumes; longer than this fails
    usage_mode: str = "auto"                # auto | turns | dollars — detected from the credential

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="LAZYBOY__", env_nested_delimiter="__",
                                      extra="ignore")
    paths: PathSettings = PathSettings()
    server: ServerSettings = ServerSettings()
    ado: AdoSettings = AdoSettings()             # Phase 1
    app_insights: AppInsightsSettings = AppInsightsSettings()   # Phase 3
    agent: AgentSettings = AgentSettings()
    runtime: RuntimeSettings = RuntimeSettings() # not persisted: session_token, actual port

    @property
    def db_path(self) -> Path: return self.paths.home / "lazyboy.db"
```

Seeded `config.yaml` (written verbatim on first launch, comments included):

```yaml
# ~/.lazyboy/config.yaml — LazyBoy configuration.
# Every key can be overridden by env: LAZYBOY__SERVER__PORT=8000
# Secrets NEVER live here. They live in the OS keychain (see Phase 1).

server:
  port: 7717                  # preferred; falls back to a free port if taken
  # host is always 127.0.0.1 and is not configurable.

ado:
  organization: ""            # e.g. "contoso"  -> https://dev.azure.com/contoso
  project: ""                 # THE project — one org, one project (master §8). Required.
  scope: project              # project | org; v1 is always `project`
  tenant_id: ""               # Entra tenant GUID; blank = credential default
  auth_mode: auto             # auto | entra | pat

app_insights:
  default: ""                 # key from `resources` below
  resources: {}
  #  portal_eus_appinsight:
  #    subscription_id: 5cff65c6-d6f7-4a96-b7ed-64cbf1ed5e89
  #    resource_group: rg_b2c_eus_portalprod_app
  #    name: portal_eus_appinsight
  #    environment: prod

agent:
  investigator_model: claude-opus-4-6
  fixer_model: claude-sonnet-4-6
  # Budgets are expressed in TURNS, not dollars — LazyBoy uses a Claude subscription
  # (`claude login`), so there is no per-run dollar figure to cap.
  max_turns_investigate: 60
  max_turns_fix: 80
  max_turns_per_run: 180      # across all stages of one run
  task_budget_investigate: 8  # max subagent tasks; fan-out is where turns multiply
  task_budget_fix: 12
  effort_investigate: high    # routing knob, not a limit
  effort_fix: medium
  rate_limit_max_wait_s: 1800 # on a 429 the run pauses and resumes; beyond this it fails
  usage_mode: auto            # auto | turns | dollars; auto detects `claude` session vs API key
  web_fetch_allow_domains:    # investigator only; fixer has no network
    - learn.microsoft.com
    - docs.microsoft.com

git:
  author_name: ""             # blank -> read from global git config
  author_email: ""
  default_target_branch: develop

ui:
  theme: system               # system | light | dark
  events_page_size: 200

logging:
  level: info                 # debug | info | warning | error
  retain_days: 14
```

### 7. SQLite + migrations

One writer, so: `PRAGMA journal_mode=WAL`, `synchronous=NORMAL`, `foreign_keys=ON`, `busy_timeout=5000`. SQLModel gives the ORM; migrations are a hand-rolled ladder (Alembic is overkill for a single-file, single-user DB and adds a runtime dep + a migration directory to ship).

```python
# src/lazyboy/db/migrations.py
MIGRATIONS: list[tuple[int, str]] = [
    (1, """
        CREATE TABLE IF NOT EXISTS run (
            id TEXT PRIMARY KEY,
            work_item_id INTEGER,
            state TEXT NOT NULL,
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL,
            error TEXT
        );
        CREATE TABLE IF NOT EXISTS run_event (
            seq INTEGER PRIMARY KEY AUTOINCREMENT,
            run_id TEXT NOT NULL REFERENCES run(id) ON DELETE CASCADE,
            ts TEXT NOT NULL,
            kind TEXT NOT NULL,
            payload TEXT NOT NULL
        );
        CREATE INDEX IF NOT EXISTS ix_event_run_seq ON run_event(run_id, seq);
    """),
]

def run_migrations(engine) -> int:
    with engine.begin() as cx:
        cx.exec_driver_sql("CREATE TABLE IF NOT EXISTS schema_version (version INTEGER NOT NULL)")
        row = cx.exec_driver_sql("SELECT version FROM schema_version").fetchone()
        current = row[0] if row else 0
        if not row:
            cx.exec_driver_sql("INSERT INTO schema_version(version) VALUES (0)")
        for version, sql in MIGRATIONS:
            if version > current:
                for stmt in filter(str.strip, sql.split(";")):
                    cx.exec_driver_sql(stmt)
                cx.exec_driver_sql("UPDATE schema_version SET version = ?", (version,))
                current = version
    return current
```

Migrations are additive-only and idempotent. Downgrade is not supported; a version newer than the code knows about aborts startup with a clear message ("this database was written by LazyBoy ≥ x; upgrade or delete ~/.lazyboy/lazyboy.db").

`run_event.seq` is a **global** `AUTOINCREMENT`, not per-run. That means seq is globally monotonic and the `(run_id, seq)` index makes the replay query a range scan. The client only ever compares seqs within one run, so gaps are irrelevant.

### 8. EventBus + SSE

Contract:

- **Write path:** `bus.emit(run_id, kind, payload)` inserts the row (getting `seq` from SQLite), then fans out to in-memory subscriber queues. DB first, always — a subscriber that misses a fan-out recovers by replaying; a subscriber that got an event that isn't in the DB can never recover.
- **Read path:** the SSE endpoint subscribes *first*, then replays from `after`, then drains the live queue, discarding anything with `seq <= last_replayed`. Subscribe-then-replay is what closes the race window.
- **Backpressure:** each subscriber queue is bounded (512). On overflow the subscriber is dropped with a `stream_overflow` sentinel; the browser reconnects with `?after=<last seq>` and replays from the DB. Slow clients degrade to polling-by-replay instead of ballooning memory.
- **Heartbeat:** an SSE comment (`: hb`) every 15 s so proxies and the browser's dead-connection detector behave.

```python
# src/lazyboy/core/events.py
from __future__ import annotations
import asyncio, json
from dataclasses import dataclass
from datetime import UTC, datetime

QUEUE_MAX = 512

@dataclass(frozen=True, slots=True)
class RunEvent:
    seq: int
    run_id: str
    ts: str
    kind: str                       # "run.state" | "log" | "agent.text" | "tool.pre" | ...
    payload: dict

    def to_sse(self) -> dict:
        return {"id": str(self.seq), "event": self.kind,
                "data": json.dumps({"seq": self.seq, "run_id": self.run_id,
                                    "ts": self.ts, "kind": self.kind, **self.payload},
                                   separators=(",", ":"))}

class EventBus:
    """Durable append-only event log with in-process fan-out."""

    def __init__(self, engine) -> None:
        self._engine = engine
        self._subs: dict[str, set[asyncio.Queue[RunEvent | None]]] = {}
        self._write_lock = asyncio.Lock()      # single SQLite writer

    async def emit(self, run_id: str, kind: str, payload: dict | None = None) -> RunEvent:
        payload = payload or {}
        ts = datetime.now(UTC).isoformat()
        async with self._write_lock:
            seq = await asyncio.to_thread(self._insert, run_id, ts, kind, payload)
        ev = RunEvent(seq=seq, run_id=run_id, ts=ts, kind=kind, payload=payload)
        for q in list(self._subs.get(run_id, ())):
            try:
                q.put_nowait(ev)
            except asyncio.QueueFull:
                self._subs[run_id].discard(q)
                with contextlib.suppress(asyncio.QueueFull):
                    q.put_nowait(None)          # sentinel: overflow -> client must reconnect
        return ev

    def _insert(self, run_id: str, ts: str, kind: str, payload: dict) -> int:
        with self._engine.begin() as cx:
            cur = cx.exec_driver_sql(
                "INSERT INTO run_event(run_id, ts, kind, payload) VALUES (?,?,?,?)",
                (run_id, ts, kind, json.dumps(payload, separators=(",", ":"))))
            return int(cur.lastrowid)

    @contextlib.contextmanager
    def subscribe(self, run_id: str):
        q: asyncio.Queue[RunEvent | None] = asyncio.Queue(maxsize=QUEUE_MAX)
        self._subs.setdefault(run_id, set()).add(q)
        try:
            yield q
        finally:
            self._subs.get(run_id, set()).discard(q)
            if not self._subs.get(run_id):
                self._subs.pop(run_id, None)

    def replay(self, run_id: str, after: int, limit: int = 2000) -> list[RunEvent]:
        with self._engine.begin() as cx:
            rows = cx.exec_driver_sql(
                "SELECT seq, ts, kind, payload FROM run_event "
                "WHERE run_id = ? AND seq > ? ORDER BY seq LIMIT ?",
                (run_id, after, limit)).fetchall()
        return [RunEvent(seq=r[0], run_id=run_id, ts=r[1], kind=r[2], payload=json.loads(r[3]))
                for r in rows]

    async def aclose(self) -> None:
        for subs in self._subs.values():
            for q in subs:
                with contextlib.suppress(asyncio.QueueFull):
                    q.put_nowait(None)
        self._subs.clear()
```

The SSE endpoint:

```python
# src/lazyboy/api/events.py
import asyncio, contextlib
from fastapi import APIRouter, Request, HTTPException
from sse_starlette.sse import EventSourceResponse

router = APIRouter()
HEARTBEAT_S = 15.0

@router.get("/runs/{run_id}/events")
async def stream_run_events(run_id: str, request: Request, after: int = 0):
    bus: EventBus = request.app.state.bus
    if not run_exists(request.app.state.engine, run_id):
        raise HTTPException(404, "run not found")

    # Last-Event-ID wins over ?after= — it's what the browser sends on auto-reconnect.
    if (leid := request.headers.get("last-event-id")) and leid.isdigit():
        after = int(leid)

    async def gen():
        with bus.subscribe(run_id) as q:                 # subscribe BEFORE replay
            cursor = after
            while backlog := bus.replay(run_id, cursor, limit=2000):
                for ev in backlog:
                    yield ev.to_sse()
                cursor = backlog[-1].seq
            yield {"event": "ready", "data": '{"after":%d}' % cursor}

            while True:
                if await request.is_disconnected():
                    return
                try:
                    ev = await asyncio.wait_for(q.get(), timeout=HEARTBEAT_S)
                except TimeoutError:
                    yield {"comment": "hb"}
                    continue
                if ev is None:                            # overflow / shutdown
                    yield {"event": "stream_overflow",
                           "data": '{"after":%d}' % cursor}
                    return
                if ev.seq <= cursor:                      # de-dup replay overlap
                    continue
                cursor = ev.seq
                yield ev.to_sse()

    return EventSourceResponse(
        gen(),
        ping=None,                                        # we own the heartbeat
        headers={"Cache-Control": "no-store", "X-Accel-Buffering": "no"},
    )
```

Reconnect semantics, stated once so every later phase can rely on them:

| Situation | Client behaviour | Server behaviour |
|---|---|---|
| Network blip | `EventSource` auto-reconnects, sends `Last-Event-ID` | replays `seq > id`, then live |
| Browser refresh | hook sends `?after=<last seq in cache>` (or 0) | full replay from that point |
| Run finished before connect | replay yields everything, `ready`, then a terminal `run.state` event already in the backlog | stream stays open (cheap) until the client closes |
| Client too slow | receives `stream_overflow`, closes, reconnects with `after` | drops the queue, no memory growth |
| Server shutdown | connection drops; hook backs off 1 s → 30 s | generators cancelled by Uvicorn |

Event kinds defined in Phase 0 (the taxonomy grows in `reference/data-model.md`): `run.created`, `run.state`, `run.error`, `log`, `heartbeat` (never sent as data — it's a comment), `stream_overflow`, `ready`.

### 9. RunManager skeleton

```python
# src/lazyboy/core/run_manager.py
class RunManager:
    def __init__(self, engine, bus: EventBus) -> None:
        self._engine, self._bus = engine, bus
        self._tasks: dict[str, asyncio.Task] = {}

    async def start(self, work_item_id: int | None = None) -> str:
        run_id = new_run_id()                       # ULID: sortable, URL-safe
        self._persist_new(run_id, work_item_id)
        await self._bus.emit(run_id, "run.created", {"work_item_id": work_item_id})
        task = asyncio.create_task(self._supervise(run_id), name=f"run:{run_id}")
        self._tasks[run_id] = task
        task.add_done_callback(lambda _t: self._tasks.pop(run_id, None))
        return run_id

    async def _supervise(self, run_id: str) -> None:
        try:
            await self._set_state(run_id, "running")
            await self._pipeline(run_id)            # Phase 0: a 5-step demo; Phase 3+: real stages
            await self._set_state(run_id, "done")
        except asyncio.CancelledError:
            await self._set_state(run_id, "cancelled")
            raise
        except Exception as exc:                    # noqa: BLE001 — top of the task
            log.exception("run %s failed", run_id)
            await self._bus.emit(run_id, "run.error",
                                 {"type": type(exc).__name__, "message": str(exc)})
            await self._set_state(run_id, "failed", error=str(exc))

    async def cancel(self, run_id: str) -> bool:
        if (t := self._tasks.get(run_id)) is None:
            return False
        t.cancel()
        with contextlib.suppress(asyncio.CancelledError, TimeoutError):
            await asyncio.wait_for(asyncio.shield(t), timeout=5.0)
        return True

    async def recover_orphans(self) -> int:
        """Nothing is running at startup, so any non-terminal run is a corpse."""
        live = {"running", "harvesting", "resolving", "investigating", "fixing"}
        with self._engine.begin() as cx:
            ids = [r[0] for r in cx.exec_driver_sql(
                "SELECT id FROM run WHERE state IN (%s)" %
                ",".join("?" * len(live)), tuple(live)).fetchall()]
        for rid in ids:
            await self._bus.emit(rid, "run.error", {"type": "OrphanedByRestart",
                                                    "message": "process exited while this run was active"})
            await self._set_state(rid, "failed", error="orphaned_by_restart")
        return len(ids)

    async def shutdown(self, grace: float = 5.0) -> None:
        tasks = list(self._tasks.values())
        for t in tasks:
            t.cancel()
        if tasks:
            await asyncio.wait(tasks, timeout=grace)
```

Cancellation contract every future stage must honour: **stages are cancellation-safe, not cancellation-immune.** Long blocking work goes through `asyncio.to_thread` or a subprocess with an explicit kill path; `finally` blocks do cleanup only (no awaits that can hang); no bare `except Exception` swallowing `CancelledError` (it's `BaseException` in 3.12, but the linter rule stays).

### 10. Logging + redaction

`setup_logging()` installs a `RichHandler` for the console and a `RotatingFileHandler` (`~/.lazyboy/logs/lazyboy.log`, 10 MB × 5) emitting JSON lines. Uvicorn's loggers are re-parented (`log_config=None` in the launcher) so everything shares the filter.

```python
# src/lazyboy/logging.py
SECRET_PATTERNS = [
    re.compile(r"\beyJ[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}"),  # JWT
    re.compile(r"\b[a-z2-7]{52}\b"),                       # ADO PAT (base32, 52 chars)
    re.compile(r"\bgh[pousr]_[A-Za-z0-9]{20,}"),           # GitHub-style
    re.compile(r"\bsk-ant-[A-Za-z0-9\-_]{20,}"),           # Anthropic key
    re.compile(r"(?i)(authorization|x-api-key)\s*[:=]\s*\S+"),
    re.compile(r"(?i)\b(pat|token|secret|password|bearer)\b\s*[:=]\s*\S+"),
]

class RedactingFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.msg = self._scrub(record.getMessage())
        record.args = ()                     # message is already interpolated
        return True

    @staticmethod
    def _scrub(text: str) -> str:
        for pat in SECRET_PATTERNS:
            text = pat.sub("***REDACTED***", text)
        return text
```

Interpolating in the filter (then clearing `args`) is deliberate: it guarantees a secret passed as a lazy `%s` argument is scrubbed too. The cost is losing structured args, which we don't use for anything but the message.

### 11. Security (localhost-only)

Two middlewares, both cheap and both mandatory:

**`OriginGuardMiddleware`** — for any non-`GET` request, and for `/api/*` GETs, require `Origin`/`Referer` to be absent (curl, EventSource same-origin in some browsers) or exactly in the allowlist (`http://127.0.0.1:<port>`, `http://localhost:<port>`, plus `http://localhost:5173` when `LAZYBOY_DEV=1`). Anything else → 403 `origin_not_allowed`. This is the DNS-rebinding / drive-by-CSRF defence. We also reject `Host` headers that aren't `127.0.0.1:<port>`/`localhost:<port>`.

**`SessionTokenMiddleware`** — one 32-byte URL-safe token per launch, never persisted. Accepted from, in order: the `lazyboy_session` cookie, the `X-LazyBoy-Token` header, the `?t=` query param. On a `?t=` match for a document request, the response sets `Set-Cookie: lazyboy_session=<t>; Path=/; SameSite=Strict; HttpOnly` and 302s to the same path without the query, so the token doesn't linger in history or referrers. `/api/health` and `/api/meta` are exempt (needed for the "already running?" probe). Comparison uses `secrets.compare_digest`.

Threat model for Phase 0: a malicious page in the user's browser cannot read the token (SameSite=Strict + no CORS headers → no cross-origin response reads), and cannot forge requests (Origin check). Another local process running as the same user *can* read `~/.lazyboy` and attach a debugger — out of scope, same as any local tool. Full model in Phase 1.

### 12. Frontend scaffold

`web/vite.config.ts`:

```ts
export default defineConfig({
  plugins: [react(), tailwindcss()],
  build: { outDir: "../src/lazyboy/static", emptyOutDir: true, sourcemap: true },
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://127.0.0.1:7717",
        changeOrigin: false,          // keep Origin: localhost:5173 for the dev allowlist
        ws: false,
      },
    },
  },
});
```

TanStack Query is configured once with a `queryFn` that goes through a fetch wrapper adding `X-LazyBoy-Token` (read from `?t=` on first load and stashed in `sessionStorage`), throwing typed `ProblemError`s, and mapping 401 → a global "session expired, reopen from the terminal" banner. `staleTime: 30_000`, `retry: (n, e) => e.status >= 500 && n < 3`, `refetchOnWindowFocus: true`.

`useRunStream` — the one hook every later phase's live UI uses:

```ts
// web/src/hooks/useRunStream.ts
import { useEffect, useReducer, useRef, useState } from "react";
import { sessionToken } from "../api/client";

export type RunEvent = { seq: number; run_id: string; ts: string; kind: string } & Record<string, unknown>;
export type StreamStatus = "connecting" | "live" | "reconnecting" | "closed";

export function useRunStream(runId: string | undefined, opts: { max?: number } = {}) {
  const max = opts.max ?? 5000;
  const [events, push] = useReducer(
    (acc: RunEvent[], e: RunEvent | "reset") =>
      e === "reset" ? [] : (acc.length >= max ? [...acc.slice(1), e] : [...acc, e]),
    [],
  );
  const [status, setStatus] = useState<StreamStatus>("connecting");
  const lastSeq = useRef(0);
  const backoff = useRef(1000);

  useEffect(() => {
    if (!runId) return;
    let es: EventSource | null = null;
    let timer: number | undefined;
    let cancelled = false;

    const connect = () => {
      if (cancelled) return;
      const url = `/api/runs/${runId}/events?after=${lastSeq.current}&t=${sessionToken()}`;
      es = new EventSource(url);            // EventSource can't set headers -> token in query

      es.addEventListener("ready", () => { setStatus("live"); backoff.current = 1000; });

      es.addEventListener("stream_overflow", () => {
        es?.close();
        connect();                          // immediate reconnect: server told us to
      });

      es.onmessage = () => {};              // all our events are named; ignore default

      // one handler for every named kind we care about
      for (const kind of ["run.created", "run.state", "run.error", "log",
                          "agent.text", "tool.pre", "tool.post", "finding", "gate"]) {
        es.addEventListener(kind, (ev) => {
          const data = JSON.parse((ev as MessageEvent).data) as RunEvent;
          if (data.seq <= lastSeq.current) return;
          lastSeq.current = data.seq;
          push(data);
        });
      }

      es.onerror = () => {
        es?.close();
        if (cancelled) return;
        setStatus("reconnecting");
        timer = window.setTimeout(connect, backoff.current);
        backoff.current = Math.min(backoff.current * 2, 30_000);
      };
    };

    push("reset");
    lastSeq.current = 0;
    connect();
    return () => { cancelled = true; es?.close(); if (timer) clearTimeout(timer); setStatus("closed"); };
  }, [runId, max]);

  return { events, status, lastSeq: lastSeq.current };
}
```

Note the deliberate choices: the token goes in the query string because `EventSource` cannot set headers (a `fetch`-based polyfill is available if we later need headers — parked); `lastSeq` is a ref so reconnects don't re-render; the reducer caps memory at 5 000 events and the full history stays queryable from `/api/runs/{id}/events?after=0` or a paged REST endpoint added in Phase 5.

**Layout shell.** `AppShell` = a 220 px left rail (LazyBoy wordmark, nav: Inbox / Runs / Settings, a connection dot fed by `/api/health`) + a content area with a sticky header (breadcrumb, run status pill, cost meter placeholder) + `<Outlet/>`. Routes: `/` → Inbox (Phase 2 fills it; Phase 0 shows an empty state with a "Start demo run" button), `/runs/:id` → RunView (a two-pane layout: event timeline left, report/diff pane right — both stubs), `/settings` → Settings (Phase 1).

Tailwind theme: a small token set defined as CSS variables so light/dark is a class flip, not a conditional in every component.

```css
/* web/src/index.css */
@import "tailwindcss";
@theme {
  --color-bg: oklch(99% 0 0);          --color-fg: oklch(22% 0.02 260);
  --color-surface: oklch(97% 0.005 260); --color-border: oklch(90% 0.01 260);
  --color-accent: oklch(58% 0.15 255);
  --color-ok: oklch(62% 0.14 150);  --color-warn: oklch(72% 0.15 75);  --color-err: oklch(58% 0.19 25);
  --font-mono: "JetBrains Mono", ui-monospace, monospace;
}
.dark {
  --color-bg: oklch(18% 0.01 260);     --color-fg: oklch(94% 0.01 260);
  --color-surface: oklch(23% 0.015 260); --color-border: oklch(32% 0.02 260);
}
```

Event rows, diffs, and log output are monospace; everything else is the system UI stack. Density over decoration — this is a cockpit, not a marketing page.

### 13. Dev workflow

```makefile
.PHONY: dev dev-api dev-web build test lint fmt clean
dev:        ; @$(MAKE) -j2 dev-api dev-web
dev-api:    ; LAZYBOY_DEV=1 uv run uvicorn lazyboy.app:dev_app --factory \
                --reload --reload-dir src --host 127.0.0.1 --port 7717
dev-web:    ; cd web && npm run dev
build:      ; cd web && npm ci && npm run build
test:       ; uv run pytest -q
lint:       ; uv run ruff check . && uv run mypy src && cd web && npm run typecheck
fmt:        ; uv run ruff format . && cd web && npm run format
clean:      ; rm -rf src/lazyboy/static dist .pytest_cache
```

`dev_app()` is a zero-arg factory that loads settings, sets a *fixed* dev session token (`LAZYBOY_DEV_TOKEN`, default `dev`) so hot reload doesn't invalidate the browser's cookie, and adds `http://localhost:5173` to the origin allowlist. It is never reachable in a packaged build (guarded on `LAZYBOY_DEV`).

Backend hot reload: `--reload-dir src` only, so Vite's writes into `static/` don't restart Uvicorn. Frontend hot reload: Vite HMR, with `/api` proxied to 7717 — including SSE, which Vite's proxy passes through unbuffered once `compress` is off (it is, by default, in dev).

---

## API (Phase 0 surface)

| Method | Path | Body / Query | Returns |
|---|---|---|---|
| GET | `/api/meta` | — | `{app:"lazyboy", version, pid, started_at}` — unauthenticated, used by the "already running?" probe |
| GET | `/api/health` | — | `{status:"ok", db:"ok", schema_version:1, runs_active:n}` |
| GET | `/api/runs` | `?state=&limit=&cursor=` | `{items:[RunSummary], next_cursor}` |
| POST | `/api/runs` | `{work_item_id?: int}` | `201 {id, state}` — Phase 0 starts the demo pipeline |
| GET | `/api/runs/{id}` | — | `RunDetail` |
| POST | `/api/runs/{id}/cancel` | — | `202 {id, state:"cancelling"}` |
| GET | `/api/runs/{id}/events` | `?after=<seq>` | `text/event-stream` |
| GET | `/api/config` | — | redacted settings (secrets never present) for the Settings screen |

---

## Tests

| File | What it pins |
|---|---|
| `test_config.py` | seeding, yaml→Settings, env override precedence, `$LAZYBOY_HOME` |
| `test_migrations.py` | 0→1 on empty DB, idempotent re-run, future-version abort |
| `test_events.py` | seq monotonicity, emit-before-subscribe visible via replay, queue overflow → sentinel + unsubscribe, concurrent emitters don't interleave seqs |
| `test_sse_replay.py` | **the important one.** Emit 50 events, connect with `after=20`, assert exactly 21…50 arrive in order; kill the connection at 35, reconnect with `Last-Event-ID: 35`, assert 36… with no dupes; assert a heartbeat comment arrives within 20 s (clock patched) |
| `test_run_manager.py` | start→done; cancel mid-pipeline lands `cancelled` within 2 s; exception lands `failed` with a `run.error` event; `shutdown()` cancels everything |
| `test_crash_recovery.py` | insert a `running` row directly, boot the app, assert `failed`/`orphaned_by_restart` |
| `test_security.py` | no token → 401; wrong token → 401; `?t=` sets cookie + redirects; bad `Origin` → 403; `/api/meta` exempt |
| `test_spa_fallback.py` | `/`, `/runs/x`, `/assets/app-abc.js`, `/api/nope` (JSON 404), `/../../etc/passwd` (404, no traversal) |
| `test_redaction.py` | each pattern in `SECRET_PATTERNS` against a realistic sample; lazy-arg case |
| `test_launcher.py` | `_bind` falls back to ephemeral when the preferred port is taken; token is 43 chars and differs per call |
| `web/src/hooks/useRunStream.test.ts` | vitest + a fake `EventSource`: dedupe on `seq`, backoff schedule, reset on `runId` change |

Fixtures: `app_factory(tmp_path)` builds an isolated `Settings` with `LAZYBOY_HOME=tmp_path`; `client` is `httpx.ASGITransport` (no sockets); SSE tests use `httpx_sse.aconnect_sse`. Target ≥ 85 % line coverage on `core/` and `security.py`; the rest is best-effort.

---

## Risks & mitigations

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | SSE buffered by an intermediary (corporate proxy PAC, AV web shield) → UI looks frozen | high | `X-Accel-Buffering: no`, 15 s heartbeat, and an explicit "reconnecting" pill in the UI. If a user's AV still buffers, a `?transport=poll` fallback (poll `?after=` every 2 s over plain JSON) is a 30-line addition — the replay API already supports it. |
| R2 | SQLite `database is locked` under concurrent emit + read | medium | WAL + `busy_timeout=5000` + a single async write lock in `EventBus`; all writes go through `to_thread`, never on the loop |
| R3 | Event log grows unbounded on long runs | low | events are cheap (~200 B), but Phase 9 adds retention (`logging.retain_days`) and a `VACUUM` on startup if the DB exceeds 200 MB |
| R4 | Browser opens before the server is listening → error page | medium | we bind the socket *before* Uvicorn starts, and delay the browser 400 ms; worst case the user hits refresh |
| R5 | Port 7717 occupied by another tool | low | ephemeral fallback + `/api/meta` probe to detect a second LazyBoy |
| R6 | Redaction filter is a blocklist and will miss a novel secret shape | medium | never log request bodies or headers wholesale; the connectors (Phase 1) log a token's *fingerprint* (`sha256[:8]`) rather than the token; filter is defence-in-depth, not the primary control |
| R7 | `--reload` restarts kill live runs during development | low | accepted; dev pipeline is a demo. Real runs are resumable from Phase 5 (`resume=<session_id>`) |
| R8 | Shipping a wheel without `static/` (forgot `make build`) | medium | `mount_spa` degrades to a 501 with the exact remedy; Phase 9 adds a hatch build hook that fails the build if `static/index.html` is absent |
| R9 | Global `AUTOINCREMENT` seq shared across runs confuses a future multi-run UI | low | documented: seqs are comparable only within a run; the inbox uses `run.updated_at`, not seq |

---

## Effort

**0.5–1 day** (matches the master estimate).

| Chunk | Est. |
|---|---|
| Scaffold, pyproject, Makefile, CI lint job | 1 h |
| Config + paths + logging/redaction | 1 h |
| DB + migrations + models | 1 h |
| EventBus + SSE + replay tests | 2 h |
| RunManager + cancellation + crash recovery | 1.5 h |
| Launcher + SPA mount + security middlewares | 1.5 h |
| Frontend scaffold, shell, `useRunStream`, Tailwind theme | 2 h |

**Next:** [Phase 1 — Identity & Credential Vault](phase-1-identity.md).
