# Zone-Based Matching Plan for UNM-Taos Rideshare

## Overview
This document outlines a hybrid zone-and-route matching model tailored for The University of New Mexico-Taos pilot. The goal is to keep most interactions fast and lightweight while ensuring detours remain reasonable when riders and drivers finalize a match.

## Product Shape

### Core Flows
- **Rider**: Select a pickup zone, choose "arrive by" or "depart at" time (±20 minutes), and request a seat.
- **Driver**: Declare the zones they pass through (ordered), seat availability, and a time window, then accept or decline rider requests.
- **Both**: Meet only at pre-approved public landmarks within each zone; avoid private residences.

### Two-Tier Matching
1. **Zone-first match** (primary workflow)
   - Surfaces drivers whose trips pass through the rider's zone within the overlapping time window.
   - Minimizes reliance on live routing APIs, keeping the experience responsive and cost-effective.
2. **Optional route confirmation**
   - Triggered when either party taps "Preview" or "Confirm".
   - Executes a single routing call to validate that the pickup detour is ≤3–5 minutes before a booking is finalized.

## Zones and Meet Points
Use simple circles (250–400 m radius) or polygons with one to three named meet points. Initial Taos pilot zones:

| Zone | Primary Meet Point | Alternate Meet Point |
| --- | --- | --- |
| Taos Plaza | Plaza Gazebo | Kit Carson Park lot |
| Ranchos de Taos | San Francisco de Asís Church lot | Ranchos Post Office |
| Southside / Walmart | Walmart main lot far row | — |
| Northside / Cid's Market | Cid's overflow area | Taos Center for the Arts lot (evenings) |
| Taos Pueblo Gate | Pueblo visitor parking | — |
| El Prado / US-64 Junction | Smith's far NW row | — |
| Arroyo Seco | Seco Plaza public lot | — |
| Orilla Verde / St. Francis Bridge Park-n-Ride | Designated pullout lot | — |
| UNM-Taos Klauer (campus) | North Lot, South Lot, Bus Stop circle | — |

Future expansion zones: Dixon, Questa, Peñasco.

## Matching Logic

### Inputs
- **Rider**: `zone_id`, `time_target`, `direction` (to campus / from campus), ±20 minute flexibility.
- **Driver**: `zone_ids_pass_through` (ordered), `seat_count`, `time_window`, `direction`.

### Matching Steps
1. **Zone pass**
   - Filter by matching direction and overlapping time windows.
   - Require the driver's ordered zone list to include the rider's zone (to campus) or to originate from a campus-adjacent zone (from campus).
2. **Capacity and policy checks**
   - Ensure available seats remain (`seat_total > seat_taken`).
   - Enforce blocks, safety flags, and rider/driver eligibility.
3. **Optional route verification**
   - Baseline route: driver origin → campus at the chosen time.
   - Pickup variant: driver origin → rider zone meet point → campus.
   - Detour = `variant_time – baseline_time`; reject if >3–5 minutes.
4. **Ranking**
   - Primary sort: smallest time delta within the same zone.
   - Tiebreakers: driver acceptance history, ratings, no-show records.

### Meet Point Selection
- **Preferred meet point resolution**
  1. Start with the rider's explicitly ranked meet points for the zone (defaulting to program-suggested ordering when no rider preference exists).
  2. Score each candidate using a blend of (a) distance/time off the driver's declared baseline path and (b) driver-stated meet point preferences captured from past confirmations.
  3. Select the top-scoring meet point that satisfies any accessibility constraints (e.g., van-accessible, lighting requirements). If all options are equal, prefer the rider's highest-ranked choice.
- **Routing hand-off**
  - Once selected, surface the meet point's canonical coordinates (`lat`, `lon`) to the matching service so the detour computation uses the exact waypoint fed to routing APIs (e.g., Mapbox Directions). Engineers should pass the baseline origin/destination plus this single resolved waypoint to ensure consistency between the meet-point picker and detour validator.
- **UX hooks**
  - Riders may reorder meet points within a zone in settings; the backend stores the ranking to seed step 1.
  - Drivers can opt-in to "preferred pickup" toggles per zone (e.g., "avoid plaza"), which seed step 2's scoring weight.
  - Future experiments can expose "remember last accepted meet point" so the system nudges toward previously successful exchanges while still respecting rider accessibility needs.

## Data Model (PostgreSQL + PostGIS)

### Tables
- **zones**: `id`, `name`, `geom`, `meet_points` (JSON array of `{name, lat, lon}`), `active`.
- **users**: `id`, `name`, `role`, verification flags (SSO, license, insurance), rating stats, safety flags.
- **driver_trips**: trip metadata including direction, date, `time_window_start/end`, `seat_total`, `seat_taken`, notes. The ordered path information moves to `driver_trip_zones` for normalization.
- **driver_trip_zones**: `trip_id`, `zone_id`, `sequence_index`. Stores the ordered zones a driver declares for a specific trip and supports waypoints that can be reused across templates.
- **rider_requests**: `direction`, date, `time_target`, `time_flex_minutes` (default 20), `zone_id`, status, `matched_driver_trip_id`.
- **matches_log**: record of candidate matches with ranking position and acceptance outcome.
- **safety_reports**: submissions with categories, notes, and timestamps.

### Querying ordered zones
Use `driver_trip_zones` to enforce the rider's pickup occurs in the same order a driver travels. A typical query:

```sql
SELECT dt.id AS driver_trip_id
FROM driver_trips dt
JOIN driver_trip_zones dz_pickup
  ON dz_pickup.trip_id = dt.id
JOIN driver_trip_zones dz_dropoff
  ON dz_dropoff.trip_id = dt.id
WHERE dz_pickup.zone_id = :rider_zone_id
  AND dz_dropoff.zone_id = :campus_zone_id
  AND dz_pickup.sequence_index < dz_dropoff.sequence_index
  AND dt.direction = :direction
  AND tstzrange(dt.time_window_start, dt.time_window_end) && :rider_time_window;
```

The rider zone is joined to the same trip twice: once for the pickup (`dz_pickup`) and once for the subsequent campus (or other destination) zone (`dz_dropoff`). Comparing `sequence_index` ensures the driver reaches the rider before the campus leg. Additional filters (seat availability, safety blocks) layer on top of this base query.

### Indexing strategy
- Composite B-tree on `(trip_id, sequence_index)` for `driver_trip_zones` to accelerate sequential scans within a trip and support `ORDER BY sequence_index` when presenting waypoints.
- Composite B-tree on `(zone_id, sequence_index)` to quickly find drivers passing through a zone and understand their relative ordering without scanning all trips.
- Existing indexes on `driver_trips` (direction, time window ranges) remain important for filtering before joining to `driver_trip_zones`.

## Technology Stack
- **Mobile**: React Native (iOS/Android) or Flutter, with a responsive web fallback via React PWA.
- **Backend**: Node.js (NestJS/Express) or Python (FastAPI).
- **Database**: PostgreSQL + PostGIS for zone geometry and point-in-polygon checks.
- **Caching**: Redis for hot zone lookups and peak-hour queries.
- **Auth**: UNM SSO via OIDC/SAML.
- **Maps/Routing**: Mapbox Directions, Google Directions, or self-hosted OSRM (routing only on confirm).
- **Notifications**: Firebase Cloud Messaging and APNs; email fallback via campus SMTP.

## API Sketch

All endpoints require a **Bearer token** minted after a successful UNM OIDC login. Tokens are issued by the campus identity provider and carry user role claims that the API must validate for every request. Requests lacking a valid token (signature, issuer, audience) must be rejected with `401 Unauthorized`; tokens missing required role claims must receive `403 Forbidden`.
- `GET /zones` – Retrieve active zones and meet points. Accepts tokens with any of the roles below because the directory is shared across rider and driver experiences.
- `POST /driver/trips` – Create/update driver trip templates. Requires tokens with the `driver` role/claim in addition to the default `user` claim so only vetted drivers publish routes.
- `POST /rider/requests` – Submit a ride request. Requires the `rider` role/claim plus the base `user` claim.
- `GET /matches/zone-first` – Fetch zone-first driver candidates. Requires `rider` claim when scoped to rider search; `driver` claim when used for driver dashboards. Admin tokens may access both modes for troubleshooting.
- `POST /matches/confirm` – Run detour check and book a seat if within threshold. Requires `rider` claim to request a booking and `driver` claim to confirm acceptance. Admin tokens can override to resolve escalations.
- `POST /messages/:trip_id` – In-app chat with masked contact details. Requires either `rider` or `driver` claim and must validate that the caller participates in the trip thread; admins may access for moderation.
- `POST /safety/report` – File safety reports. Accepts any authenticated `user` (rider, driver, admin) and additionally allows admins to fetch or triage reports via future `GET /safety/report` endpoints.

### Roles and Claims

- **Base Claims**
  - `user`: present on all issued tokens once OIDC login completes.
  - `sub`: stable user identifier from UNM OIDC.
- **Rider**
  - `role: rider` claim, added after rider onboarding (student/staff eligibility check).
  - Optional `rider_verified` boolean claim once safety checks (e.g., student status) pass; required for booking endpoints.
- **Driver**
  - `role: driver` claim, set after license/insurance verification.
  - Optional `driver_verified` claim to flag background/vehicle checks; required for trip publishing and confirmation routes.
- **Admin**
  - `role: admin` claim restricted to Student Affairs / program staff.
  - Optional scoped claims such as `admin:safety_review` or `admin:user_mgmt` to gate sensitive operations.

### Token Lifetime and Renewal

- **Access Tokens**: 60-minute lifetime to balance security with mobile usability.
- **Refresh Tokens**: 12-hour lifetime (rolling) issued alongside access tokens after OIDC login. Store in client keychain/secure storage and rotate on each use.
- **Renewal Flow**:
  1. Mobile/web clients silently call `/auth/refresh` with the refresh token when the access token is within five minutes of expiry.
  2. Backend validates the refresh token, issues new access and refresh pair, and revokes the previous refresh token (rotate-on-use).
  3. If refresh fails (e.g., revoked, expired), clients must prompt for full UNM SSO re-authentication.
- **Revocation Hooks**: Admin tools trigger token revocation when a user is suspended; backend must maintain a revocation list or rely on short-lived tokens plus refresh rotation.

## Safety, Privacy, and Operations
- Store only zone IDs and meet points—never home addresses.
- Highlight public, well-lit meet points and display first name + last initial with verification badges.
- Provide emergency share links and an always-visible code of conduct during booking.
- Offer admin tooling for Student Affairs to review reports and deactivate users.
- Keep the service free to avoid TNC classification; encourage optional off-platform tips.
- Consider campus parking perks for drivers completing 10+ carpools per month.
- Display clear peer-to-peer disclaimers to set expectations.

## MVP Scope
1. Zone directory with meet points and SSO authentication.
2. Driver trip templates (days/times, ordered zones).
3. Rider request flow with zone-first match list.
4. Booking confirmation and in-app chat.
5. Optional route detour check prior to final confirmation.

## Pilot Plan (UNM-Taos)
- **Duration**: 6–8 weeks.
- **Schedule focus**: Weekday arrivals (7–10 a.m.) and departures (3–6 p.m.).
- **Launch zones**: Taos Plaza, Ranchos de Taos, Northside/Cid's, Southside/Walmart, Taos Pueblo Gate, El Prado, Arroyo Seco, UNM-Taos campus.
- **Targets**: 40 active drivers, 120 riders, 300 successful matches.
- **Metrics**: Fill rate, average wait, average detour, cancellations, safety reports.

