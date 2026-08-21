# CABS Parking Router — Project Plan

**Goal (v1):** Given your full weekly class schedule (multiple buildings), tell you the fastest path from car → class and class → car, choosing between Carmack and Buckeye lots, including walk time, bus wait, and ride time. Single user (you), real-time bus data stubbed for now.

---

## 1. Data Model

Design these as if other users/permits will exist later, even though v1 only needs your case.

### `permits`
```
{ code: "CXC", name: "Buckeye Lot (Student)", lots: ["carmack", "buckeye"] }
```
CampusParc (osu.campusparc.com) lists every permit type with its own page (CXC, CXS, C, CG, CE, WC, WCE, WCO, D, WD, CXD, etc.) — each page states which lots/garages that permit can use. You only need to transcribe CXC for now; the shape supports adding more later without a rewrite.

### `lots`
```
{ id: "carmack", name: "Carmack Lot", lat: null, lng: null }
{ id: "buckeye", name: "Buckeye Lot", lat: null, lng: null }
```
**TODO:** Get real coordinates. Buckeye Lot's address is 2701 Fred Taylor Dr, Columbus, OH 43210 — geocode that. Carmack has 5 sub-lots (Carmack 1, 2/3, 4, 5); for v1 you can treat "Carmack" as one point (wherever you actually park) and refine later if it matters which sub-lot.

### `stops`
```
{ id: "...", name: "...", lat: ..., lng: ..., routes: ["CC", "MC", ...] }
```
CABS runs 5 routes across 46 stops. The route/stop list is published on ttm.osu.edu/cabs (route pages list stops in order per direction) — no bulk download found, so this will need manual transcription from those route pages once, or scraping the page HTML. Worth checking the "Ohio State" app or Moovit's CABS listing too — Moovit apps are usually built on top of a GTFS feed, so if one exists publicly, that's less manual work than typing in 46 stops by hand.

### `buildings`
```
{ id: "...", name: "...", lat: ..., lng: ... }
```
**TODO:** Only need the buildings on your actual schedule for v1 — a handful of lookups, not all of campus. Easiest: MapKit JS geocoding (see below) or just pull lat/lng off Google Maps/Apple Maps for each building once.

### `schedule`
```
{ day: "Mon", start: "09:10", end: "10:05", building: "..." }
```
One entry per class meeting (repeat for MWF vs TTh as separate entries, not a single class object) — this format makes "what's my next class" and "when do I need to leave" trivial to compute.

---

## 2. Real-Time Bus Data — Full Walkthrough

### 2.0 The concept, from scratch

The "Ohio State" app on your phone shows you a live bus position on a map. It doesn't calculate that position itself — a computer somewhere else (OSU's servers, or a vendor's servers) tracks the actual GPS location of every bus, and the app just **asks that server "where are the buses right now?"** every few seconds and draws the answer on a map.

That "asking" happens over the internet using HTTP — the same protocol your browser uses to load a webpage — except instead of getting back an HTML page to display, the app gets back raw data, usually in a format called **JSON** (JavaScript Object Notation — just text that looks like this):
```json
{
  "busId": "CC-4",
  "route": "Campus Connector",
  "lat": 40.0067,
  "lng": -83.0305,
  "nextStopEta": 3
}
```
This request/response exchange is called an **API call**. The address it's sent to (e.g. `https://api.something.osu.edu/buses/live`) is called an **endpoint**. Your goal is: find that address, find out how to talk to it correctly, and then your own app can ask it the exact same question the official app does — you don't need OSU's permission or a special partnership to do this, you just need to know the address and the "shape" of the request, which you get by watching what the official app already sends.

### 2.1 How you "watch" what an app sends: a proxy

Normally you can't see the traffic between an app and its server — it's encrypted (HTTPS) and happens invisibly. A **proxy tool** sits between your phone and the internet and lets you see every request and response, even encrypted ones, by having your phone trust a special certificate the tool installs. This is a completely standard, legal technique developers use all the time to understand APIs (it's how most "unofficial" apps for things like this get built) — you're just watching your own phone's own traffic.

Two tools that do this, for iOS:
- **Proxyman** — has a native iOS app, easiest to set up, free tier is enough for this.
- **mitmproxy** — free, runs on a computer, your phone connects through it over Wi-Fi. More manual setup, more powerful.

I'd start with **Proxyman**, since you're new to this — it's built for exactly this workflow and has iOS-specific setup instructions in-app.

### 2.2 Step-by-step

1. **Install Proxyman** on your phone (App Store) and/or on your computer if you'd rather watch from there while your phone connects through it.
2. **Install Proxyman's root certificate** on your phone — the app walks you through this. This is what lets Proxyman "unlock" encrypted traffic. (This only affects traffic you're watching yourself — it doesn't send your data anywhere, it just lets your phone show you what's normally hidden.)
3. **Start capturing**, then open the official "Ohio State" app and navigate to wherever it shows live bus positions/ETAs.
4. **Watch the list of requests appear in Proxyman** as the app loads bus data. You're looking for one that:
   - Has a URL that isn't just loading images/fonts/analytics
   - Returns a response body full of numbers that look like coordinates (around lat 40.0, lng -83.0 for Columbus) or arrival-time predictions
5. **Tap into that request** in Proxyman to see:
   - The full URL and any query parameters
   - The **request headers** — this is where you'll find out if it needs an `Authorization` header (a token proving you're logged in) or an API key
   - The **response body** — the actual JSON with bus data

### 2.3 What you're likely to find, and what to do with it

Given the old "OSU Transportation Route Information Program" was taken down after a 2017 security breach, whatever powers the current app was probably rebuilt with more care. Two likely scenarios:

- **Scenario A — no auth needed, or a static key.** The request works with no `Authorization` header, or with a header value that's the same every time (a fixed API key baked into the app). This is the easy case: copy that exact request (URL + headers) into your own app's fetch call, and you're done — it'll work the same way every time.
- **Scenario B — tied to your OSU login.** The request includes a token that looks like a long random string (a JWT typically has two dots in it, like `xxxxx.yyyyy.zzzzz`) and it's probably short-lived (expires after some time, requiring the app to log in again behind the scenes). This is more work: you'd need to figure out how the app originally got that token (likely by you logging in with your OSU credentials somewhere), and your app would need to do the same login flow to get its own fresh token periodically. This is more invasive (you're now handling your own OSU credentials in your app) and worth pausing on rather than assuming it's fine — if it's this case, come back and we can figure out the least risky way to handle it, or fall back to a lower-fidelity data source instead.

### 2.4 The stub, so you're not blocked either way

Regardless of which scenario you hit, build against this interface now so your routing logic never has to wait on the reverse-engineering:
```js
// returns minutes until next bus at this stop, for a given route
async function getNextArrival(stopId, routeId) {
  // STUB: return a plausible fake value (e.g. random 2-12 min)
  // LATER: replace internals with the real fetch call you found via Proxyman
}
```
Once you've found the real endpoint, only the *inside* of this one function changes — nothing else in your app needs to know or care where the number came from.

---

## 3. Walking Directions — Open Source, No Apple Dependency

Swapped out MapKit JS for this reason: it requires your Apple Developer account to keep working, and you don't want the app to break if that account lapses. Two open-source options instead — same idea (send two coordinates, get back a walking time), no Apple account involved.

### Option A: OpenRouteService (fastest to set up)
- Free API key from a normal email signup at openrouteservice.org — not tied to Apple, not tied to any developer program.
- Free tier: 2,000 requests/day, which is plenty for personal use.
- The routing engine itself is open source, so even the *service* isn't a black box.
- Call their `directions` endpoint with `profile=foot-walking` and two coordinates; response includes `duration` in seconds.

### Option B: Self-hosted OSRM (fully independent, more setup)
- OSRM (Open Source Routing Machine) is open-source software you run yourself — on your own computer or a cheap always-on server.
- You download an OpenStreetMap data extract for Ohio (free, from Geofabrik), run OSRM against it in a Docker container, and it answers routing queries locally.
- Zero external account, zero rate limit, nothing that can be taken away from you later — you own the whole stack.
- More setup work upfront (installing Docker, downloading the extract, running the container), but genuinely bulletproof long-term, and it's a legitimate systems project in its own right if that interests you.

**Recommendation:** start with OpenRouteService to get the app working end-to-end quickly, and treat self-hosting OSRM as a natural "harden it" step once the core app is solid.

---

## 4. Routing Algorithm (pseudocode)

```
function bestRoute(fromCar: bool, targetBuilding, permit):
  candidateLots = permit.lots
  best = null

  for lot in candidateLots:
    origin = fromCar ? lot : targetBuilding
    destination = fromCar ? targetBuilding : lot

    for stop in stopsNear(origin, maxWalkMinutes=15):
      walk1 = walkTime(origin, stop)  // via OpenRouteService (or self-hosted OSRM later)

      for route in stop.routes:
        wait = getNextArrival(stop.id, route)
        for dropStop in stopsOnRoute(route, near=destination):
          rideTime = scheduledRideTime(stop, dropStop, route)  // static, from schedule/observation
          walk2 = walkTime(dropStop, destination)

          total = walk1 + wait + rideTime + walk2
          if best == null or total < best.total:
            best = { lot, stop, route, dropStop, total }

  return best
```

A few notes:
- `rideTime` between two stops on the same route can start as a rough static estimate (distance / avg CABS speed, or just eyeballed from riding it) — refine once you have real-time vehicle position data to calibrate against.
- Cap the walk-time search radius (e.g. 15 min) so you're not evaluating every stop on campus for every query.
- Run this twice per class: once for arrival (car → building) with your class start time as the deadline, once for departure (building → car) right after class ends.

---

## 5. MVP Scope (confirmed)

- Your full weekly schedule, multiple buildings — not hardcoded to one class.
- Still single-user — no accounts, no permit generalization beyond CXC, no multi-tenant backend yet.
- Real-time feed stubbed; walking times real (via OpenRouteService); ride times roughly estimated.
- Single HTML file, "Add to Home Screen" install, no App Store yet.

---

## 6. Suggested File Structure (still one HTML file)

Keep it one file for now, but organize into clearly separated sections so nothing needs a rewrite when you swap pieces later:

```html
<script>
  // 1. DATA — permits, lots, stops, buildings, schedule (plain objects/arrays)
  // 2. WALKING — walkTime(a, b) wrapping OpenRouteService (swap for self-hosted OSRM later)
  // 3. REALTIME — getNextArrival(stopId, routeId) — stub for now
  // 4. ROUTING — bestRoute(...) per the pseudocode above
  // 5. UI — render schedule, trigger routing on class change / button press, display result
</script>
```
Keeping UI code out of the data-fetching functions (no fetch calls buried inside render functions) means swapping the stub for the real feed, or later moving to a backend, only touches section 3.

---

## 7. Open Research TODOs Before/While Coding

- [ ] Transcribe CABS stop coordinates (from ttm.osu.edu/cabs route pages, or find if a GTFS feed exists via Moovit's backend)
- [ ] Geocode Carmack and Buckeye lot coordinates
- [ ] Geocode your specific class building(s)
- [ ] Sign up for a free OpenRouteService API key
- [ ] Install Proxyman on your phone, install its certificate, do a first capture session against the official Ohio State app's bus tracker
- [ ] Identify the live-bus request in the capture; note whether it needs an auth header (and if so, what kind — static key vs. login-based token)
- [ ] Decide how you're estimating ride time between stops until real-time data calibrates it

---

## 8. Later Phase (not now, just so data shapes don't need rework)

- Backend that polls whatever real-time endpoint you find and caches it, so multiple users' devices aren't each hammering OSU's servers.
- Full permit → lot table (all CampusParc permit types) and full building/stop coverage.
- Capacitor wrap for an actual App Store build — same HTML/CSS/JS, thin native shell, only worth doing once the tool is good and you're ready to distribute. Watch for OSU trademark issues (CABS name, colors/logo) in the App Store listing and icon.
