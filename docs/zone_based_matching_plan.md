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

## Data Model (PostgreSQL + PostGIS)

### Tables
- **zones**: `id`, `name`, `geom`, `meet_points` (JSON array of `{name, lat, lon}`), `active`.
- **users**: `id`, `name`, `role`, verification flags (SSO, license, insurance), rating stats, safety flags.
- **driver_trips**: trip metadata including direction, date, `time_window_start/end`, `seat_total`, `seat_taken`, ordered `zone_ids_pass_through`, notes.
- **rider_requests**: `direction`, date, `time_target`, `time_flex_minutes` (default 20), `zone_id`, status, `matched_driver_trip_id`.
- **matches_log**: record of candidate matches with ranking position and acceptance outcome.
- **safety_reports**: submissions with categories, notes, and timestamps.

## Technology Stack
- **Mobile**: React Native (iOS/Android) or Flutter, with a responsive web fallback via React PWA.
- **Backend**: Node.js (NestJS/Express) or Python (FastAPI).
- **Database**: PostgreSQL + PostGIS for zone geometry and point-in-polygon checks.
- **Caching**: Redis for hot zone lookups and peak-hour queries.
- **Auth**: UNM SSO via OIDC/SAML.
- **Maps/Routing**: Mapbox Directions, Google Directions, or self-hosted OSRM (routing only on confirm).
- **Notifications**: Firebase Cloud Messaging and APNs; email fallback via campus SMTP.

## API Sketch
- `GET /zones` – Retrieve active zones and meet points.
- `POST /driver/trips` – Create/update driver trip templates.
- `POST /rider/requests` – Submit a ride request.
- `GET /matches/zone-first` – Fetch zone-first driver candidates.
- `POST /matches/confirm` – Run detour check and book a seat if within threshold.
- `POST /messages/:trip_id` – In-app chat with masked contact details.
- `POST /safety/report` – File safety reports.

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

