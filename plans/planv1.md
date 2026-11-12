# AquaTwin — Water Treatment Digital Twin (Plan)

## Context

Build a digital twin dashboard like `project-reference.jpeg`: a dark-themed web app with an interactive, data-driven **3D water treatment plant** in the centre, surrounded by panels: filter duration comparison (A vs B bars), comparison analysis (water/power consumption), filter list with health scores and an expandable detail card, a real-time + predicted volume chart, and an outlet NTU chart.
The repo is empty (only the image). Two team members work on separate feature branches in a **monorepo** and merge to `main`, coordinated by a **shared contract**.

Decisions taken:

- **Split:** vertical by module. Member A = 3D twin + live data. Member B = analytics + charts + prediction.
- **3D:** procedural (react-three-fiber primitives), no Blender asset.
- **Data:** Python simulator writing to SQLite; same interface can later read real sensors.

## Tech stack (kept minimal)

| Layer      | Choice                                             | Why                                            |
| ---------- | -------------------------------------------------- | ---------------------------------------------- |
| Frontend   | Vite + React 18 + TypeScript                       | Fast dev, simple                               |
| 3D         | `three`, `@react-three/fiber`, `@react-three/drei` | Declarative 3D in React                        |
| Styling    | Tailwind CSS                                       | Fast responsive layout, dark theme tokens      |
| State      | Zustand (live stream) + TanStack Query (REST)      | Small, no boilerplate                          |
| Charts     | ECharts (`echarts-for-react`, canvas)              | Smooth with 6 live series + dashed predictions |
| Backend    | FastAPI + Uvicorn, Pydantic v2                     | REST + WebSocket, auto OpenAPI docs            |
| Storage    | SQLite (`sqlite3`/SQLModel)                        | Zero setup                                     |
| Prediction | NumPy (linear fit / exponential smoothing)         | Simple, explainable                            |
| Tests      | pytest, Vitest                                     |                                                |

## Monorepo layout

```
aquatwin/
├─ shared/contract/            # SHARED — changed only via a "contract" PR approved by both
│  ├─ README.md                # endpoint + message spec (human-readable)
│  ├─ types.ts                 # TS types used by the frontend
│  └─ fixtures/*.json          # sample payloads, used as frontend mocks + backend tests
├─ backend/
│  ├─ app/
│  │  ├─ main.py               # FastAPI app, CORS, mounts routers, starts simulator
│  │  ├─ config.py  db.py      # SQLite connection + `readings` table + insert/query helpers
│  │  ├─ schemas.py            # Pydantic models mirroring shared/contract/types.ts
│  │  ├─ live/        (A)      # simulator.py, ws.py, routes.py
│  │  └─ analytics/   (B)      # routes.py, service.py, predict.py
│  ├─ tests/                   # test_contract.py validates fixtures against schemas.py
│  └─ requirements.txt
├─ frontend/
│  ├─ src/
│  │  ├─ app/                  # Layout, TopBar, Sidebar, theme (Phase 0, shared)
│  │  ├─ lib/                  # api.ts, ws.ts, mocks.ts (VITE_USE_MOCKS)
│  │  ├─ store/liveStore.ts    # Zustand store fed by WebSocket
│  │  ├─ features/twin/    (A) # 3D scene + plant components
│  │  ├─ features/filters/ (A) # right-hand filter list + detail card
│  │  └─ features/analytics/(B)# duration bars, comparison card, volume + NTU charts
│  └─ vite.config.ts           # alias `@contract` -> ../shared/contract
├─ README.md  .gitignore  .editorconfig
```

## Shared contract (v1)

**Filter IDs:** `F1`–`F6` (shown as "Filter 1#"…"Filter 6#").

Types (in `types.ts`, mirrored in `schemas.py`):

```ts
type FilterId = "F1" | "F2" | "F3" | "F4" | "F5" | "F6";
type FilterStatus = "normal" | "warning" | "alarm";
interface FilterReading {
	// one filter at one instant
	id: FilterId;
	ts: string; // ISO-8601 UTC
	health: number; // 0–100 (heart score)
	filterSpeed: number; // m³/h
	volume: number; // m³/h
	head: number;
	standard: number; // m
	ntu: number; // outlet turbidity
	power: number; // kWh
	waterLevel: number; // 0–1, drives 3D water height
	status: FilterStatus;
}
interface LiveSnapshot {
	type: "snapshot";
	ts: string;
	filters: FilterReading[];
}
interface TimeSeries {
	filterId: FilterId;
	points: [string, number][];
}
interface VolumeMonitor {
	actual: TimeSeries[];
	predicted: TimeSeries[];
	predictFrom: string;
}
interface DurationCompare {
	rows: { filterId: FilterId; a: number; b: number }[];
}
interface ComparisonAnalysis {
	water: { value: number; changePct: number; preMode: number };
	power: { value: number; changePct: number; preMode: number };
}
```

Endpoints:
| Owner | Method | Path | Returns |
|---|---|---|---|
| A | WS | `/ws/live` | `LiveSnapshot` every 1 s |
| A | GET | `/api/filters` | `FilterReading[]` (latest) |
| A | GET | `/api/filters/{id}` | `FilterReading` |
| B | GET | `/api/analytics/duration?aStart&aEnd&bStart&bEnd` | `DurationCompare` |
| B | GET | `/api/analytics/comparison?start&end` | `ComparisonAnalysis` |
| B | GET | `/api/history/ntu?minutes=30` | `TimeSeries[]` |
| B | GET | `/api/monitor/volume?minutes=30&horizon=10` | `VolumeMonitor` |
| — | GET | `/api/health` | `{status:"ok"}` |

Rule: any change to `shared/contract/`, `schemas.py` or `db.py` goes in its own small PR (`contract/<change>`), both members approve, then each rebases.

## Phase 0 — Foundation (both together, ~1–2 days, directly on `main`)

1. Scaffold `backend/` (FastAPI, `/api/health`, CORS) and `frontend/` (Vite React TS, Tailwind, ESLint/Prettier).
2. Write `shared/contract/` (types, README, fixtures) + `backend/app/schemas.py` + `test_contract.py`.
3. `db.py`: `readings(ts, filter_id, health, filter_speed, volume, head, standard, ntu, power, water_level, status)` with index on `(filter_id, ts)`; helpers `insert_readings()`, `query_range()`.
4. App shell: `Layout` with TopBar ("AquaTalk-Digital Twin", icons), icon Sidebar, and named slots: `<TwinSlot/>` (full-bleed background), `<LeftPanels/>`, `<RightPanels/>`, `<BottomCharts/>`. Each member only fills their slot → near-zero merge conflicts.
5. `lib/api.ts`, `lib/ws.ts` (auto-reconnect), `lib/mocks.ts` (fixture-based; `VITE_USE_MOCKS=true` lets each member's UI run without the other's backend).
6. Then create branches `feature/twin-live` (A) and `feature/analytics` (B).

## Member A — `feature/twin-live` (3D twin + live data)

**Backend (`backend/app/live/`)**

- `simulator.py`: asyncio background task (started in FastAPI lifespan), ticks every 1 s, generates realistic per-filter values (base + sine daily pattern + noise; NTU random walk; occasional degraded filter to show warning/alarm), derives `health` and `status`, calls `db.insert_readings()`, pushes to an in-memory broadcaster. Seeds 7 days of history on first run so B has data.
- `ws.py`: connection manager, broadcast `LiveSnapshot` to all clients.
- `routes.py`: `/api/filters`, `/api/filters/{id}`.

**Frontend**

- `store/liveStore.ts`: Zustand store `{ snapshot, selectedFilter, setSelected }` fed by `ws.ts`.
- `features/twin/` (procedural plant, matching the reference):
  - `TwinCanvas.tsx` — `<Canvas>` lazy-loaded, `dpr={[1, 2]}`, `OrbitControls` with clamped angles/zoom, isometric-ish default camera, fog + environment lighting, dark background.
  - `Platform.tsx` (concrete slabs), `Tank.tsx` (cylinders: chemical tanks, chimney), `Pool.tsx` (6 filter basins as boxes with translucent animated water plane), `Pipe.tsx` (tube along a path using `CatmullRomCurve3`/`TubeGeometry`, emissive cyan), `Pump.tsx`, `Railing.tsx` (`InstancedMesh`).
  - `plantLayout.ts` — single config of positions per filter/pipe, so geometry is data-driven.
  - **Dynamic behaviour:** water height ← `waterLevel`; water tint ← `ntu`; pipe flow = scrolling emissive texture offset ∝ `volume`; status glow (cyan/orange/red); hover highlight + `Html` tooltip; click basin → `setSelected` + camera fly-to.
  - Read live values inside `useFrame` via `useLiveStore.getState()` (no React re-render per tick).
- `features/filters/`: right panel list — heart icon + health score (coloured by status), filter dropdown; expanded card shows Filter Speed, Volume, Head, Standard, NTU (as in reference). Selecting in list ↔ selecting in 3D stay in sync via the store.

**Milestones:** A1 simulator + WS + REST → A2 static plant geometry → A3 live bindings + interactions → A4 filter panel + perf pass.

## Member B — `feature/analytics` (analytics, charts, prediction)

**Backend (`backend/app/analytics/`)**

- `service.py`: SQL aggregations over `readings`:
  - duration compare: total volume per filter in range A vs range B;
  - comparison: water & power totals in range, `changePct` vs previous equal-length period, `preMode` = previous period's value;
  - NTU history: per-filter series, downsampled (bucket avg) to ≤ 120 points.
- `predict.py`: per filter, fit last N minutes of `volume` (NumPy `polyfit` deg 1–2 or Holt exponential smoothing) → `horizon` future points.
- `routes.py`: endpoints from the contract table, with query validation and sensible defaults.

**Frontend (`features/analytics/`)**

- `DurationCompare.tsx`: "Duration Time A / B" pills (date-range pickers), mirrored horizontal bars per filter with values (pure CSS bars, no chart lib needed).
- `ComparisonCard.tsx`: date-range input, Water Consumption (m³) & Power Consumption (kWh) with ↓/↑ % and "Pre Mode" value.
- `VolumeMonitorChart.tsx`: ECharts lines — solid actual, dashed predicted, vertical "Predict Point" markLine, toggleable 1#–6# legend.
- `OutletNtuChart.tsx`: 6 live lines, toggleable legend; appends latest NTU from `liveStore` each tick, rolling 30-min window.
- Shared `chartTheme.ts` (dark palette, 6 filter colours matching legend chips), `Panel.tsx` (glass card used by both members — create in Phase 0 if possible).
- TanStack Query with `refetchInterval` (30–60 s) for aggregates.

**Milestones:** B1 aggregation + prediction APIs (+ pytest) → B2 left panels → B3 bottom charts → B4 date pickers, empty/loading/error states.

## Responsive & performance

- **≥1280 px:** reference layout — 3D full-bleed, panels overlay as translucent cards (left, right, bottom).
- **768–1279 px:** side panels become collapsible drawers; charts in a 2-col row under the canvas.
- **<768 px:** sidebar → bottom/ hamburger nav; 3D canvas at ~45 vh; panels stacked as cards; filter list as horizontal chips.
- Performance: lazy-load `TwinCanvas` and ECharts (code-split); `InstancedMesh` for repeated parts; merge static geometry; low-poly segments; drei `PerformanceMonitor`/`AdaptiveDpr` to drop quality on weak GPUs; pause rendering when tab hidden; WS → store with selector subscriptions so only consuming components re-render; charts update at most 1×/s; downsample history server-side.

## Git workflow

- `main` protected: PR + 1 review + CI green (lint, `pytest`, `vitest`, `vite build`).
- Branches: `feature/twin-live` (A), `feature/analytics` (B); optional short sub-branches merged into these.
- Rebase on `main` daily; merge to `main` at each milestone (small PRs, not one big one at the end).
- Contract changes only via `contract/*` PRs (see above).
- Conventional commits (`feat(twin): …`, `feat(analytics): …`).

## Phase 5 — Integration (both, ~2 days)

Turn off mocks, run full stack, verify selection sync, live NTU in both 3D tint and chart, responsive pass, README with run instructions, optional `docker-compose.yml`.

## Verification

- Backend: `cd backend && pip install -r requirements.txt && uvicorn app.main:app --reload`; open `/docs` and exercise every endpoint; `pytest` (contract fixtures validate against `schemas.py`, aggregation + prediction unit tests on a seeded temp DB).
- WebSocket: browser DevTools → Network → WS shows a `snapshot` each second.
- Frontend: `cd frontend && npm i && npm run dev`; with `VITE_USE_MOCKS=true` and false. Check: 3D water/pipes animate, clicking a basin selects the filter in the right panel and vice versa, charts update live, dashed prediction after the Predict Point.
- Responsive: DevTools device modes (iPhone, iPad, 1080p); no horizontal scroll.
- Performance: `npm run build` bundle check, Lighthouse, ~60 fps on a laptop iGPU (r3f-perf / Chrome FPS meter).
