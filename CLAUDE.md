# Hall dashboard — context for Claude Code

A wall-mounted dashboard for the entrance hall, shown full-screen on an iPad.
It shows: clock/date, weather, next departures per stop with disruption
warnings, a Paris webcam, a live traffic map, and buttons that run Apple Shortcuts
to control the home (Apple Home).

## Target device — read this before changing anything
- **iPad Pro 9.7" (2016), stuck on iPadOS 16 → Safari 16.** Do not use web features
  newer than Safari 16 (no CSS `:has()` reliance, no container queries, no
  `@layer`-only styling, no View Transitions, no `Array.prototype.findLast`-era
  surprises without checking caniuse). Keep JS plain ES2017-ish.
- Old A9X chip: keep it light. No frameworks, no build step, no heavy animations,
  and no polling faster than needed.
- Main orientation: **landscape** (1024×768 CSS px). Portrait must still work.
- Runs as a **home-screen web app** (Add to Home Screen) under Guided Access.
  Home-screen web apps on iPadOS have their **own localStorage**, separate from Safari.
- Dark theme on purpose (entrance hall, low light). Night dimming overlay 23:00–06:00.

## Project shape
- **Single file: `index.html`** (HTML + CSS + inline JS). Keep it that way unless
  there's a strong reason; it's hosted as-is on GitHub Pages from `main` / root.
- No secrets in the repo. The PRIM API key and per-user settings (departure screen
  URLs, webcam link, home address) are entered in the in-page ⚙︎ Réglages panel
  and saved in `localStorage` (key `hall-dashboard-config-v1`). `DEFAULTS` in the
  script define the shape; saved config is deep-merged over it.
- UI language: **French**.

## Data sources
| Feature | Source | Notes |
|---|---|---|
| Weather (now, 15-min rain, hourly, 5 days) | Open-Meteo `api.open-meteo.com/v1/forecast` | No key, CORS OK. Location from settings (`cfg.location`). WMO weather codes mapped in `WX`. |
| Next departures | PRIM "Prochains passages (requête unitaire)" (SIRI Lite): `…/marketplace/stop-monitoring?MonitoringRef=STIF:StopArea:SP:<zone>:`, header `apikey` | CORS OK (checked 2026-11-15). One request per stop returns all its lines; filtered by IDFM line id (`C0xxxx`). Default quota for a new key ≈ 1000/day (free raise via PRIM "Ma consommation") → refresh interval is computed from the `quota` setting and the waking hours; countdowns tick locally between fetches. A stop's `ref` may also be a departs.leon.gp URL (ids are parsed out). Data shape not yet verified against a real key — test with ⚙︎ "Tester PRIM". `?demo` in the URL shows fake data for layout work. |
| Line status | PRIM (Île-de-France Mobilités) "Info trafic – requête globale": `https://prim.iledefrance-mobilites.fr/marketplace/disruptions_bulk/disruptions/v2`, header `apikey` | Free key from prim.iledefrance-mobilites.fr. Response: `lines[]` (id, shortName, mode, impactedObjects[].disruptionIds) + `disruptions[]` (id, severity BLOCKING/DISTURBED/INFORMATION, cause WORKS/DISTURBANCE/INFORMATION, title, message (HTML), applicationPeriods with `YYYYMMDDTHHMMSS`). Lines matched by IDFM id (`line:IDFM:C01743` ↔ `C01743`). CORS OK (checked 2026-11-15: `Access-Control-Allow-Origin: *`, `apikey` header allowed) — no relay needed. URL is overridable in ⚙︎ if a relay is ever wanted. Refresh every 5 min — mind daily quota. |
| Webcam | YouTube embed (`youtube-nocookie.com/embed/<id>` or `live_stream?channel=UC…`) | User pastes a YouTube link; muted autoplay; reloaded every 30 min. |
| Traffic | Waze live map iframe `embed.waze.com/iframe?zoom=&lat=&lon=&ct=livemap` centred on `cfg.traffic` (falls back to `cfg.location`) | "Itinéraire" opens Apple Maps directions (home address from settings). No travel-time number (would need a keyed routing API). |
| Home buttons | `shortcuts://x-callback-url/run-shortcut?name=…&x-success=<page url>` | Runs an Apple Shortcut by exact name, then returns to the dashboard. |

**Privacy: the repo is public.** No personal places in code or in this file — stops, followed lines,
weather point and map centre are entered in ⚙︎ (or imported from the gitignored
`private/hall-config.json`) and stored in localStorage. Real values: see `CLAUDE.local.md` (gitignored).
`DEFAULTS` stay neutral (Paris centre, no stops/lines); `?demo` uses made-up stops.
IDs/colours come from IDFM open data (`data.iledefrance-mobilites.fr` datasets `arrets-lignes`,
`arrets`, `referentiel-des-lignes`; no key, CORS OK).

Layout (landscape): transit = left 2/3 (line-status chips strip + 2×3 stop cards with
Citymapper-style warning banners); right 1/3 = weather + webcam/traffic tabs.

## Working on it
- Preview locally: `python3 -m http.server` in the repo, open `http://localhost:8000`
  (not `file://` — fetch/iframes behave differently). Use a 1024×768 viewport.
- Settings live in localStorage, so to test status/departures locally, open ⚙︎ and
  enter a PRIM key / URLs, or stub `fetch` responses.
- After changes: check landscape and portrait, check nothing overflows, check the
  JS still parses (`node --check` on the extracted script is a quick sanity test).
- Deploy = commit + push to `main`; GitHub Pages updates in ~1 min. The iPad picks
  it up on next reload (the page also re-renders feeds periodically).

## Ideas / backlog
- Night dimming from sunset→sunrise (Open-Meteo `daily.sunrise/sunset`) instead of fixed hours.
- Tap-to-wake: at night keep it very dark; a tap restores full brightness for ~1 min.
- "Je pars" view: next departures still catchable given walking time to each stop, weather, drive time.
- "🔌 Charge" button running a Shortcut that turns on the charger smart plug for an hour.
