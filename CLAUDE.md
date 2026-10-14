# CLAUDE.md

This file guides Claude Code when it works in this repository.

## What this is

A self-hosted, ad-free weather PWA. It runs as one Docker container on a home server and is installed on phones by a handful of family and friends. **Smoothness is the top priority**: the app opens instantly from cache, refreshes in the background with zero layout shift, and runs gestures and radar at the display's full refresh rate.

The full plan, source research and budgets are in `docs/PLAN.md`. Read it before any non-trivial change. **Status: planning; no code yet.** Update this file as the commands and layout below become real.

## Layout (planned)

```
apps/web/        SolidJS 1.9 + Vite 8 static SPA, hand-rolled service worker (src/sw.ts)
apps/server/     Node 24 LTS + Hono 4: API, tile proxy, refresher, static hosting
packages/shared/ WeatherBundle schema and types shared by server and web
perf/            Playwright perf harness, budgets.json, fixtures
docs/            PLAN.md, perf/device-protocol.md, perf/results.md
```

## Commands (planned; pnpm workspace)

```
pnpm install
pnpm dev                 # server (with fixture or live upstreams) + Vite dev server
pnpm test                # vitest, all packages
pnpm e2e                 # playwright functional tests
pnpm perf                # perf harness against budgets.json (modern + floor profiles)
pnpm size                # size-limit bundle budgets
pnpm lint && pnpm typecheck
docker build -t adams-weather . && docker compose -f compose.example.yml up
```

Before reporting any UI or front-end change as done, run `pnpm size` and `pnpm perf`. A budget regression is a failing build, not a warning.

## Non-negotiable rules

### Smoothness and performance
- Animate only `transform` and `opacity`. Never animate layout properties. No JS scroll or touch handlers that read or write layout. Use CSS scroll-snap, `IntersectionObserver`, scroll-driven animations and View Transitions.
- **Zero layout shift.** Every data-bound element has a reserved fixed size: tabular numbers, line clamps, aspect-ratio boxes.
  - The alert strip is always present at a fixed height.
  - Placeholders must match the final layout exactly.
  - Never insert content above existing content when data arrives.
- Apply refreshed data in one batched update via `requestAnimationFrame`, and **defer it while a scroll or gesture is in progress**.
- The first screen renders from IndexedDB before any network request. Nothing on the critical path waits for the network, GPS or MapLibre.
- MapLibre is only loaded through the lazy map chunk, prefetched at idle. Never import it from the shell.
- No new runtime dependency on the web side without checking its gzip size against `size-limit`. Prefer about 50 lines of our own code over a library. No web fonts, no third-party scripts, no analytics SDKs.
- Charts are our own canvas code, drawn once per data change, never on scroll.
- Budgets live in `perf/budgets.json`; don't loosen them to make a change pass. The budgets table is in `docs/PLAN.md` §6.

### Data and upstreams
- **Browsers never call weather or tile upstreams directly.** Everything goes through the server, which caches and normalizes.
- Every upstream request sends the configured identifying User-Agent (`UPSTREAM_USER_AGENT`), goes through the per-host rate limiter and back-off, and is single-flight per cache key.
- Treat 403 and 429 from NWS or MET as back-off.
- Respect each source's terms (`docs/PLAN.md` §2):
  - Open-Meteo is non-commercial and requires attribution.
  - Nominatim: at most 1 request/second and never for autocomplete.
  - Never use `tile.openstreetmap.org` or RainViewer as primary sources.
- Key NWS data by grid cell (`wfo/x/y`), and Open-Meteo data by coordinates rounded to 0.01°. Round NWS `/points` coordinates to 4 decimals.
- Defend against nulls and shape drift in every upstream. Validate with valibot on the server, and drop null trailing hourly periods.
- The server does unit conversion (US units), icon mapping and display precomputation. The client binds values; it doesn't compute.
- Alerts are never shown stale without saying so. After 5 minutes without a successful alert poll, the UI shows "alerts unavailable".
- Fallback is per section, and each section carries its source and fetch time.

### Auth
- The app has **no login UI**. Identity comes from a trusted header set by the user's forward-auth proxy. The header name is configurable through `AUTH_USER_HEADER`, default `Remote-User`.
- The header is trusted **only** from `TRUSTED_PROXY_CIDRS`.
- `/api/*` responds `401` JSON when there's no identity, never a redirect.
- The service worker must never intercept `/auth*`, `/oauth2/*` or `?reauth=1` navigations.
- Client fetches use `redirect: 'manual'` and treat `opaqueredirect` or 401 as session-expired, which is distinct from offline.
- `AUTH_MODE=dev` (a fixed fake user) must refuse to start when `NODE_ENV=production`.
- Every DB query on user data is scoped by `user_id`.

### Radar
- Use one raster source and layer per frame, all preloaded at `raster-opacity: 0` (not `visibility: none`, which skips loading).
- Set `raster-fade-duration: 0` and source `maxzoom` around 9.
- Playback starts only when all frames are loaded.
- Add a new frame and remove the oldest; never call `setTiles` on a visible frame.
- Tile URLs are timestamped and immutable. Serve them with `immutable` cache headers, and serve empty tiles as `204`.

## Conventions
- TypeScript strict everywhere. The shared types in `packages/shared` are the API contract; change them first.
- Tests run against recorded upstream fixtures in `perf/fixtures` and `apps/server/test/fixtures`. Never hit live upstreams in CI.
- Config comes only from environment variables, documented in `compose.example.yml`. The server image holds no secrets.
- Persistent state lives in `/data`: `app.db` (SQLite) and `tiles/` (LRU-capped cache).
- When a source's terms or behavior changes, update `docs/PLAN.md` §2 with the date verified.
- Real-device perf results go in `docs/perf/results.md`, recording device, OS, build and numbers.
