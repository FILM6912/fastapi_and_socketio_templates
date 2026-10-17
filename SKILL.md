---
name: fastapi-and-socketio-templates
description: >-
  Scaffold a FastAPI plus Socket.IO backend in the line-app layout: routers,
  schemas, services, repositories, hardware, and workers on one ASGI process,
  with F-Docs at /docs. Use when the user asks for fastapi_and_socketio_templates,
  a FastAPI and Socket.IO template, F-Docs, f-docs, or a station/line backend
  shaped like the Celestica app.
---

# FastAPI and Socket.IO Templates

Build one process that serves REST and Socket.IO. Routers stay thin. Services own the rules. Hardware and workers own devices and loops.

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
├── hardware/               # device clients (PLC, scanner, RFID)
├── workers/                # long-running loops started from lifespan
├── socketio/               # AsyncServer, namespaces, emitters, rooms
├── dependencies/           # auth and other request guards
├── middleware/
└── static/
    └── fdocs/              # vendored f_docs.js from github.com/FILM6912/f-docs
```

`main.py` imports routers and calls `include_router`. It does not contain endpoint bodies.

## Rules

1. One area, one router file. Declare the prefix on the router.

```python
router = APIRouter(prefix="/api/v1/stations", tags=["stations"])
```

2. A router validates input, calls a service, and returns the envelope. It does not open a database connection or a device socket itself.
3. A service raises `BusinessError(error_code, message)`. It does not build `HTTPException` or a Socket.IO packet.
4. A repository reads and writes one table. It does not decide workflow.
5. A hardware module speaks one device. A worker loops. Both start in `lifespan` and shut down in the same function.
6. A Socket.IO handler parses the event, calls a service, and returns an ack. Emit through `socketio/emitters.py`, not from a router.
7. Mount Socket.IO around FastAPI. Mount static files and a SPA last, so `/api`, `/health`, `/docs`, and `/socket.io` win.
8. Settings come from environment variables through `config.py`. No secrets in code.
9. `/docs` is F-Docs by default. `DOCS_UI=swagger` is the only switch that restores builtin Swagger. Mount docs before the SPA.

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

Put namespaces in `socketio/namespaces.py`. Typical split:

| Namespace | Who connects | What it does |
|-----------|----------------|--------------|
| `/` | dashboard | subscribe and coarse status |
| `/ui` | operator screen | join a side, send commands |
| `/daemon` | device daemon | telemetry and command acks |
| `/admin` | admin screen | override commands |

Room names live in `socketio/rooms.py`. Server-to-client events go through one emitter so the envelope stays `{ command_id, station_id, side, payload, timestamp }`.

## Service and repository

```python
class BusinessError(Exception):
    def __init__(self, error_code: str, message: str, result: dict | None = None):
        self.error_code = error_code
        self.message = message
        self.result = result or {}

class StationService:
    def __init__(self, stations):
        self.stations = stations

    def get(self, station_id: str) -> dict:
        row = self.stations.get_by_id(station_id)
        if row is None:
            raise BusinessError("STATION_NOT_FOUND", "Station not found")
        return row
```

Inject the service with `Depends` from `deps.py`. Do not construct repositories inside the endpoint.

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

## Docker Compose

One `server` service runs `uvicorn app.main:app`. Give it `env_file: .env`. Add Redis when the app caches or shares pub/sub. Add a mock service per device the hardware layer can run without the real machine.

Publish the API port. Do not publish Redis. Pass `REDIS_URL` as `redis://:${REDIS_PASSWORD}@redis:6379/0` inside the compose network.

## Checklist for a new area

- [ ] `schemas/<area>.py` for the body and the response data
- [ ] `services/<area>_service.py` for the rules
- [ ] `services/repositories/<area>_repo.py` when a table is involved
- [ ] `routers/<area>.py` with `APIRouter(prefix="/api/v1/<area>")`
- [ ] `include_router` in `main.py`
- [ ] An example in `openapi/examples.py` when the derived example would be wrong
- [ ] Socket event in `socketio/namespaces.py` only if a client must push or receive live data
- [ ] Worker in `workers/` only if the work continues while no request is in flight
- [ ] Hardware client in `hardware/` only if a device protocol is involved

## Do not add

- A second public URL prefix for the same endpoint
- Business branches inside the Socket.IO callback
- Direct table calls from a router
- Secrets, host PLC addresses, or customer URLs in the template
- A SPA mount that is registered before the API routers or before `/docs`
- Swagger and F-Docs both registered on `/docs`
- A docs bundle fetched from the network at request time
