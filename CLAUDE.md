# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

Implementation has started: `index.html` is the entire app (no build step, no package manager). Open it directly in a browser to run it — there's no server, bundler, or test suite. To sanity-check the routing logic without a browser, extract the `<script>` block and run it in a Node `vm` context with `document`/`localStorage`/`fetch` stubbed (this is how changes to `bestRoute`/`walkRoute` have been smoke-tested so far, since no browser automation tool is available in this environment).

## Project overview

**CABS Parking Router** — a personal (single-user, v1) tool that, given a weekly OSU class schedule spanning multiple buildings, computes the fastest car ↔ class route by choosing between parking lots (Carmack vs. Buckeye), walking legs, and CABS (Campus Area Bus Service) bus legs, and shows the result on a map.

Full design rationale, data shapes, and research TODOs live in `cabs-router-plan.md` — read it before making architectural changes, since the notes below only cover what's implemented and how it maps to that plan.

## Architecture

`index.html` is organized into six clearly separated sections so any one piece can be swapped without touching the others:

1. **DATA** — plain arrays for `permits`, `lots`, `stops`, `buildings`, `schedule` (shapes defined in plan §1). Currently populated only for the user's real Biology 1113 schedule (Campbell Hall lecture, Jennings Hall lab) and just enough stops (Buckeye Lot, Carmack 2, Arps Hall) to route between them — most of the ~46 CABS stops and other buildings are not yet transcribed. Lot coordinates are the corresponding CABS bus stop coordinates, not the lots' geometric centers.
2. **WALKING** — `walkRoute(a, b)` wraps the OpenRouteService `directions` API (`profile=foot-walking`) and returns both duration and path geometry; `walkTime(a, b)` is a thin wrapper around it for callers that only need the number. When no ORS API key is set (Settings panel, stored in `localStorage`), both fall back to a straight-line haversine estimate instead of failing, and set `usedWalkEstimateFallback`/`estimated` flags so the UI can label results as approximate. Deliberately not MapKit JS, to avoid an Apple Developer account dependency.
3. **REALTIME** — `getNextArrival(stopId, routeId)` is still a stub returning a random 2–12 min value; it has **not** been swapped for a real feed. That requires reverse-engineering the CABS live-bus endpoint via a traffic proxy (e.g. Proxyman) against the official "Ohio State" app, per plan §2 — not yet done. Only this function's internals should change when that happens.
4. **ROUTING** — `bestRoute(fromCar, targetBuilding, permit)` implements the pseudocode in plan §4: for each lot the permit allows, search nearby stops (15 min walk cap) and routes from those stops, summing walk + wait + ride + walk to find the minimum-time path. `scheduledRideTime` is a static haversine-distance estimate, not calibrated against real bus speed. The winning result also carries the walking-leg geometries through for the map.
5. **MAP** — Leaflet + raw OpenStreetMap tiles (no API key/account, same rationale as the walking-directions choice). Draws real walking paths per leg (dashed when using the estimate fallback) and a straight dashed line for the bus leg, since actual CABS route shapes aren't sourced yet.
6. **UI** — renders the schedule and triggers routing on button press, then hands the result to the MAP section. Fetch calls must stay out of render functions — keep data-fetching (sections 2–3) and UI (section 6) strictly separated.

Design the DATA shapes to generalize beyond the single CXC permit / two-lot case even though v1 only needs that one case (e.g. `permits.lots` is a list, not a hardcoded pair) — see plan §1 and §8 for what "later phase" generalization looks like.

## Key constraints from prior decisions

- No Apple/MapKit dependency for walking directions — use OpenRouteService (or later, self-hosted OSRM) instead.
- Real-time bus data must be obtained by observing the official "Ohio State" app's own network traffic (via a local proxy tool), not by guessing or scraping undocumented endpoints blindly.
- If the real-time endpoint turns out to require OSU-login-based auth (a token/JWT rather than a static key), treat that as a decision point worth pausing on rather than assuming it's safe to implement — it means the app would handle the user's own OSU credentials.
