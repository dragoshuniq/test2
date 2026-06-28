# Sygic Multi-Stop — THE Working Solution (verified on device)

> Verified on **Sygic GPS Truck & Caravan** (consumer app), Android. **5 and 10 waypoints confirmed routing** — Sygic computes the truck road route through the waypoints. This is the locked mechanism. Test harness: https://dragoshuniq.github.io/test2/ (repo: dragoshuniq/test2).

## The mechanism (do not deviate)

**`route_download` + a hosted itinerary file with `precomputed:false`.** Sygic downloads the file and **computes** the road route through the waypoints with the truck profile.

Deeplink:
```
com.sygic.aura://route_download|<ROUTE_FILE_URL>|sif
```

Route file (served at `ROUTE_FILE_URL`, must be publicly reachable by the phone):
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
- Coords are **degrees × 100000 as integers**, fields `lon` / `lat`.
- One `routePart` per consecutive segment. Types: first `start`, middle `via`, last `finish`.
- `precomputed:false` is what makes Sygic **compute** the route (vs. trace an exact line).
- Confirmed working at **5 and 10** waypoints. Keep ≤ ~10 (cap ladder 10/15/20/25/30 is in the harness if you want to push it).

## Delivery — how the deeplink must reach Sygic

The custom scheme is NOT tappable as plain text. It must be a real `<a href>` link.

- **Web / browser (logifie.com/r/<token>):** wrap in an Android `intent://` URL (Chrome's `Intent.parseUri` builds the intent). Pipes → `%7C`, inner URL → `encodeURIComponent`:
  ```
  intent://route_download%7C<encodeURIComponent(url)>%7Csif#Intent;scheme=com.sygic.aura;S.browser_fallback_url=<encoded-playstore>;end
  ```
  iOS: use the raw `com.sygic.aura://route_download|<url>|sif` href.
- **Driver RN app:** open the raw scheme via `Linking.openURL('com.sygic.aura://route_download%7C<encodeURIComponent(url)>%7Csif')` (pipes `%7C`, url encoded once). `intent://` does NOT work via RN `Linking` (it does `Uri.parse`, not `Intent.parseUri`).

## Reachability (critical)

`route_download` makes the **phone fetch the file**. The `ROUTE_FILE_URL` host must be publicly reachable from the phone:
- Prod: `https://tms.logifie.com/api/r/<token>/route.json`
- Dev: a tunnel (e.g. `https://<id>.devtunnels.ms/...`) or LAN IP — NOT `localhost`, NOT the marketing site `logifie.com`.

## Implementation map

| Surface | File | Status |
|---|---|---|
| Route file builder | `logifie-tms/services/routing/sygic-itinerary.ts` (`buildSygicItineraryFile`) | produces the format above |
| Route file endpoint | `logifie-tms/app/api/r/[token]/route.json/route.ts` (full + `?leg=N`) | serves it |
| Web `/r/<token>` link | `logifie-tms/features/route-planner/driver-route-view.tsx` + `sygic-deep-link.ts` (`toAndroidIntentUrl`) | `route_download|sif` + `intent://` |
| Driver app link | `logifie-driver/src/lib/navigation-links.ts` (`buildSygicItineraryLink`, `routeFileUrl`) | `route_download%7C…%7Csif`; route base derived from TMS origin |
| Driver opener | `logifie-driver/src/components/order-detail/open-sygic.ts` | raw scheme via `Linking.openURL` |

## What does NOT work on the consumer Truck & Caravan app (tested)
- Multi-`coordinate` in one deeplink (`coordinate|lon1|lat1|...|drive`).
- `routeimport` inline JSON.
- `route_download` with a sparse polygon (needs ~180m dense points → use the itinerary format above instead).
- `go.sygic.com` universal links (single or `via[]`) — consumer GPS app only, not Truck.
- Single `coordinate|lon|lat|drive` DOES work (use for single-destination, no prestops).
