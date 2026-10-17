---
name: fastapi_and_socketio_templates
description: >-
  Scaffold a FastAPI plus Socket.IO backend in the line-app layout: async
  routers, schemas, services, repositories, hardware, and workers on one ASGI
  process, with F-Docs at /docs. PLC access uses Mitsubishi MC Protocol (3E
  frame). Apply python-pro and python-performance-optimization: async I/O,
  type hints, and profile before tuning. Use when the user asks for
  fastapi_and_socketio_templates, a FastAPI and Socket.IO template, F-Docs,
  f-docs, an MC Protocol PLC client, or a station/line backend shaped like
  the Celestica app.
---

# FastAPI and Socket.IO Templates

Build one process that serves REST and Socket.IO. Routers stay thin. Services own the rules. Hardware and workers own devices and loops. Every request path is `async def` and awaits its I/O.

Copy the layer split. Do not copy a previous product's station names, MES calls, or PLC map into a new app.

## When to use

- A new backend that needs HTTP and realtime on one port
- Reshaping a monolith into routers, services, and a Socket.IO mount
- Docker Compose for the API, Redis, and device mocks

## Layout

```
app/
├── main.py                 # FastAPI app, lifespan, exception handlers, ASGI export
├── config.py               # pydantic-settings, env aliases
├── deps.py                 # Depends() factories for settings, db, repos
├── db.py                   # database client
├── logging_setup.py
├── routers/                # one APIRouter per area
│   └── fdocs.py            # F-Docs at /docs, or Swagger when DOCS_UI=swagger
├── openapi/
│   ├── examples.py         # curated request and response examples
│   └── inject.py           # copy examples onto schema.example for F-Docs
├── schemas/                # pydantic models and the response envelope
├── services/               # business rules
│   ├── errors.py           # BusinessError and error_code to HTTP status
│   └── repositories/       # table access only
├── hardware/               # device clients (PLC via MC Protocol, scanner, RFID)
│   └── plc.py              # pymcprotocol Type3E, one socket, one queue
├── workers/                # long-running loops started from lifespan
├── socketio/               # AsyncServer, namespaces, emitters, rooms
├── dependencies/           # auth and other request guards
├── middleware/
└── static/
    └── fdocs/              # vendored f_docs.js from github.com/FILM6912/f-docs
```

`main.py` imports routers and calls `include_router`. It does not contain endpoint bodies.

## Rules

1. One area, one router file. Declare the prefix on that router as `/api/v1/<area>`. Do not hang product JSON routes on the app root or on one shared default router. Pages, `/health`, and `/docs` stay unversioned.

```python
router = APIRouter(prefix="/api/v1/stations", tags=["stations"])
```

2. A router validates input, calls a service, and returns the envelope. It does not open a database connection or a device socket itself.
3. A service raises `BusinessError(error_code, message)`. It does not build `HTTPException` or a Socket.IO packet.
4. A repository reads and writes one table. It does not decide workflow.
5. A hardware module speaks one device. A worker loops. Both start in `lifespan` and shut down in the same function.
6. A Socket.IO handler parses the event, calls a service, and returns an ack. Emit through `socketio/emitters.py`, not from a router.
7. Mount Socket.IO around FastAPI. Mount static files and a SPA last, so `/api/v1`, `/health`, `/docs`, and `/socket.io` win.
8. Settings come from environment variables through `config.py`. No secrets in code.
9. `/docs` is F-Docs by default. `DOCS_UI=swagger` is the only switch that restores builtin Swagger. Mount docs before the SPA.
10. A PLC speaks Mitsubishi MC Protocol, 3E frame, through `pymcprotocol.Type3E`. Do not use Modbus for that PLC.
11. Routes, Socket.IO handlers, lifespan hooks, and service methods that do I/O are `async def`. The caller uses `await`. A blocking call does not run on the event loop.

## Response envelope

Success:

```json
{ "success": true, "data": {}, "meta": { "request_id": "req_...", "timestamp": "..." } }
```

Error:

```json
{ "success": false, "error_code": "NOT_FOUND", "message": "...", "detail": {}, "meta": { "request_id": "req_...", "timestamp": "..." } }
```

Register handlers for `HTTPException` and `BusinessError` on the FastAPI app so both shapes stay the same. Map `error_code` to HTTP status in `services/errors.py`. Use the same codes in Socket.IO failure acks.

## App entry

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import socketio

from app.config import get_settings
from app.routers.fdocs import docs_url_for, mount_selected_docs
from app.routers.health import router as health_router
from app.socketio.server import create_asgi_app

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Start each worker on its own. One failure must not skip the rest.
    yield
    # Close hardware and reset workers here.

_docs_ui = get_settings().docs_ui
fastapi_app = FastAPI(
    title="API",
    lifespan=lifespan,
    docs_url=docs_url_for(_docs_ui),
    redoc_url=None,
)
fastapi_app.include_router(health_router)
mount_selected_docs(fastapi_app, _docs_ui)

_asgi = create_asgi_app(fastapi_app)
app = CORSMiddleware(
    _asgi,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Run `uvicorn app.main:app --host 0.0.0.0 --port 8000`. The exported `app` is the outer ASGI app, not the bare `FastAPI` instance.

## F-Docs

Interactive docs are [F-Docs](https://github.com/FILM6912/f-docs). Vendor the bundle at `app/static/fdocs/f_docs.js` and record the release in `app/static/fdocs/VERSION`. Do not load the bundle from a CDN.

`config.py` holds `docs_ui` with alias `DOCS_UI` and default `fdocs`.

`routers/fdocs.py` owns the switch:

- `normalize_docs_ui` maps `fdocs`, `f-docs`, and unknown values to `fdocs`. It maps `swagger` and `swagger-ui` to `swagger`.
- `docs_url_for` returns `None` for F-Docs so FastAPI does not register Swagger on `/docs`. It returns `"/docs"` only for Swagger.
- `mount_selected_docs` calls `mount_fdocs` only for F-Docs.
- `mount_fdocs` serves assets at `/fdocs-static` and HTML at `/docs`. The page sets `window.NEXUS_CONFIG` to `{"openApiUrl": "/openapi.json", "title": "<app title>"}` and loads `/fdocs-static/f_docs.js`. Keep a `/f_docs.js` alias for older links. Both routes use `include_in_schema=False`.

F-Docs reads `schema.example` and `schema.default`. It does not read `media.example` alone. Assign `app.openapi` to a function that calls `build_openapi_schema` and caches the result on `app.openapi_schema`. `openapi/inject.py` walks every operation and copies a parameter, request body, and success response example onto the schema object. Prefer a curated example in `openapi/examples.py`. Fall back to a value derived from the schema. A non-JSON success (CSV, binary, HTML) keeps its real media type and a string example.

Call `mount_selected_docs` after routers and before any SPA catch-all. `/openapi.json` stays on. ReDoc stays off (`redoc_url=None`).

## Socket.IO

```python
import socketio

sio = socketio.AsyncServer(
    async_mode="asgi",
    cors_allowed_origins="*",
    ping_interval=8,
    ping_timeout=8,
)

def create_asgi_app(fastapi_app):
    return socketio.ASGIApp(sio, other_asgi_app=fastapi_app, socketio_path="/socket.io")
```

Put namespaces in `socketio/namespaces.py`, one `AsyncNamespace` class each. Register all four. The default namespace is only the dashboard. Do not put operator, daemon, or admin events on `/`.

| Namespace | Who connects | What it does |
|-----------|----------------|--------------|
| `/` | dashboard | subscribe and coarse status |
| `/ui` | operator screen | join a side, send commands |
| `/daemon` | device daemon | telemetry and command acks |
| `/admin` | admin screen | override commands |

Room names live in `socketio/rooms.py`. Server-to-client events go through one emitter so the envelope stays `{ command_id, station_id, side, payload, timestamp }`.

## Service and repository

```python
import asyncio

class BusinessError(Exception):
    def __init__(self, error_code: str, message: str, result: dict | None = None):
        self.error_code = error_code
        self.message = message
        self.result = result or {}

class StationService:
    def __init__(self, stations):
        self.stations = stations

    async def get(self, station_id: str) -> dict:
        row = await asyncio.to_thread(self.stations.get_by_id, station_id)
        if row is None:
            raise BusinessError("STATION_NOT_FOUND", "Station not found")
        return row
```

```python
@router.get("/{station_id}")
async def get_station(
    station_id: str,
    service: StationService = Depends(get_station_service),
):
    return await service.get(station_id)
```

Inject the service with `Depends` from `deps.py`. Do not construct repositories inside the endpoint. If the repository client is synchronous, the service awaits `asyncio.to_thread`. It does not call that client directly.

## Lifespan

Start workers independently:

```python
async def _start_service(name: str, start) -> bool:
    try:
        await start()
        return True
    except Exception:
        logger.exception("Startup failed for %s", name)
        return False
```

Shutdown order is the reverse of startup. Close device connections before cancelling tasks.

## Async, Python, and performance

Write this app the way `python-pro` and `python-performance-optimization` require. Target Python 3.12+. Type every public function. Validate bodies with Pydantic. Format and lint with ruff. Test the async paths with pytest and `httpx.AsyncClient` or FastAPI `TestClient`. Prefer the standard library. Add a package only when this layout needs it.

A route that touches the database, the PLC, HTTP, or the filesystem is `async def` and awaits the service. A Socket.IO handler is the same. `def` is only for a pure function with no I/O.

These calls block the event loop. Do not put them in an `async def` body:

- `time.sleep`
- `requests`
- a synchronous database call
- `Type3E` methods

Run the synchronous PLC client inside the hardware queue with `asyncio.to_thread`, as the MC Protocol section says. Call other HTTP services with one shared `httpx.AsyncClient` created in lifespan and closed on shutdown. Use a connection pool for the database. Do not open a new connection per request.

Profile before you change a hot path. Time the slow request with `time.perf_counter`, or run `python -m cProfile` and sort by cumulative time. Change the function the profile names. Leave cold code alone.

On a path the profile shows is hot:

- Read many PLC words in one `batchread_wordunits` call. Do not round-trip once per word.
- Keep `get_settings` behind `lru_cache`.
- Look up by dict or set. Do not scan a list for membership.
- Stream a large export. Do not build the whole file in memory first.
- Do not copy a payload that the next function only reads.

A worker is an async task started from lifespan. CPU work that would stall the loop goes to `asyncio.to_thread` or a process pool.

## PLC (MC Protocol)

Put the client in `hardware/plc.py`. Depend on `pymcprotocol`. One PLC endpoint gets one `Type3E` client. A second PLC gets a second client, not a second socket on the first.

Settings in `config.py`:

| Env | Role |
|-----|------|
| `PLC_HOST` | PLC or plc-mock hostname |
| `PLC_PORT` | MC port. A Q/L series CPU is often `5000`. A mock may use `502`. |
| `PLC_TEST_MODE` | Skip the socket and serve an in-memory device |
| `PLC_MOCK_ON_UNREACHABLE` | On connect failure, fall into that in-memory device and log it. Otherwise raise |

Connect with `Type3E().connect(host, port)`. Then set `SO_LINGER` to on with timeout `0` so close sends RST. A Mitsubishi CPU keeps one MC client. A normal FIN can leave that slot busy.

`pymcprotocol` is synchronous. All reads and writes go through one `asyncio` queue and one worker. Run each command with `asyncio.to_thread`. A router, a service, and a Socket.IO callback must not touch the socket.

Words use `batchread_wordunits(address, count)` and `batchwrite_wordunits(address, values)`. Bits use `batchread_bitunits`. Before a word write, mask each value with `0xFFFF`. If the result is `>= 0x8000`, store `word - 0x10000`. `batchwrite_wordunits` encodes a signed 16-bit range, `-32768..32767`.

A health check writes a heartbeat word that toggles `0` and `1`. A read does not prove the CPU accepted a command.

The address map (which D or M word means what) lives in config or a repository. `hardware/plc.py` reads and writes the addresses it is given. Do not hardcode a previous product's register list.

In Compose, add a `plc-mock` service that answers MC Protocol on the port `PLC_PORT` uses. Set `PLC_HOST` to that service name on the compose network. Publish the mock port only when a tool on the host must reach it.

The mock also serves a controller page. Use the glass dashboard shell: a sticky header, a heartbeat reading, a connection badge, and one tab per sheet in the address workbook. Machine-status words are metric cards. Repeated sections such as slots and a data stack are cards in a grid. Values on the page follow the mock memory. Build the cards from that workbook. Do not copy another line's station names, jog controls, auto-run, or tower lamps onto this page.

## Docker Compose

One `server` service runs `uvicorn app.main:app`. Give it `env_file: .env`. Add Redis when the app caches or shares pub/sub. Add a mock service per device the hardware layer can run without the real machine.

Publish the API port. Do not publish Redis. Pass `REDIS_URL` as `redis://:${REDIS_PASSWORD}@redis:6379/0` inside the compose network.

## Checklist for a new area

- [ ] `schemas/<area>.py` for the body and the response data
- [ ] `services/<area>_service.py` for the rules
- [ ] `services/repositories/<area>_repo.py` when a table is involved
- [ ] `routers/<area>.py` with `APIRouter(prefix="/api/v1/<area>", tags=["<area>"])`
- [ ] `include_router` in `main.py` for that area router, not a catch-all on `/`
- [ ] An example in `openapi/examples.py` when the derived example would be wrong
- [ ] Socket event on `/`, `/ui`, `/daemon`, or `/admin` in `socketio/namespaces.py` only if a client must push or receive live data. Do not add it on the default namespace unless the dashboard is the client.
- [ ] Worker in `workers/` only if the work continues while no request is in flight
- [ ] Hardware client in `hardware/` only if a device protocol is involved
- [ ] `hardware/plc.py` using `Type3E` when a PLC is in scope, plus a `plc-mock` MC server in Compose
- [ ] `plc-mock` controller page: glass header, heartbeat, one tab per address-book sheet, no borrowed station grid

## Do not add

- A second public URL prefix for the same endpoint
- Product JSON routes outside `/api/v1/<area>`, or every area piled onto one default router
- Operator, daemon, or admin events on the default Socket.IO namespace
- Business branches inside the Socket.IO callback
- Direct table calls from a router
- Secrets, host PLC addresses, or customer URLs in the template
- A SPA mount that is registered before the API routers or before `/docs`
- Swagger and F-Docs both registered on `/docs`
- A docs bundle fetched from the network at request time
- A Modbus client for a PLC that speaks MC Protocol
- A plc-mock page that is only a raw register table, or that copies another product's stations, jog, or auto-run
- A second socket to the same PLC beside the command queue
- A `def` route that calls the database, the PLC, or HTTP
- `time.sleep` or `requests` inside `async def`
- A performance rewrite of a path that has not been profiled
