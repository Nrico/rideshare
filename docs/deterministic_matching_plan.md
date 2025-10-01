# Deterministic Matching Plan for Campus-Only Rideshare

## Goals
- Limit access to University of New Mexico-Taos participants via NetID/SSO login.
- Enable safe, privacy-preserving carpools that respect 20-minute arrival windows and 3–5 minute detour ceilings.
- Offer a dependable request/accept workflow without requiring AI or ML to launch.

## Core Feature Set

### Authentication & Identity
- Require UNM-Taos NetID single sign-on to create or access accounts.
- Profiles capture role (driver, rider, or both), typical commute schedule, and available vehicle seats.
- Display first name, last initial, verified photo, and optional license/insurance badges.

### Pickup & Privacy Controls
- Riders choose a public pickup landmark or a snapped point along a main corridor; never expose private addresses.
- Store generalized locations as 7–8 character geohashes or sanitized corridor points.
- Encourage “meet in public spots” via UI prompts and campus pickup defaults.

### Trip Windows & Requests
- Allow “arrive by” or “depart at” selections with ±20 minute flexibility.
- Trips can be requested and accepted up to 24 hours in advance.
- Provide cancellation reasons, block/report options, and a visible code of conduct.

### Communication & Safety
- In-app chat masks phone numbers while enabling coordination.
- Include name/photo verification, ratings, trip sharing links, emergency button, and report tools.
- Log all interactions for safety audits and Student Affairs review.

## Matching Workflow (Deterministic)

1. **Baseline Route** – Compute the driver’s campus trip without detours.
2. **Pickup Variant** – For each rider candidate, compute origin → rider pickup → campus.
3. **Detour Calculation** – `Δ = time(variant) – time(baseline)`; discard matches when Δ exceeds 3–5 minutes.
4. **Window Check** – Ensure the resulting arrival stays within the rider’s ±20 minute interval.
5. **Scoring** – Example score: `100 – (10 × Δ minutes) + (5 × schedule overlap score) + (2 × rating)`. Return the top N matches per user.

## Maps & Routing Options
- **Google Directions + Distance Matrix**: best traffic-aware detour estimates; paid usage.
- **Mapbox Directions Matrix**: strong matrix support; paid.
- **OSRM/OpenRouteService**: self-host to cut per-request costs at the expense of ops overhead.
- **Apple MapKit JS**: viable, but less mature matrix/isochrone tooling than Google/Mapbox.

Use live routing only during confirmation to keep costs down; rely on cached zone-based filtering for initial suggestions.

## Technology Stack
- **Mobile/Web**: React Native (iOS/Android) with responsive React web app fallback.
- **Backend**: Node.js (NestJS/Express) or Python (FastAPI) service layer.
- **Database**: PostgreSQL + PostGIS for zone geometry, meet points, and detour calculations.
- **Caching**: Redis for high-frequency match lookups.
- **Notifications**: Firebase Cloud Messaging, APNs, and campus SMTP email fallback.

## Data Handling
- Store hashed/sanitized location references, schedules, trip intents, ratings, and safety logs.
- Retain message history in-app; omit raw phone numbers or home addresses.
- Limit personal data to what is required for safety and compliance.

## Safety & Policy Checklist
- Restrict access to the UNM-Taos community (.edu emails + SSO).
- Publish terms of use, code of conduct, and “peer-to-peer, not a common carrier” disclosures.
- Keep launch version free of in-app payments; if adding payments later, consult legal and insurance.
- Provide tools to block, report, and document last-minute cancellations.
- Highlight campus pickup zones and “meet in public place” reminders.
- Offer optional driver document uploads for verification badges.

## AI Considerations
- Not required for MVP matching; deterministic routing-based checks are reliable and cost-effective.
- Future enhancements could include AI-assisted ranking explanations, safety signal detection in chats, or personalized preferences.

## MVP Launch Checklist
1. SSO-backed onboarding and profile creation.
2. Zone directory with approved meet points.
3. Driver trip declarations with ordered zones and seat counts.
4. Rider request flow with match scoring and 24-hour request window.
5. Deterministic detour validation during booking confirmation.
6. In-app chat with masked identities, emergency share, and report workflows.

