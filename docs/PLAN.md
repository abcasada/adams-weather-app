# Adam's Weather: Build Plan

Status: **Draft, awaiting approval.** Research verified 2026-10-14. Open questions are in §10.

## 0. Goals, in priority order

1. **Smoothness.** The app opens instantly from cached data and refreshes in the background with zero layout shift. Gestures, scrolling, transitions and radar run at the display's full refresh rate (120 Hz on current phones).
2. **Radar and maps that never lag.** Rendering is GPU-accelerated, frames are preloaded, and you can pan or zoom while the animation plays.
3. **Measured, not assumed.** Concrete budgets are enforced in CI and confirmed on a real phone.
4. **Free, permitted data**, fetched once on the server and shared by every user.
5. **Layout built around use.** The first screen shows today and any active alerts. Hourly, Daily and Radar are each one tap or one swipe away.
6. **A good homelab citizen.** One small Docker container that idles near zero CPU. Auth comes from your shared login layer.

---

## 1. Recommended stack

| Layer | Choice | Why (smoothness first) |
|---|---|---|
| UI framework | **SolidJS 1.9** (stable) as a static SPA, built with **Vite 8** (Rolldown) | Fine-grained reactivity with no virtual DOM. A data refresh updates only the text nodes that changed, so there's no re-render cascade mid-gesture. It has the smallest measured runtime: about 4.3 KB gzip, versus about 10 KB for Svelte 5. No SSR or meta-framework is needed, because the service worker serves the shell. Solid 2.0 is still in RC, so we start on 1.9 and migrate later. Svelte 5 is the fallback choice if you prefer it. |
| Language | TypeScript everywhere; shared types package | One `WeatherBundle` schema used by both server and client. |
| Styling | Plain CSS with custom properties; system font stack; inline SVG icon sprite | No CSS-in-JS runtime and no web-font loading, which also means no font-swap layout shift. |
| View pager | **Native CSS scroll-snap** horizontal container | The browser's compositor thread drives the swipe, so momentum and snapping stay smooth even if JS is busy. No JS gesture library. |
| Transitions | CSS transforms and opacity only; scroll-driven animations (iOS 26+, Chrome 115+); View Transitions for sheets | Compositor-only properties never trigger layout. Older devices simply get no animation. |
| Charts | **Custom canvas** for the hourly temperature line and precipitation-chance bars (about 2 KB) | uPlot is about 23 KB gzip, bigger than the whole app shell. We draw once per data change and never redraw while scrolling. The scrub cursor is a separate layer moved with `transform`. |
| Maps | **MapLibre GL JS 6.x** (WebGL2, current 6.14), **lazy-loaded** | GPU-rendered vector basemap and raster radar. It costs about 285 KB gzip plus a 143 KB worker, so it never sits in the first-screen bundle. It's prefetched at idle so Radar opens instantly. |
| Service worker | **Hand-rolled** (about 150 lines), precache manifest injected at build time | Full control over caching. Workbox is slow-moving and heavier than we need. |
| Client storage | IndexedDB (`idb-keyval`, under 1 KB) for the last bundle per location; `navigator.storage.persist()` | Instant render on open. The cache is always treated as rebuildable. |
| Backend | **Node 24 LTS** (move to 26 LTS after 2026-10-28) + **Hono 4** | Tiny and fast, with web-standard Request/Response. It serves both the API and the static build from one container. |
| Database | **SQLite** via `better-sqlite3` (stable API, prebuilt binaries) | Stores users, saved locations, push subscriptions, the persistent upstream cache and RUM samples. One file on a volume. |
| Upstream validation | `valibot` on the **server only** | Upstream APIs change shape (NWS has several times). We fail loudly on the server instead of shipping bad data to phones. |
| Push (later) | `web-push` (VAPID) | Works for Android Chrome and for iOS home-screen apps (16.4+). |
| Packaging | One multi-stage Dockerfile (`node:24-bookworm-slim`), `/data` volume, healthcheck | Simple to run behind your existing reverse proxy. |
| Testing | Vitest (unit), Playwright 1.64 (e2e and perf harness), Lighthouse 13 run directly, `size-limit` 14, `web-vitals` 6 | See §6. |

### Rejected alternatives

| Option | Reason |
|---|---|
| React / Next.js | Larger runtime, and VDOM reconciliation runs on every refresh. |
| SvelteKit 3 / SolidStart | Released weeks ago, and SSR adds nothing for an installed PWA. |
| Chart.js / ECharts | Far over the bundle budget. |
| Leaflet | DOM- and CSS-transform-based, not GPU-rendered. It stutters with many raster layers. |
| deck.gl | Unnecessary for raster radar. It remains an option for a later custom layer. |
| Bun / Deno / Go | Fine choices, but no measurable benefit at family scale. Node + Hono keeps one language. |

---

## 2. Data sources and limits (verified 2026-10-14)

Browsers never call upstream services directly. Everything goes through our backend, which sends one identifying User-Agent (`AdamsWeather/1.0 (+https://<your-domain>; <contact-email>)`), caches responses and normalizes them.

### 2.1 Forecast and alert data

| Source | Use it for | Auth | Limits and terms | Backend TTL |
|---|---|---|---|---|
| **NWS API** (api.weather.gov, API v3.12.2) | **Primary for US.** `/points` gives the grid cell, zones, nearest city and sunrise/sunset. Also hourly forecast (temperature, precip chance), 12-hour periods with text, raw grid (gusts, sky cover, QPF, heatRisk), station observations, **alerts**, Area Forecast Discussion. | None. A **User-Agent with contact is required**; without one you get an Akamai 403. | Public domain. The rate limit is unpublished ("reasonable"); errors clear in about 5 s. Treat 403 and 429 as back-off. Points outside the US return 404. | `/points` 7 days (re-resolve on 404); hourly and 12-hour 30–60 min (respect `Expires`); raw grid 60 min; observations 10 min; alerts **60 s**; AFD 30–60 min |
| **Open-Meteo** | **Fallback when NWS fails, coverage outside the US**, `minutely_15` precipitation for the next 2 hours (native HRRR in the continental US), days 8–16, air quality (CAMS, coarse) | None | **Non-commercial use only. A private, ad-free family app qualifies explicitly.** CC BY 4.0, so we show "Weather data by Open-Meteo.com". Limits are 600/min, 5,000/hour and 10,000/day **per IP**; requests with more than 10 variables count as more than one call. Exceeding them returns 429. | 15–30 min (`minutely_15`: 15 min); air quality 60 min |
| **SPC outlooks** (spc.noaa.gov GeoJSON) | Day 1–3 convective outlook (shown in the overview "Today: Slight risk" chip, and as a map layer); Day 4–8 | None | Public domain. **Use the `cig*` files**; the `sig*` hatching was retired around 2026-03-03. | 15–30 min |
| **WPC Excessive Rainfall Outlook** | Map layer, Day 1–5 | None | Public domain | 30 min |
| **NHC** `CurrentStorms.json` | Tropical cone and track layer (KMZ converted to GeoJSON on the server) | None | Public domain | 30 min |
| Open-Meteo Geocoding | Location search as you type (debounced) | None | Same terms as Open-Meteo; GeoNames data | 30 days per query |
| NWS `/points` `relativeLocation`; US Census geocoder | Naming "current location" (US) | None | Public domain, no stated limits | Keyed by coordinates rounded to 0.01° |
| Nominatim (OSM) | Reverse-geocode fallback for non-US points **only** | None | **At most 1 request/second, no autocomplete, caching required**, attribution required | 30 days |

**Considered and not used:**

| Source | Reason |
|---|---|
| MET Norway | No precipitation chance for US points. It's a possible third fallback. |
| Pirate Weather | Needs a key, 10,000 calls/month. Open-Meteo `minutely_15` covers it. |
| OpenWeatherMap | One Call is now pay-per-call. |
| Tomorrow.io | Commercial terms and low limits. |
| AirNow | Better US AQI than CAMS, but needs a key. **See question Q7.** |

**Load estimate.** The backend polls only the *hot set*: every saved location of every user, plus locations viewed in the last 24 h. Forecasts are keyed by **NWS grid cell** (about 2.5 km), so family members in the same town share one cache entry. Take 15 hot locations: alerts every 60 s plus forecasts every 30 min is roughly 25,000 NWS requests/day and fewer than 1,500 Open-Meteo calls/day. That is far under Open-Meteo's 10,000/day cap, and the NWS load is no higher than one person leaving weather.gov open. Alerts are polled per *zone union*, as one `/alerts/active?zone=…` request, so in practice it's a few thousand requests/day.

**Known upstream quirks we handle:**
- NWS hourly periods sometimes have null temperature or precip chance at the end of the window (since about July 2026), so we filter them.
- Akamai can serve a stale forecast, so we compare `updateTime`/`generatedAt` and keep the newer copy.
- NWS doesn't return 304s, so we cache by `Expires`.
- NWS `/points` coordinates must have at most 4 decimals; with more you get a 301.
- Only about 6% of active alerts carry a polygon. The rest reference zones, so we cache zone geometries.
- The NWS raw grid is in SI units, so we convert on the server.

### 2.2 Map and tile sources

| Layer | Primary | Fallback | Notes |
|---|---|---|---|
| **Radar** | **Iowa Environmental Mesonet** NEXRAD N0Q mosaic, `ridge::USCOMP-N0Q-YYYYMMDDHHMI` via `/c/tile.py` | **NWS opengeo** `conus_bref_qcd` WMS with a time dimension (MRMS QC'd reflectivity, about 2-minute steps, 2 h window); then **nowCOAST** MRMS mosaic (about 4-minute steps, 6 h) | IEM: public domain, no key, CORS `*`. **Timestamped URLs never change**, which makes them ideal for preloading and caching forever. Frames every 5 min; useful detail up to about z9, MapLibre overzooms past that. There's no SLA (academic service) and no published rate limit, so the backend caches and is polite. NOAA WMS renders slowly (0.4–1.1 s per tile), so it always goes through the backend cache. |
| **Satellite** | **nowCOAST GOES-19/18** infrared, visible and water-vapor WMS-T (5-minute steps, about 7 min latency, 6 h history) | **NASA GIBS** GOES GeoColor WMTS (true color, about 50 min latency, max z7) | Public domain. Infrared works day and night and animates the same way as radar. |
| **Severe overlays** | NWS alert polygons (zone shapes resolved and cached); SPC Day 1–3 outlooks and active watches (IEM `spcwatch.py` GeoJSON); SPC mesoscale discussions; WPC ERO; NHC cone; local storm reports (IEM `lsr.geojson`) | NOAA `mapservices.weather.noaa.gov` FeatureServers (always bbox-filtered; one unfiltered query returned 83 MB) | All vector GeoJSON, simplified on the server. These render as GPU line and fill layers. |
| **Lightning** | nowCOAST `ldn_lightning_strike_density` (15-minute raster bins) | GOES-19 GLM raw data (phase 9, optional) | The density product is NOAA-redistributed. Exact redistribution terms aren't confirmed, so it's fine for private use and stays off by default. |
| **Basemap** | **OpenFreeMap** vector tiles (`dark` / `positron` styles): no key, no request limits, attribution required | **Self-hosted Protomaps PMTiles** US extract (`@protomaps/basemaps` dark/light flavors) | See question Q2. Phones away from home would pull self-hosted tiles through your home upload bandwidth. OpenFreeMap avoids that, and the service worker caches the basemap tiles you look at often, so your home area is local after the first view. |

**Not recommended:**
- **RainViewer.** Since 2026-01-01 it's capped at z7, 10-minute frames, one color scheme, no nowcast, and 100 requests/min per IP shared across all our users. Personal use only.
- **tile.openstreetmap.org.** Its policy forbids prefetching and proxy caching.
- **SSEC RealEarth.** Watermarks beyond a quota.
- **MapTiler / Stadia free tiers.** Keys and monthly caps, with no advantage over OpenFreeMap.

**Phase 9 (optional), our own radar rendering.** Pull MRMS GRIB2 from AWS `noaa-mrms-pds`: about 1 MB every 2 minutes, available about 1 minute after valid time. The server encodes reflectivity into terrain-RGB-style tiles, and MapLibre's built-in `color-relief` layer applies the color scale on the GPU. That gives 2-minute frames, QC'd data, palettes you can switch with no re-download, and no dependence on IEM. It's only worth doing if IEM proves unreliable or you want the better data.

---

## 3. Architecture

```
Phone (installed PWA)
  ├─ Service worker
  │    shell: precached (cache-first)
  │    /api: network with timeout, IndexedDB mirror
  │    tiles: bounded cache
  ├─ IndexedDB: last WeatherBundle per location, radar frame list
  └─ Solid app: 4-pane scroll-snap pager + location sheet
          │  HTTPS, behind your reverse proxy and forward auth
          ▼
weather container (Node 24 + Hono)
  ├─ /api/v1/*: auth from trusted headers; returns 401 JSON, never 302
  ├─ /tiles/*: radar/satellite proxy with disk LRU cache; immutable URLs; 204 for empty tiles
  ├─ static: hashed assets immutable; index.html and sw.js no-cache
  ├─ Refresher: schedules upstream polls for the hot set; single-flight; jittered
  ├─ Upstream clients: NWS, Open-Meteo, IEM, NOAA WMS, SPC/WPC/NHC; per-host rate limiter and back-off
  ├─ Normalizer: everything becomes WeatherBundle in US units, precomputed for display
  └─ SQLite (/data/app.db): users, locations, upstream_cache, push_subs, rum
     Disk (/data/tiles): LRU tile cache, size-capped (default 2 GB)
```

### Key design points

**One request per screen.** `GET /api/v1/bundle/:locationId` returns everything the four views need:
- current conditions
- alerts
- 48 hours of hourly data
- 7-day forecast, extended to 10 days from Open-Meteo
- the next 2 hours of precipitation (15-minute steps)
- sun and moon times
- the SPC risk for the point
- freshness and source metadata for each section

The response is about 15–25 KB gzip and has a strong ETag, so a refresh that changed nothing costs a 304.

**The server does the display math.** Unit conversion, icon mapping, day and night grouping, chart min/max and precip-chance arrays are all precomputed. The client only binds values to the DOM.

**Source fallback is per section.** If NWS hourly fails, that section comes from Open-Meteo and is labeled with its source. One bad upstream never blanks the screen.

**The server serves stale data while it revalidates.** User requests never wait on upstream when the cache holds anything. They get the cached data immediately, plus an `X-Data-Age` header. The refresher keeps the hot set warm, so in practice responses are always cache hits (p95 under 20 ms).

**Duplicate upstream calls are merged.** Concurrent requests for the same upstream key share a single in-flight promise.

**The cache survives restarts.** Upstream responses are persisted in SQLite, so a container restart doesn't stampede upstream sources.

**Auth plugs into your shared login layer.** There is no login UI in this app.
- **Default, trusted-header mode.**
  - The reverse proxy's forward-auth sets `Remote-User` (Authelia and Tinyauth), `X-authentik-username` (authentik) or `X-Forwarded-User` (oauth2-proxy). The header name is configurable.
  - The server trusts the header **only from configured proxy IPs or CIDRs** and auto-creates a user row the first time it sees one.
  - An optional groups header maps to an `admin` role, which gets the RUM and perf dashboard and cache stats.
- **API sessions must return 401, not 302.** A 302 to the auth domain inside a background `fetch` fails CORS and looks exactly like being offline.
  - Configure the proxy so `/api/*` returns 401: oauth2-proxy `--api-route`, Tinyauth's non-browser response, or an Authelia rule with the header-authorization method.
  - The client also uses `redirect: 'manual'` and treats `opaqueredirect` or 401 as "session expired", which is distinct from "offline".
- **When the session expires**, cached weather stays on screen with a "Sign in again" chip. Tapping it does a top-level navigation to `/?reauth=1`, which the service worker always passes to the network so the proxy can run its login flow. The service worker never touches `/auth*`, `/oauth2/*` or callback paths.
- **iOS home-screen apps have their own cookie jar**, so each family member logs in once inside the installed app.
- **Optional `oidc` mode** for later: the app acts as its own OIDC client against Pocket ID, Authelia or authentik, as a backend-for-frontend. Only worth it if forward auth proves awkward with the PWA.
- **`dev` mode** uses a fixed fake user and is refused when `NODE_ENV=production`.

---

## 4. Caching strategy (end to end)

| Tier | What | Policy |
|---|---|---|
| **Upstream → server** | NWS, Open-Meteo, SPC and other JSON | TTLs per §2.1. Respect `Expires`/`Cache-Control` when longer. Persisted in SQLite. Stale-while-revalidate with a hard max-stale of 6 h for forecasts and **5 min for alerts**; past that, alerts are shown as "unavailable", never as stale-but-silent. |
| | Radar and satellite tiles | Timestamped URLs are immutable: keep them until the frame ages out of the window (3 h), then evict. Disk LRU with a cap. Empty or transparent tiles are stored as a 0-byte marker and served as `204`. |
| | Frame lists (`/tiles/radar/frames.json`) | Rebuilt every 60 s from the IEM 5-minute clock (checking the newest frame exists), or from WMS GetCapabilities. |
| **Server → phone** | `/api/v1/bundle` | `Cache-Control: no-cache` plus ETag, so it revalidates cheaply. |
| | Hashed JS/CSS | `max-age=31536000, immutable` |
| | Tiles | `max-age=31536000, immutable` (timestamp in the URL) |
| | `index.html`, `sw.js`, `manifest.webmanifest` | `no-cache` |
| **Phone** | Service worker precache | App shell and the first-screen chunk, plus the MapLibre chunk once it's prefetched |
| | IndexedDB | Last bundle per location, written after each successful fetch |
| | Service-worker tile cache | Radar/satellite: about 300 entries, swept by age. Basemap: about 2,000 entries, LRU. |

### Open and refresh flow on the phone
1. Launch. The service worker serves the shell from cache. JS reads the last location's bundle from IndexedDB and paints real data, marked `wx:first-data-paint`. If there's no cache yet, fixed-size placeholders are shown.
2. In the same tick, the app fetches the bundle with `If-None-Match`.
3. On 200, new data is applied in one batched update via `requestAnimationFrame`. **If a gesture or scroll is in progress, the update is held until scrolling ends**, so content never changes under the user's finger.
4. While the app is visible, it refreshes on `visibilitychange`, on a 5-minute timer, and checks alerts every 60 s with a tiny `/api/v1/alerts/:id` ETag request.
5. At first idle after the initial paint (`requestIdleCallback`), it prefetches:
   - the MapLibre chunk;
   - `frames.json`;
   - the radar tiles for the default radar viewport (12 frames). On a metered connection or with `saveData` set, only the frame list is fetched.

   This makes Radar instant on the first tap.

### No layout shift, by construction
- Every data-bound element has a fixed reserved size: fixed-height rows, `font-variant-numeric: tabular-nums`, single-line text clamps, and an aspect-ratio box for the chart.
- **The alert strip is always present** at a fixed height. With no alerts it shows "No active alerts" with the SPC risk. With alerts it shows the highest-severity alert plus "+N more". New alerts change the content, never the layout. Tapping the strip opens a sheet with full alert text.
- Placeholders match the final layout exactly. A Playwright test compares element bounding boxes between the cached and fresh renders.

---

## 5. Layout and interaction

**Four panes in a horizontal scroll-snap pager**, with a bottom tab bar that mirrors it. Every view is one tap or one swipe away.

**1. Now.** This pane shows, from top to bottom:
- **The alert strip**, color-coded by severity.
- A **large current temperature** and the condition, with feels-like and today's high and low.
- **The next 2 hours of precipitation** as a 15-minute bar strip, with text such as "Rain starting in ~25 min".
- **A compact 12-hour mini-strip**; tapping it jumps to Hourly.
- **A details grid**: wind and gust, humidity and dew point, UV, AQI, sunrise and sunset, pressure, visibility.
- **The NWS "Today/Tonight" text**, collapsed to 3 lines.

**2. Hourly.**
- A canvas chart of the **temperature line plus precip-chance bars** for 48 hours. It scrolls horizontally with native momentum.
- Touch-scrub shows the values for that hour.
- Below the chart, a fixed-row list gives each hour's details.

**3. Daily.** Seven NWS day/night pairs, each row showing:
- icon, high and low, a temperature range bar scaled across the week, and precip chance;
- the NWS text when the row is expanded.

Days 8–10 come from Open-Meteo, labeled with a small source mark.

**4. Radar.**
- A full-bleed map with play/pause, a time scrubber and a layer button. Layers: radar, satellite IR/visible, alerts, SPC outlook, ERO, tropical, storm reports and lightning.
- **Pager swipe is disabled while the map is in front**, so map gestures win. Leave Radar via the tab bar or an edge swipe.

**Header.**
- Shows the location name, freshness ("Updated 2 min ago", turning amber when stale) and a ⌖ marker when the current location is selected.
- **Tap the location name** to open a bottom sheet with "Current location", your saved locations (drag to reorder) and a search field. Selecting a location is instant if it's cached.

**Other UI behavior.**
- **Theme:** follows the system light/dark setting; the map uses a matching basemap.
- **Haptics:** none, since the Vibration API is unsupported on iOS.
- **No splash animation.** The cached screen *is* the splash.

**Current location.**
- GPS is never on the critical path. On open we show the last-used location immediately.
- If "Current location" is selected, we request `getCurrentPosition` with `maximumAge` of 10 min. Coordinates are rounded to about 1 km before being sent to the server.
- If the new position falls in a different NWS grid cell, the bundle swaps in without shifting the layout.

---

## 6. Performance budgets and how we test them

Budgets live in `perf/budgets.json` and are enforced in CI. "Modern profile" means a Playwright Pixel 10 / iPhone 17 descriptor with no CPU throttling and a 5G-like network. "Floor profile" means a Moto G Power-class device: 4× CPU throttle, slow 4G. The floor profile catches regressions; it is not the design target.

### Budgets

| Metric | Modern profile | Floor profile | How it's measured |
|---|---|---|---|
| **Time to first render from cache** (navigation start → `wx:first-data-paint`, warm service worker plus IndexedDB) | **≤ 200 ms** | ≤ 600 ms | Performance marks in the Playwright harness (median of 10 runs) |
| LCP, warm cache | ≤ 300 ms | ≤ 800 ms | `web-vitals` `onLCP` |
| LCP, first-ever visit (cold) | ≤ 1.2 s | ≤ 2.5 s | Lighthouse 13 (run directly; LHCI is stuck on Lighthouse 12) |
| **CLS**, including the background refresh swap | **0.00** (any shift fails) | 0.00 | `onCLS` plus `layout-shift` entries plus a bounding-box diff test (iOS has no CLS API) |
| **Interaction latency** (tab tap, location switch, sheet open, chart scrub) | INP **≤ 50 ms**; input to next paint ≤ 1 frame for tab and swipe | INP ≤ 150 ms | `onINP` with scripted interactions; Event Timing |
| **Scroll and swipe smoothness** (scripted 5 s fling on Hourly, Daily and the pager) | **0 long animation frames**; ≥ 99% of frames on time | No frame over 50 ms | Chromium LoAF observer plus CDP trace frame events; on a real device, the DevTools/Safari frames track |
| **Radar playback** (12 frames looping while panning and zooming) | ≥ 99% of frames on time at the display rate; **0 blank or flashing frames**; no frame over 33 ms | No frame over 50 ms | CDP trace during a scripted pan with playback; frame-loaded assertions |
| Radar ready-to-play (tap Radar → all 12 frames loaded for the viewport) | ≤ 300 ms prefetched; ≤ 1.5 s with a warm server cache | ≤ 3 s | Custom mark once every frame's `isSourceLoaded` is true |
| Long tasks during the background refresh | **None ≥ 50 ms** | None ≥ 100 ms | `longtask`/LoAF observer around the swap |
| **Initial JS** (shell + Now + framework) | **≤ 40 KB gzip** | | `size-limit` in CI |
| Initial CSS | ≤ 10 KB gzip | | `size-limit` |
| Lazy chunks: Hourly/Daily, sheet, search | ≤ 15 KB gzip each | | `size-limit` |
| Map chunk (our code, excluding MapLibre) | ≤ 25 KB gzip (MapLibre itself is about 430 KB gzip including the worker, prefetched at idle and never blocking the first render) | | `size-limit` |
| Web fonts | 0 | | Lint rule |
| API bundle payload | ≤ 25 KB gzip | | Server test |
| Server `/bundle` latency (cache hit) | p95 ≤ 20 ms | | Server benchmark test |
| Server footprint | ≤ 150 MB RSS steady; about 0% CPU idle between polls | | `docker stats` in the release checklist |
| Radar GPU memory | Cap at 12 frames by default, or 8 when `navigator.deviceMemory ≤ 4` | | Device check |

### Testing layers
1. **CI on every push:** `size-limit`, unit tests, Playwright e2e, and the **perf harness** on the modern and floor profiles.
   - The harness installs the service worker, seeds the cache, reloads, records performance marks, and runs scripted flings with CDP synthesized gestures.
   - It collects LoAF, layout-shift and trace frames, then compares them to `budgets.json`. A regression past any budget fails the build.
   - Upstreams are replaced by recorded fixtures, so runs are deterministic.
2. **Lighthouse 13**, run directly on the production build for cold-load and PWA checks.
3. **Real-device acceptance** before each phase is marked done (`docs/perf/device-protocol.md`). Headless Chromium can't reproduce a 120 Hz display or a phone GPU, so the real device is the final judge:
   - **Android:** `chrome://inspect` over adb, a Performance-panel recording of each scripted gesture, and the frames track checked for drops.
   - **iPhone:** Safari Web Inspector, the Timelines tab with Rendering Frames, on the installed home-screen app.
   - Results are logged in `docs/perf/results.md` with device, OS and build.
4. **Real-user monitoring from the family's phones.**
   - `web-vitals` (about 3 KB, loaded after first paint) plus our own marks are beaconed to `/api/v1/rum` and stored in SQLite. An admin page shows p75 values.
   - A `?perf=1` debug overlay shows a live FPS meter and timings.

### Rules that keep it smooth
- Animate only `transform` and `opacity`.
- No JS scroll handlers that touch layout; use scroll-snap, `IntersectionObserver` and scroll-driven animations.
- Keep adjacent panes rendered. Don't use `content-visibility` on pager panes, since rendering a pane mid-swipe causes jank.
- Map work is paused whenever the Radar pane is hidden, and the animation timer stops.
- No third-party scripts and no analytics SDKs.

---

## 7. Radar animation design

1. **Server.** Publishes `frames.json` listing the last 12–24 frame timestamps and their tile URL templates (`/tiles/radar/iem/{ts}/{z}/{x}/{y}.png`). It pre-warms the tiles for each user's saved-location viewports as new frames land, so most radar tile requests are cache hits on our server.
2. **Client: one MapLibre `raster` source and layer per frame**, all added up front at `raster-opacity: 0`.
   - Hidden-by-visibility layers don't load tiles, but opacity-0 layers do, so this preloads every frame.
   - `raster-fade-duration: 0`, with the opacity transition at 0 ms for a crisp step.
   - Source `maxzoom: 9`, so deeper zooms overzoom with no new requests.
3. **Playback starts only when every frame's source is loaded.** A progress ring shows during loading.
   - Frames advance on a `requestAnimationFrame`-driven clock: about 400 ms per frame, with the last frame held for 1.5 s.
   - Each step is two `setPaintProperty` calls (previous frame to 0, next frame to the target opacity).
4. **Pan and zoom during playback.** Already-loaded frames keep animating. Newly visible tiles load for all frames, and the clock skips frames whose tiles aren't ready yet, so you never see a blank frame.
5. **A new frame arrives** (every 5 min): add a new source and layer at opacity 0, wait for it to load, then remove the oldest. `setTiles` is never called on a visible layer.
6. **Spike first.** Phase 5 starts with a throwaway benchmark of 12 versus 24 opacity layers on a real phone. If drawing 12 transparent layers costs too much per frame, the fallback is to keep only the current frame ±1 at opacity above 0 and set the others `visibility: none` *after* their tiles are cached.

---

## 8. Phased build order

Each phase ends with its tests green, the perf harness inside budget, and (for UI phases) a real-device check.

| Phase | Deliverable |
|---|---|
| **0. Foundations** | pnpm workspace (`apps/web`, `apps/server`, `packages/shared`, `perf/`), TypeScript config, lint and format, Vitest, Playwright, **perf harness and `budgets.json` from day one**, `size-limit`, Dockerfile plus example compose file, CI workflow. |
| **1. Backend data core** | Hono server; upstream clients for NWS and Open-Meteo (User-Agent, rate limiter, back-off, single-flight); SQLite upstream cache; normalizer into `WeatherBundle`; `/api/v1/bundle` for an ad-hoc lat/lon; recorded fixtures; Docker image runs. |
| **2. Shell and Now view** | Service worker (precache, API strategy, auth-path bypass); IndexedDB mirror; scroll-snap pager plus tab bar; Now view with the alert strip and 2-hour precipitation; background refresh with the deferred swap; manifest and icons; installable. **Budget gate:** first render from cache, CLS 0, initial JS ≤ 40 KB. |
| **3. Hourly and Daily** | Canvas chart with scrub; hourly list; daily list with range bars and expandable text; extended days. **Budget gate:** fling smoothness, INP. |
| **4. Users and locations** | Trusted-header auth (configurable header, trusted-proxy check, auto-provisioning); 401 handling and re-auth flow; saved locations CRUD and reorder; location sheet; search (Open-Meteo geocoding); current location plus reverse geocoding; refresher hot set. Tested behind your actual proxy. |
| **5. Radar** | Animation spike → MapLibre lazy chunk with idle prefetch; basemap (OpenFreeMap); IEM tile proxy with disk LRU and 204s; `frames.json`; animation controller; NWS opengeo fallback; scrubber and controls. **Budget gate:** radar playback while panning, ready-to-play. |
| **6. More map layers** | Satellite (nowCOAST IR/visible animated, GIBS GeoColor); alert polygons (with zone-geometry cache); SPC outlooks, watches and MDs; WPC ERO; NHC cone; storm reports; lightning; layer picker that remembers your choices. |
| **7. Hardening** | Per-section fallbacks with source labels; offline mode; stale indicators; RUM endpoint and admin page; accessibility pass (VoiceOver/TalkBack, Dynamic Type, reduced motion); attribution screen; full real-device perf pass. |
| **8. Push alerts (optional)** | Server diffs alerts per zone union every 60 s (following `references` for updates and cancellations); VAPID Web Push; per-user and per-location settings (which severities, quiet hours); iOS needs the home-screen install plus a tap to subscribe. |
| **9. Optional extras** | Self-rendered MRMS radar (`color-relief`, 2-minute frames); self-hosted Protomaps basemap; AirNow AQI; NWS Area Forecast Discussion viewer; tablet layout. |

---

## 9. Risks and mitigations

| Risk | Mitigation |
|---|---|
| IEM is a single academic service with no SLA | Fall back automatically to NWS opengeo / nowCOAST. Phase 9 MRMS rendering removes the dependency. |
| NWS outages or stale data | Per-section Open-Meteo fallback; compare `updateTime`. |
| iOS storage eviction or cookie quirks in home-screen apps | `storage.persist()`, a rebuildable cache, one-time login per device, and a real-device test of the reauth flow. |
| Forward-auth 302 breaks background fetches | Proxy returns 401 for `/api`, plus a client-side `redirect: 'manual'` backstop (§3). |
| MapLibre bundle size | Lazy-loaded and prefetched at idle; never on the first-render path. |
| Opacity-0 layer draw cost | Phase 5 spike, with a documented fallback (§7.6). |
| Upstream terms change (as RainViewer's did) | Sources sit behind adapter interfaces; source choices are recorded in this file with verification dates. |

---

## 10. Questions before approval

1. **Auth layer.** Which login layer and reverse proxy do you run, or plan to (Authelia, authentik, Tinyauth + Pocket ID, oauth2-proxy; Caddy, Traefik, NGINX Proxy Manager)? Can its `/api` rule return 401 instead of redirecting?
2. **Home server.** What CPU architecture (x86-64 or ARM), how much free disk for the tile cache, and roughly what **upload** speed? Upload speed decides whether self-hosting the basemap is worth it, since away from home your phone pulls everything through that connection.
3. **Phones.** Which phones do you and your family use (iPhone or Android, which models)? I'll tune for and test on yours, and iOS affects how push notifications are installed.
4. **Coverage.** US only, or do you need forecasts for places abroad too (travel, family overseas)? This decides how polished the Open-Meteo path needs to be.
5. **Hostname and contact.** NWS requires a contact in the User-Agent. What domain or subdomain and email should it use? It's only sent to NOAA and MET servers.
6. **Framework.** Are you happy with SolidJS, or do you prefer Svelte 5 (about 6 KB more runtime, very similar performance)?
7. **Extras.** Do you want official US AQI from AirNow (free key, signup required), or is the coarser Open-Meteo AQI fine? Is 10 days the right forecast length, or should it go to 14–16 from Open-Meteo?
8. **Push.** Keep it as phase 8, or move it earlier? And which alerts should notify: warnings only, or watches and advisories too?
