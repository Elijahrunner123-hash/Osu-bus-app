# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

This repository currently contains only a planning document (`cabs-router-plan.md`) — no code has been written yet. There is no build system, package manager, or test suite to document. When implementation begins, update this file with real build/lint/test commands.

## Project overview

**CABS Parking Router** — a personal (single-user, v1) tool that, given a weekly OSU class schedule spanning multiple buildings, computes the fastest car ↔ class route by choosing between parking lots (Carmack vs. Buckeye), walking legs, and CABS (Campus Area Bus Service) bus legs. Real-time bus data is stubbed in v1.

Full design rationale, data shapes, and research TODOs live in `cabs-router-plan.md` — read it before implementing, since the notes below are only the parts that constrain architecture.

## Intended architecture (per the plan)

The app is meant to ship as **a single HTML file** (installable via "Add to Home Screen", no App Store/backend in v1), but organized internally into five clearly separated sections so any one piece can be swapped later without a rewrite:

1. **DATA** — plain objects/arrays for `permits`, `lots`, `stops`, `buildings`, `schedule` (shapes defined in plan §1).
2. **WALKING** — a `walkTime(a, b)` function wrapping the OpenRouteService `directions` API (`profile=foot-walking`), returning duration in seconds. Deliberately not MapKit JS, to avoid an Apple Developer account dependency. Self-hosted OSRM is a possible later swap for the same function.
3. **REALTIME** — `getNextArrival(stopId, routeId)`: stubbed with a plausible fake value until the real CABS live-bus endpoint is reverse-engineered (via a traffic proxy like Proxyman against the official "Ohio State" app — see plan §2). Only this function's internals should change when the real feed is wired in; nothing else should need to know the data's source.
4. **ROUTING** — `bestRoute(fromCar, targetBuilding, permit)`: for each lot the permit allows, searches nearby stops (capped walk radius, e.g. 15 min) and CABS routes from those stops, summing walk + wait + ride + walk to find the minimum-time path. Pseudocode in plan §4. Called twice per class: once for arrival (car → building, deadline = class start) and once for departure (building → car, right after class ends).
5. **UI** — renders the schedule and triggers routing on class change / button press. Fetch calls must stay out of render functions — keep data-fetching (sections 2–3) and UI (section 5) strictly separated.

Design the DATA shapes to generalize beyond the single CXC permit / two-lot case even though v1 only needs that one case (e.g. `permits.lots` is a list, not a hardcoded pair) — see plan §1 and §8 for what "later phase" generalization looks like.

## Key constraints from prior decisions

- No Apple/MapKit dependency for walking directions — use OpenRouteService (or later, self-hosted OSRM) instead.
- Real-time bus data must be obtained by observing the official "Ohio State" app's own network traffic (via a local proxy tool), not by guessing or scraping undocumented endpoints blindly.
- If the real-time endpoint turns out to require OSU-login-based auth (a token/JWT rather than a static key), treat that as a decision point worth pausing on rather than assuming it's safe to implement — it means the app would handle the user's own OSU credentials.
