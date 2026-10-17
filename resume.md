# Resume notes: Adam's Weather PWA

Last updated: 2026-10-17. Branch: `ccr-efd6b4af-qqmcnu` (develop and push only here). Repo: `abcasada/adams-weather-app`.

## Status in one paragraph

The planning phase is finished and **waiting on the user's approval and answers to 8 questions**. No application code exists. The repo contains only `CLAUDE.md` (project rules), `docs/PLAN.md` (the full plan) and this file. Do not start Phase 0 until the user approves the plan; the user asked for a written plan first.

## What the user asked for (original brief, condensed)

A self-hosted weather PWA in a Docker container on a home server, installed on phones, for the user plus a few family and friends, each with their own saved locations.

- **Smoothness is the top priority.** Instant open from cache, background refresh, no layout shift, full-refresh-rate scrolling, swipes and transitions. Optimize for current phones; stay functional on older ones.
- **Radar and maps must not lag.** GPU-accelerated maps, preloaded radar frames, fluid pan and zoom during playback.
- **Measure it.** Concrete budgets (first render from cache, interaction latency, frame rate during scroll and radar, bundle size), tested on a real modern phone profile.
- **Free data sources** with verified terms and limits; fetch and cache server-side so users don't multiply API calls.
- **Layout:** the first screen is today's overview plus active watches and warnings. Hourly temperature and rain chance, the multi-day forecast, and radar are each one tap or swipe away. US units, saved locations plus current location.
- **Maps:** animated radar plus other free layers worth including (satellite, severe outlooks).
- **Auth:** plug into one shared login layer across the user's self-hosted apps; do not build login.
- **Later (optional):** push notifications for warnings at saved locations.
- **Deliverables requested before any code:** a written plan (stack, sources and limits, caching, budgets and testing, phased build order), a CLAUDE.md, and questions for the user.

A later request asked for this `resume.md`, then a commit of everything.

## What has been done

1. Ran three parallel research agents on 2026-10-14 (weather data, radar/map tiles, front-end stack and auth). Their findings are folded into `docs/PLAN.md` §1 to §2. Live requests were made where possible; unverified items are listed below.
2. Wrote `docs/PLAN.md` (stack, sources, architecture, caching, layout, budgets and test approach, radar animation design, 10 phases, risks, 8 open questions).
3. Wrote `CLAUDE.md` (non-negotiable rules for smoothness, data and upstreams, auth, radar; conventions; planned layout and commands).
4. Committed and pushed both (`9057808`). This file was added afterwards.

## Decisions made (all pending user approval)

- **Front end:** SolidJS 1.9 + Vite 8 static SPA, hand-rolled service worker, native CSS scroll-snap pager, custom canvas charts, IndexedDB (`idb-keyval`) mirror of the last bundle per location. Svelte 5 is the alternative (about 6 KB more runtime).
- **Maps:** MapLibre GL JS 6.x (WebGL2, ESM-only), lazy-loaded and prefetched at idle. It's about 285 KB gzip plus a 143 KB worker, so it never touches the first-render path.
- **Backend:** Node 24 LTS (move to 26 after 2026-10-28) + Hono 4 + SQLite via `better-sqlite3`; valibot for upstream validation; `web-push` for later.
- **Auth:** trusted-header forward-auth (`AUTH_USER_HEADER`, default `Remote-User`; trusted only from `TRUSTED_PROXY_CIDRS`); `/api/*` returns 401 JSON, never 302; optional OIDC (backend-for-frontend) mode later; `dev` mode refuses to run in production.
- **Data:** NWS primary for the US, Open-Meteo as fallback, for global coverage and for 15-minute rain nowcasts. SPC, WPC and NHC for severe layers. Open-Meteo geocoding for search. NWS `/points` or Census for naming the current location; Nominatim reverse-geocode only as a fallback.
- **Radar:** IEM `ridge::USCOMP-N0Q-YYYYMMDDHHMI` tiles via `/c/tile.py` (immutable timestamped URLs) as primary; NWS opengeo `conus_bref_qcd` WMS-T, then nowCOAST, as fallbacks. One MapLibre raster layer per frame, preloaded at `raster-opacity: 0`.
- **Satellite:** nowCOAST GOES IR and visible (5-minute steps) plus NASA GIBS GeoColor (max zoom 7).
- **Basemap:** OpenFreeMap (`dark`/`positron`) as the default; self-hosted Protomaps PMTiles is the alternative, pending Q2.
- **Not using:** RainViewer (capped at zoom 7, 100 req/min/IP since 2026-01-01), tile.openstreetmap.org (forbids prefetch and caching), SSEC RealEarth (watermarks), MapTiler/Stadia free tiers, OpenWeatherMap, Tomorrow.io.
- **Budgets (modern profile):** first render from cache at most 200 ms; CLS 0.00; INP at most 50 ms; 0 long animation frames in scripted flings; radar at least 99% on-time frames with no blank frames; radar ready in at most 300 ms when prefetched; initial JS at most 40 KB gzip. Full table in `docs/PLAN.md` §6.
- **Testing:** Playwright 1.64 harness (device descriptors, CDP CPU throttle, LoAF, trace frames) on a modern profile and a Moto G Power-class floor profile; Lighthouse 13 run directly (LHCI 0.15.1 is stuck on Lighthouse 12); `size-limit` 14; `web-vitals` 6; real-device checks (Android over adb with `chrome://inspect`, iPhone with Safari Web Inspector) before each phase is marked done; RUM beacons from the family's phones.

## Open questions (unanswered; ask the user again)

1. Which login layer and reverse proxy do they run (Authelia, authentik, Tinyauth + Pocket ID, oauth2-proxy; Caddy, Traefik, NPM), and can `/api` return 401 instead of redirecting?
2. Home server CPU architecture, free disk for the tile cache, and upload speed (decides self-hosting the basemap).
3. Which phones do they and their family use (iPhone or Android, models)?
4. US only, or forecasts abroad too?
5. Domain and contact email for the NWS User-Agent (sent only to NOAA and MET servers).
6. SolidJS or Svelte 5?
7. AirNow official AQI (free key) or Open-Meteo AQI? Forecast length: 10 days, or 14 to 16?
8. Push alerts: keep as phase 8 or earlier? Warnings only, or watches and advisories too?

If the user answers in the next session, update `docs/PLAN.md` (§1 stack, §2 sources, §10 questions) and `CLAUDE.md` to match before starting Phase 0.

## Research caveats (unverified or fast-moving)

These were flagged by the research agents; re-check before relying on them.

- **NWS:** exact rate limit and throttle status codes (403 vs 429) are not published. Conditional requests did not return 304 in tests. Since about July 2026 the last 1 to 2 hourly periods can have null values. Stricter TLS ciphers take effect about 2027-03-02 (SCN 26-80). A proposed VTEC-to-CAP change (PNS26-62) has comments open to 2026-11-20.
- **NWS alerts:** only about 6% of active alerts carry a polygon (28 of 444 when checked), so zone geometries have to be resolved and cached. The `limit=` parameter is rejected on `/alerts/active`.
- **Open-Meteo:** the docs are inconsistent on whether NBM's native precipitation probability is used; live results showed the same value across models. Per-IP limits are 600/min, 5,000/hour and 10,000/day.
- **IEM:** no published rate limit and no SLA. MRMS layers on IEM need even-minute timestamps; N0Q `ridge::` layers need 5-minute multiples. `ridge::USCOMP-N0Q-0` is not equivalent to the mosaic layer.
- **NASA GIBS:** about 50 minutes latency; terms page was unreachable, so terms are unverified. No near-real-time GLM lightning layer exists there.
- **Lightning:** redistribution terms for NOAA's NLDN/GLD360 density products are unverified (fine for private use, off by default).
- **Protomaps:** the size of a US z15 extract was not measured (likely tens of GB; `--maxzoom=12` or `13` shrinks it a lot).
- **iOS:** Safari 27 is out, but its exact release date and any iOS 27 changes to home-screen apps were not verified. Standalone geolocation re-prompt behavior is unverified. The 7-day storage eviction exemption for home-screen apps is not confirmed in practice, so the plan assumes the cache is rebuildable and calls `storage.persist()`. Test the auth redirect and iOS in-app browser sheet on a real device.
- **Not verified at all:** AirNow's 500/hour limit, Pirate Weather per-minute limits and license, OpenWeatherMap card requirement, Tomorrow.io current limits, Photon typeahead policy, NWWS-OI eligibility, whether the legacy `alerts.weather.gov/cap` feeds are retired.
- **Versions seen on 2026-10-14:** Svelte 5.57.2, SvelteKit 3.0.1, Solid 1.9.17 (2.0 is RC), SolidStart 2.0.6, Preact 11.0.1, Vite 8.3.4, MapLibre GL 6.14.0, Node 24.22.0 LTS, Hono 4.13.13, better-sqlite3 13.0.3, web-push 3.6.7 (stale since 2024 but works), Playwright 1.64.1, Lighthouse 13.5.0, web-vitals 6.2.3, size-limit 14.2.0, Authelia 4.39.28, Tinyauth v5.0.7, oauth2-proxy v7.15.5, Pocket ID v2.5.0.

## Next steps when resuming

1. Read `docs/PLAN.md` and `CLAUDE.md`. They are the source of truth; this file only summarizes.
2. Show or recap the plan and ask the open questions above. Do not begin coding without the user's approval.
3. After approval and answers, update the plan and `CLAUDE.md` for any changed choices.
4. Start **Phase 0 (Foundations)**: pnpm workspace (`apps/web`, `apps/server`, `packages/shared`, `perf/`), strict TypeScript, lint and format, Vitest, Playwright, the perf harness plus `perf/budgets.json` from day one, `size-limit`, Dockerfile and `compose.example.yml`, CI workflow. Then Phase 1 (backend data core) and so on, per `docs/PLAN.md` §8.
5. Update the "Layout" and "Commands" sections of `CLAUDE.md` as they become real (they are currently marked planned).

## Housekeeping notes

- Develop and push only on `ccr-efd6b4af-qqmcnu`. Do not open a pull request unless the user asks. If a pull request for this branch was merged in the meantime, restart the branch from the latest default branch (same name) before adding work.
- The research agents' scratch builds and probe outputs were in the session scratchpad (`/tmp/claude-0/-home-user-adams-weather-app/1ecbc74b-9882-5b48-81d7-631d7261f0ec/scratchpad/`), which does not persist to a new session. Nothing there is needed; all findings are recorded above and in `docs/PLAN.md`.
- Commit trailers for this repo: end commit messages with the attribution lines the session provides.
