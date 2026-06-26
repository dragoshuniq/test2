# Sygic Deeplink Dispatch — Verified Knowledge

> On-device verified against **Sygic GPS Truck & Caravan (consumer app) v26.0.6**, Android, links opened in Chrome via `intent://`. Date: 2026-06.
> Goal: dispatcher picks a route/corridor in the TMS, driver's Sygic navigates it with the **truck profile**, and the route is cheap-toll-aware. Toll *cost* is estimated with other maps (HERE/HOGS, Google/SWRgo) which are NOT truck-optimized — Sygic does the actual truck navigation.

---

## TL;DR — the one mechanism that works for "few points, Sygic computes the roads"

**`route_download` + a hosted Sygic *itinerary* file with `precomputed:false`.**

- You send ~5 waypoints.
- Sygic **computes the truck road route** through them (verified: points placed on a dead-straight off-road line still bend onto the real motorways → Sygic is genuinely routing, not connecting dots).
- Tiny file (~0.8 KB for 5 points).
- This is the production target.

---

## What WORKS (verified on device)

| Mechanism | Result |
|---|---|
| `com.sygic.aura://coordinate\|lon\|lat\|drive` | ✅ single destination |
| `route_download \| <url> \| sif` with **itinerary** file (`precomputed:false`) | ✅ **Sygic computes roads through the waypoints** ← USE THIS |
| `route_download \| <url> \| json` with **dense polygon** file (~180m point spacing) | ✅ traces an EXACT precomputed line (rigid) |

## What FAILS (do NOT use)

| Mechanism | Failure |
|---|---|
| `coordinate\|lon1\|lat1\|...\|lonN\|latN\|drive` (multi-point) | app does nothing |
| `truckSettings\|...` and `&&&` action chaining | consumer app ignores the whole command (SDK-only feature) |
| `routeimport\|<inline-json>\|...` | "Format error / character limit" |
| `route_download` with a **sparse polygon** (few far-apart points) | "maximum points reached" + minutes-long calc (map-match fails → falls back to treating points as waypoints → hits cap) |
| `https://go.sygic.com/directions?...&via[]=...` | consumer **GPS** app only, NOT the Truck app |

---

## The two route-file formats

### A) Itinerary — Sygic COMPUTES (use for "few shaping points") ✅
`route_download | <url> | sif`

```json
{
  "name": "route",
  "version": "2.2",
  "directives": { "allowItineraryEdit": true },
  "routeParts": [
    { "properties": { "routeMappingType": "none", "precomputed": false },
      "waypointFrom": { "lon": 2101220, "lat": 5222970, "type": "start" },
      "waypointTo":   { "lon": 1692520, "lat": 5240640, "type": "via" } },
    { "properties": { "routeMappingType": "none", "precomputed": false },
      "waypointFrom": { "lon": 1692520, "lat": 5240640, "type": "via" },
      "waypointTo":   { "lon": 746530,  "lat": 5151360, "type": "finish" } }
  ]
}
```

- Coordinates are **degrees × 100000 as integers**, fields named `lon` / `lat`.
- One `routePart` per consecutive segment; types: `start` (first), `via` (middle), `finish` (last).
- `precomputed:false` is the flag that makes Sygic route through the points.

### B) Polygon — Sygic TRACES an exact line (use for guaranteed-exact toll)
`route_download | <url> | json`

```json
{ "polygon": { "lineString": { "points": [ { "x": 2101223, "y": 5222968 }, ... ] } },
  "stations": [ { "polyIdx": 0, "wayPointType": "START" },
                { "polyIdx": 7160, "wayPointType": "VIA" },
                { "polyIdx": 14320, "wayPointType": "DESTINATION" } ] }
```

- `points` is the route geometry; `x`=lon×100000, `y`=lat×100000.
- **MUST be dense (~180m gaps)** so each segment lies on a road, or it fails (see "sparse polygon" above).
- Rigid: Sygic drives the line exactly → the toll you calculated = the toll driven.
- This is what the existing TMS `buildSygicRouteFile` already produces.

**Choose A (itinerary) for the new feature** — few points, light, Sygic does the truck routing. Choose B only when you need a guaranteed-exact line.

---

## Delivery — how the link must reach the phone

The custom scheme is **NOT** tappable as plain text in Telegram or pasted into a browser address bar. It must be a real `<a href>` link, delivered via an HTTPS landing page.

**Android** — wrap in `intent://`:
```
intent://route_download%7C<FULLY-ENCODED-URL>%7Csif#Intent;scheme=com.sygic.aura;S.browser_fallback_url=<encoded-playstore-url>;end
```
- Replace the scheme separators `|` with `%7C`.
- **Fully percent-encode the inner route-file URL** (encode `:` and `/` too — `https%3A%2F%2F...`). Leaving `://` raw makes the intent silently no-op.
- `browser_fallback_url` = `https://play.google.com/store/apps/details?id=com.sygic.aura` (encoded).

**iOS** — use the raw scheme as the href (no `intent://`):
```
com.sygic.aura://route_download|<url>|sif
```

**Telegram flow:** send an HTTPS link to a landing page → driver opens it (in Chrome; Telegram's in-app browser can block intents) → page has the `<a href>` button → tap fires Sygic.

---

## Existing TMS code (already ~80% there)

- `services/routing/route-dispatch.ts` — `buildSygicFullRouteLink`, `buildSygicWaypointsLink`, `buildDispatchLinks`, dispatch token flow.
- `services/routing/sygic-export.ts` — `buildSygicRouteFile` (polygon format B, densifies to 180m).
- `features/route-planner/sygic-deep-link.ts` — `toAndroidIntentUrl` (intent:// builder).
- `app/api/r/[token]/route.json/route.ts` — serves the route file (the `route_download` target).
- `constants/routing.ts` — `ROUTE_DISPATCH_MODES` (waypoints / full_route / destination), `SYGIC_MAX_POINT_GAP_M = 180`.

### Two confirmed BUGS in the current TMS dispatch (block production)
1. `buildSygicFullRouteLink` prepends `truckSettings|...&&&` → the consumer Truck app **ignores the whole command**. Must be removed (set truck profile separately, or rely on the driver's in-app truck settings).
2. `toAndroidIntentUrl` only encodes `|`, leaving the inner `https://` raw → intent silently no-ops. Must **fully URL-encode** the route-file URL.

---

## Open items / to confirm

- **Max waypoints for the itinerary format**: 5 confirmed working. A ladder test (10/15/20/25/30) exists in the harness but the exact ceiling is unconfirmed. The polygon format hit "maximum points reached" at 87. **Design for ≤ ~15–20 shaping points and verify before relying.**
- Whether the itinerary route applies the **truck profile** automatically or uses the app's current vehicle settings (test: route through a known truck-restricted road).

---

## Working test harness

- Repo: `github.com/dragoshuniq/test2` · Pages: `https://dragoshuniq.github.io/test2/`
- Local: `~/Desktop/Work/Logifie-ecosystem/route-testing/`
- Working example file: `itinerary5.json` (the 5-point compute file). Deeplink built by the generator scripts in that folder's git history.
