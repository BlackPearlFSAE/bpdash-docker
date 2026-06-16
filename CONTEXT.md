# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Docker Compose stack (`compose.yml`) that wires together three services: **db** (Postgres 16), **backend** (Node.js/Express), and **frontend** (React/Vite served via nginx). The `frontend/` and `backend/` directories are separate git repos cloned in place.

## Commands

### Full stack (Docker)
```bash
docker compose up --build          # build + start all services
docker compose up -d --build       # detached
docker compose logs -f backend     # tail backend logs
docker compose restart <service>   # restart one service
docker compose down                # stop (data volume preserved)
docker compose down -v             # stop + wipe db_data volume
```

### Backend (run locally without Docker)
```bash
cd backend
npm install
npm run dev        # nodemon hot-reload
npm start          # plain node
```
Backend requires a running Postgres and `DATABASE_URL` set (or a `.env` file). For schema changes during local dev, uncomment `sequelize.sync({ alter: true })` in `server.js`.

### Frontend (run locally without Docker)
```bash
cd frontend
npm install
npm run dev        # Vite dev server, proxies /api and /ws to localhost:3000
npm run build      # production build
npm run lint       # ESLint
npm run preview    # serve the production build locally
```

There are no test suites in either package.

### Environment variables (root `.env` next to `compose.yml`)
| Variable | Default | Notes |
|---|---|---|
| `POSTGRES_USER` | `bpdash` | |
| `POSTGRES_PASSWORD` | `bpdash` | Change for any real deployment |
| `POSTGRES_DB` | `bpdash` | |
| `PUBLISH_INTERVAL` | `200` | WS broadcast throttle in ms |
| `FRONTEND_URL` | `http://localhost:8080` | CORS allowlist |
| `FRONTEND_DEPLOY_URL` | _(empty)_ | Second CORS origin for prod |

---

## Architecture

### Data flow (end-to-end)

```
Vehicle MCU (ESP32)
    │  WebSocket /ws  (no role param)
    ▼
backend/server.js
    │  utils/dataProcessor.js — flatten arrays + apply SCALE_CONFIG
    ├──► WS broadcast (throttled per client, backpressure guard) ──► Browser /ws?role=dashboard
    │
    └──► in-memory buffer (1 s flush) ──► PostgreSQL via Stat.bulkCreate()
                                              │
                          GET /api/session/:id/data?normalized=true
                                              ▼
                                        Browser (history playback)
```

The backend is the **only place normalization happens** — both the live WS path (`normalizeTelemetry()`) and the history REST path (`normalizeStatRecord()`) run through the same `SCALE_CONFIG` in `backend/utils/dataProcessor.js`. The frontend receives identically-shaped flat objects in both cases and never scales raw values itself.

### Backend (`backend/`)

- **`server.js`** — entry point. Initializes Sequelize, WebSocket server, DB flush interval, heartbeat, and mounts routes.
- **`routes/sessionRoutes.js`** — REST endpoints for recording control. Exports `activeSession` (in-memory singleton) and `setActiveSession`; `server.js` imports these to know whether to buffer WS messages to DB.
- **`routes/statRoutes.js`** — REST endpoints for raw stat records.
- **`models/stat_schema.js`** / **`models/session_schema.js`** — Sequelize model definitions.
- **`utils/dataProcessor.js`** — `normalizeTelemetry()` (live path) and `normalizeStatRecord()` (history path). `SCALE_CONFIG` here is the single source of truth for sensor scaling.
- **`datagen/datagen.py`** — Python simulator that acts as a vehicle MCU. Publishes realistic telemetry over WebSocket with offline buffering and per-group `seq` counters.

Key backend behaviors:
- DB writes only happen when `activeSession` is set (recording started via `POST /api/session/start`).
- `flushDbBuffer` runs every 1 s; a guard skips a tick if the previous flush hasn't finished.
- Dashboard clients are tracked in `dashboardClients` (Set); device clients are not tracked.
- Per-group broadcast throttling: each dashboard client tracks `_lastSentPerGroup[group]`.
- Backpressure: clients whose `ws.bufferedAmount > 64 KB` are skipped for that message.
- Heartbeat: 30 s ping/pong; no pong → `terminate()` + eviction.
- Sequence gap detection: `lastSeqByStream` map tracks `${publisherId}::${group}` → last seq. Gaps are logged but messages are still accepted.

### Frontend (`frontend/src/`)

**Context providers** (wrap the entire app in `App.jsx`):
- `ThemeContext` — dark/light CSS theming.
- `TelemetryConfigContext` — user preferences for sensor display.
- `SessionContext` — recording state (`isRecording`, `activeSession`). Polls `GET /api/session/active` every 5 s to keep tabs in sync.

**Telemetry entry point:**
- `utils/websocket.js` — opens `/ws?role=dashboard`, auto-reconnects every 2 s.
- `hooks/useTelemetryStream.js` — buffers all WS messages into a ref (no per-message re-render), flushes to React state at ~10 fps. Tracks `wsStatus` and `isStale` (no data for 10 s). Shared by Pitwall, Dynamics, Powertrain, and Battery pages.

**Page layouts — three patterns** (see `frontend/docs/component-patterns.md`):
1. **DataGroupPanel** (`DynamicsPage`, `PowertrainPage`, history playback in `HistoryPage`) — filters telemetry stream by `groups` array, renders `ChartSection` (time-series chart) + `TableSection` (raw data table).
2. **Pitwall widgets** (`PitwallPage`) — extracts latest values via `useLatest()`/`useLatestMulti()`, renders purpose-built widgets (`PowerGauge`, `BarGauge`, `StatCard`, `FaultBar`, `TrendChart`, `GPSTrack`).
3. **Battery page** (`BatteryPage`) — BMU selector dropdown; each BMU has `BMSCellChart`, `BMSTempChart`, `BMSFaultTable` from `components/battery_widget/`.

**Data group definitions** (`utils/dataGroups.js`): maps page categories to backend group names. Groups follow `<node>.<type>` naming (`front.mech`, `bmu3.cells`, etc.). When the firmware sends a `node` field, the backend prepends it: `node="front"` + `group="faults"` → `"front.faults"`.

**Sensor display names** (`utils/sensorDisplayNames.js`): maps raw sensor keys to human-readable labels. Used in chart legend and table headers. Unmapped keys fall through unchanged.

**History playback** (`HistoryPage`): loads session data via paginated `GET /api/session/:id/data?normalized=true` (10 k rows/page). `PlaybackControls` filters the loaded dataset by timestamp; the filtered subset is passed to `DataGroupPanel`. `ExportControls` converts to JSON/CSV client-side.

### nginx (`frontend/nginx.conf`)
In production (Docker), nginx serves the built frontend and proxies `/api` and `/ws` to the backend container at `http://backend:3000`. In dev, Vite's proxy config in `vite.config.js` does the same for `localhost:3000`.

### Adding a new sensor
1. Add scaling to `SCALE_CONFIG` in `backend/utils/dataProcessor.js` if the raw value needs conversion.
2. Add a human-readable label to `frontend/src/utils/sensorDisplayNames.js`.
3. If it's a new group, add it to the relevant entry in `frontend/src/utils/dataGroups.js`.

### HTTPS / TLS
Currently disabled. Port 80 only. When enabling, see `docs/tls-auto-renewal.md` and uncomment the relevant sections in `compose.yml` and `frontend/nginx.conf`.