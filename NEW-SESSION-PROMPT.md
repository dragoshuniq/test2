# New Session Prompt — Sygic shaping-points integration into TMS

Copy everything below the line into a fresh session (run it from the `logifie-tms` project).

---

## Context

I have **verified on-device** (Sygic Truck & Caravan consumer app v26.0.6) how to send a route to a driver's Sygic so that **Sygic computes the truck road route through a few shaping waypoints**. The full verified knowledge — formats, what works, what fails, delivery mechanism, and two bugs in the current TMS dispatch code — is documented here:

**READ FIRST:** `~/Desktop/Work/Logifie-ecosystem/route-testing/SYGIC-DEEPLINK-KNOWLEDGE.md`

The working mechanism (short version): `route_download | <hosted-url> | sif` where the URL serves a Sygic **itinerary** file with `precomputed:false` and waypoints (lon/lat × 100000). Delivered via an `intent://` link (Android) / raw scheme (iOS) on an HTTPS landing page. ~5 points confirmed; design for ≤ ~15–20.

## Goal

Add **shaping / via points ("pre-stop points")** to the TMS so a dispatcher can pin the corridor a driver should take (to favour a cheaper-toll route), and dispatch it to Sygic with the truck profile. Two integration surfaces:

1. **Route Planner** (`http://localhost:3200/route-planner`) — let the dispatcher add/drag shaping points on the map between stops, see the route + approximate toll (via the existing HERE/HOGS / Google map providers), then dispatch to Sygic.
2. **Order stops** — add an optional list of shaping/via points to an order (distinct from real stops; they are pass-through, not delivery/pickup points).

## Expected behaviour (THE CORE — most important)

The dispatcher must be able to use it **both ways, switching freely on the same screen**:

1. **Single point — the OLD way (default).** Just a destination → driver gets `com.sygic.aura://coordinate|lon|lat|drive`. **No pre-stops. This is the current working behaviour and MUST be preserved exactly — do not regress it.**
2. **Multi-point — when I want it.** The dispatcher **dynamically adds pre-stop points**: a variable number of coordinate inputs (add/remove rows AND/OR drag pins / search / click on the map). These shape the corridor (origin → pre-stop 1 → … → pre-stop N → destination). When **≥1 pre-stop exists**, dispatch automatically switches to the itinerary mechanism (`route_download | <url> | sif`, `precomputed:false`) so **Sygic computes the truck route through all the points**.

Rules for pre-stops:
- **Optional and dynamic** — add/remove/reorder at any time; 0..N of them.
- Entered as **coords** (manual lat/lng input), and ideally also via map click / address search / drag.
- **Pass-through**, not delivery/pickup stops (no status, no time window).
- **Mode is chosen automatically by count**: 0 pre-stops → single-destination (old way); ≥1 → itinerary multi-point. (An explicit override toggle is fine, but auto-by-count is the expectation.)
- The **same** pre-stop coords feed the toll/cost estimation on the other maps (HERE/HOGS, Google/SWRgo) — one source of truth.

## Requirements

1. **Data model**: add shaping/via points to the order/route-plan model. They are ordered, sit between real stops, and are NOT stops (no status, no time window). Keep immutable update patterns and the project's `type`-over-`interface` convention.
2. **New dispatch mode** (or extend existing): generate the **itinerary** file (format A in the knowledge doc, `precomputed:false`) and serve it from the existing `app/api/r/[token]/route.json` style endpoint with `type=sif`. Do NOT reuse the polygon (format B) path for this.
3. **Fix the two confirmed deeplink bugs** (see knowledge doc):
   - remove the `truckSettings|...&&&` prefix from the full-route link builder,
   - fully URL-encode the inner route-file URL in `toAndroidIntentUrl`.
4. **Cross-map compatibility**: the shaping points are plain coordinates and MUST also feed the existing toll/cost estimation (HERE/HOGS, Google/SWRgo) so the dispatcher sees approximate toll for the same corridor they send to Sygic. One source of truth for the points.
5. **Delivery**: keep the existing Telegram + landing-page flow; the landing page serves the `<a href>` (intent:// Android, raw iOS).
6. **Max-waypoint guard**: cap shaping points to a safe number (confirm the real Sygic limit first — test in the harness at `route-testing/`), and surface a clear UI warning when exceeded.

## Constraints / process

- **Do NOT break** the existing dispatch modes (`destination`, `waypoints`, `full_route`).
- Follow TMS conventions in `logifie-tms/.claude/CLAUDE.md` (thin pages → feature components, services hold DB logic, no code comments, `type` not `interface`).
- TDD where practical; add unit tests for the itinerary-file builder and the deeplink encoder.
- **Review the final changes with 2 independent agents** (one code-quality, one security) before finishing — per my standing preference.
- Confirm my git preference before committing (default: stage, don't push).

## Files to read before planning

- `~/Desktop/Work/Logifie-ecosystem/route-testing/SYGIC-DEEPLINK-KNOWLEDGE.md` (the verified spec)
- `logifie-tms/services/routing/route-dispatch.ts`
- `logifie-tms/services/routing/sygic-export.ts`
- `logifie-tms/features/route-planner/sygic-deep-link.ts`
- `logifie-tms/features/route-planner/*` (route planner UI)
- `logifie-tms/app/api/r/[token]/route.json/route.ts`
- `logifie-tms/constants/routing.ts`
- the order model + order stops feature/components

## Deliverable for this session

Start by **brainstorming the design** (don't write code yet): propose the data-model shape for shaping points, the new dispatch mode, the itinerary-file builder, the UI for adding/dragging shaping points in the route planner, and how the same points feed toll estimation. Present a plan and wait for my confirmation before implementing.
