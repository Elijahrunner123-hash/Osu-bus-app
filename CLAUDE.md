# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

## Repository status

`index.html` is the entire app — no build step, package manager, server, or test suite. Geolocation needs a secure context, so run it over `http://localhost` (`python3 -m http.server`), not `file://`. Append `?now=ISO` (e.g. `?now=2026-08-24T09:00`) to override the clock for testing arrive/leave logic.

To sanity-check routing without a browser (no browser automation exists in this environment), extract the app's inline `<script>` and run it in a Node `vm` context with `document`/`localStorage`/`fetch`/`navigator` stubbed; top-level code is DOM-free and `init()` only runs when `document` exists, so the pure functions (`currentTripContext`, `transitLeg`, `planTrip`) can be called directly.

## Project overview

**CABS Parking Router** — a personal (single-user, v1) tool that, given a weekly OSU class schedule and the user's live location, computes the fastest route between car and class over parking lots (Carmack vs. Buckeye), walking, and CABS (Campus Area Bus Service) legs, and shows it on a map. Time of day auto-selects the direction: **arrive** (drive to the best lot, transit to class) before class, **leave** (transit back to the parked car) during/after.

Full design rationale, data shapes, and research TODOs live in `cabs-router-plan.md` — read it before architectural changes; the notes below only cover what's implemented.

## Architecture

`index.html` is organized into labeled sections so any one can be swapped without touching the others. Keep data-fetching (WALKING, REALTIME, LOCATION) out of render functions.

1. **DATA** — `permits`, `lots`, `stops`, `buildings`, `schedule` (shapes in plan §1). Populated only for the user's Biology 1113 schedule and just enough stops (Buckeye Lot, Carmack 2, Arps Hall) to route it; most of the ~46 CABS stops/buildings aren't transcribed yet. Lot coords are the corresponding bus-stop coords, not the lots' centers. Design shapes to generalize beyond the single CXC permit / two-lot case (`permits.lots` is a list) per plan §1/§8.
2. **WALKING / DRIVING** — `routeLeg(a, b, profile)` wraps the OpenRouteService `directions` API and returns duration + path geometry; `walkRoute`/`carRoute` (and `walkTime`/`driveTime`) are thin wrappers over the `foot-walking`/`driving-car` profiles. With no ORS key (Settings, stored in `localStorage`), it falls back to a straight-line haversine estimate and sets `usedEstimateFallback`/`estimated` flags so the UI can label results approximate. Not MapKit JS, to avoid an Apple Developer account dependency.
3. **REALTIME** — `getNextArrival(stopId, routeId)` is still a stub returning a random 2–12 min value. Swapping it for a real feed means reverse-engineering the CABS live-bus endpoint via a traffic proxy against the official "Ohio State" app (plan §2); only this function's internals should change.
4. **LOCATION & TIME** — `getUserLocation()` wraps the Geolocation API and resolves to a campus-center fallback (flagged `estimated`) on denial rather than rejecting. `currentTripContext(now)` is pure: from `schedule` it picks today's most-relevant class and returns `{ mode: "arrive"|"leave", entry, today }`.
5. **ROUTING** — `transitLeg(from, to)` returns the faster of a direct walk vs. the best board→ride→alight bus option (15 min walk cap), as a generic `legs` array. `planTrip({ mode, origin, entry, permit })` orchestrates it: **arrive** adds a `carRoute` drive leg to each candidate lot, keeps the fastest, and persists the winning lot to `localStorage`; **leave** routes from `origin` back to that remembered lot. `scheduledRideTime` is a static haversine estimate, not calibrated to real bus speed.
6. **MAP** — Leaflet + CARTO Voyager tiles (no key/account, Apple-Maps-like). Draws one polyline per leg (drive solid, walk dotted, bus straight — real CABS shapes aren't sourced yet), stop dots, a destination pin, and a live "you are here" dot.
7. **UI** — full-screen map with a draggable frosted bottom sheet (Apple styling, San Francisco via `-apple-system`, light/dark). On load it gets location + trip context and routes automatically; an Arrive/Leave segmented control and class chips override the time-of-day guess.

## Key constraints from prior decisions

- Real-time bus data can be obtained by observing the official "Ohio State" app's own network traffic (via a local proxy), but an easier method may exist.
- If that endpoint requires OSU-login-based auth (a token/JWT, not a static key), pause rather than assume it's safe — it means the app would handle the user's own OSU credentials.
