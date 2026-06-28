# New Session Prompt — Auto-create the route token when an order is created

Copy everything below the line into a fresh session, run from the `logifie-tms` project.

---

## Goal
When an order is **created** (and refreshed when its stops/prestops change), automatically create a **route dispatch token + route plan** so the driver app and the `/r/<token>` web page can navigate the route in Sygic **without any manual "Send to driver" step**. Telegram/app dispatch stays optional (for sending the link to drivers who don't use the app).

## Why a token is required (don't try to avoid it)
Sygic is a separate app that **fetches the route file itself** (`route_download`). It cannot send the driver's auth, so the route file must be served from a **public, token-based URL** (`/api/r/<token>/route.json`). The token IS the capability. So every order that should be navigable needs a token.

## The locked, verified mechanism (do NOT change or pivot to GPX/universal links)
Sygic Truck & Caravan navigates a multi-stop route via:
- Deeplink: `com.sygic.aura://route_download|<route.json-url>|sif`
- Route file = a Sygic **itinerary** with `precomputed:false` → Sygic **computes** the road route through the waypoints with the truck profile.
- Verified on device at 5 and 10 waypoints. Full reference: `route-testing/SYGIC-WORKING-SOLUTION.md`.
- The route file is **generated on-the-fly** by the endpoint from the `BetaRoutePlan` in MongoDB; nothing is stored on disk. The plan/token has a 14-day TTL (`ROUTE_DISPATCH_TOKEN_TTL_DAYS`).

## What already exists (implemented in the prior session — do not redo)
- `services/routing/sygic-itinerary.ts` → `buildSygicItineraryFile(points)` (the itinerary format).
- `app/api/r/[token]/route.json/route.ts` → serves the itinerary (full + `?leg=N`), token-based, public.
- `services/routing/route-dispatch.ts`:
  - `getActiveRouteForOrder(orderId)` → returns the order's latest non-expired `{ token, kind }` or null. ← you will extend/replace this.
  - `dispatchWaypoints(...)` → creates a `BetaRoutePlan` (flattened stops, vehicle profile, token) **and sends to channels**. Reuse its plan-creation logic but WITHOUT sending.
  - `createRouteDispatch(planId, mode, channel, ...)` → mints a token/dispatch for an existing plan.
  - `selectDispatchMode(stops)` → `ITINERARY` if any `kind:"pre"`, else `DESTINATION`/`WAYPOINTS`.
  - `buildDispatchLinks(...)`, `buildSygicItineraryLink(url)` (uses `sif`).
- `services/routing/route-plan.ts` → `normalizeStops(stops)` (→ `RouteStopPoint[]` with `kind`), `groupStopsByLeg`, `legItineraryPoints`.
- `services/routing/vehicle-profile.ts` → `buildVehicleProfile(fleetAsset)`.
- `features/route-planner/waypoints.ts` → `flattenOrderStops(orderStops)` → `FlatPoint[]` (prestops then stop, `kind`). NOTE: this is a client feature file; for server use, replicate the small flatten in `services/routing` rather than importing from `features/`.
- Driver order endpoint `app/api/driver/v1/orders/[orderId]/route.ts` already returns `data: { order, route }` where `route = getActiveRouteForOrder(orderId)`.
- Driver app already uses `order.route?.token` to fire `route_download` natively (`buildStopSygicLink` / `openSygicLink`).
- `/r/<token>` web page (`features/route-planner/driver-route-view.tsx`) already builds the deeplinks and (fixed) uses the public host via `x-forwarded-host` → `env.appBaseUrl()`.

So the ONLY missing piece is: **mint the token automatically at order creation** (and refresh on change).

## What to build

1. **`ensureRouteForOrder(order)`** in `services/routing/route-dispatch.ts` (new function):
   - Input: a saved order (has `_id`, `stops` with `address.geo` + `preStops`, `execution.fleetAssetId`, `execution.driverEmployeeId`).
   - Flatten `order.stops` → ordered points with `kind` (each stop's `preStops` come BEFORE the stop), keep only routable coords (`isRoutableCoord` from `lib/geo-coords.ts`). If `< 2` routable points (or `< 1`), do nothing (no token).
   - Build the vehicle profile from the order's fleet asset (`buildVehicleProfile`), defaulting if none.
   - Create a `BetaRoutePlan` (planId, orderId, driverEmployeeId, vehicleSnapshot, normalized stops, `selectDispatchMode`) and mint a dispatch token via `createRouteDispatch` (or inline) — **but do NOT send to any channel** (no Telegram, no push). This is the key difference from `dispatchWaypoints`.
   - Idempotency / refresh: if an active (non-expired) plan already exists for the order AND its stops match the order's current stops/prestops, reuse it. If the stops/prestops CHANGED, regenerate (new plan/token or update in place) so the route is never stale.

2. **Hook into order creation:** in `services/orders.ts` → `createOrder(...)` (around line 537), after the order is saved, call `await ensureRouteForOrder(savedOrder)`. Make it best-effort (wrap in try/catch + log; a route-token failure must NOT fail order creation).

3. **Refresh on update:** in `updateOrder(...)` (same file), when `stops`/`preStops` change, call `ensureRouteForOrder` again to refresh the token's plan.

4. **`getActiveRouteForOrder`** stays the read path (driver endpoint + `/r/`). Optionally rename the create function `getOrCreateActiveRouteForOrder` if you prefer lazy-on-read as a safety net, but the PRIMARY trigger is order creation per the product decision.

## Design details / decisions
- **Mode:** `selectDispatchMode(flatStops)` → `ITINERARY` when there are prestops, else `DESTINATION` (single coordinate). The driver app/`/r/` already branch correctly on `route.kind`.
- **No channel send** on auto-create. Telegram/app push remains a separate explicit action.
- **Vehicle profile:** from `order.execution.fleetAssetId` if set; otherwise the routing defaults (`ROUTING_DEFAULTS` in `constants/routing.ts`).
- **TTL:** keep 14-day expiry. Since orders can be opened later, consider refreshing the token on order-open if expired (the driver endpoint can call `ensureRouteForOrder` when `getActiveRouteForOrder` returns null/expired).
- **Stops without prestops** still get a token (mode `destination`/`waypoints`); the `/r/` per-leg endpoint already falls back to single-coordinate for legs with no prestops.

## Constraints / conventions
- Follow `logifie-tms/.claude/CLAUDE.md`: services hold DB logic; thin API routes; `type` over `interface`; **no code comments**; don't create unnecessary files.
- Never leak forbidden order keys to the driver (the endpoint already runs `assertNoForbiddenOrderKeys`; `route` is returned as a sibling of `order`, not inside it — keep it that way).
- Soft-delete aware (`isDeleted: { $ne: true }`).
- Order ownership for drivers = `execution.driverEmployeeId`.

## Verification
- `pnpm typecheck` (note: `__tests__/driver/push-token.route.test.ts` has 2 PRE-EXISTING unrelated errors — ignore those).
- Add unit tests for `ensureRouteForOrder` (creates plan when ≥2 routable points; skips when <2; refreshes on stop change; doesn't send a channel message).
- E2E sanity: create an order with stops+prestops → fetch `/api/driver/v1/orders/<id>` → assert `data.route.token` present and `kind:"itinerary"` → GET `/api/r/<token>/route.json?leg=1` returns a valid `precomputed:false` itinerary.
- Confirm order creation still succeeds if route minting throws (best-effort).

## Process
- Review the change with 2 independent agents (one code-quality, one security) before finishing.
- Git: stage only; do not commit/push unless I say so.
