# FS_AkankshyaDakare
Carpooling and route-sharing app concept for students - design, pseudocode, and system architecture.


*********Key design choices & trade-offs (thought process)**********

1 Geospatial approach

Options:

PostGIS geometry operations (accurate) vs H3 / geohash indexing (fast).
Choice: Hybrid — use H3 (or geohash) for fast prefilter in Redis; use PostGIS geometry (ST_Buffer / ST_Length / ST_Intersection) to compute precise overlap and intersection lengths for the top candidates.

Trade-off: H3 gives O(1)-like set union performance but coarser; PostGIS is precise but expensive. Hybrid yields speed + accuracy.

2 Routing engine

Options:

Use third-party Directions API (Google/Mapbox) vs host OSRM/GraphHopper.
Choice: Start with hosted Directions API for quicker MVP, switch to self-hosted OSRM at scale to reduce costs.

Trade-off: Hosted is faster to develop but expensive at scale. OSRM requires ops effort and tile hosting.

3 Messaging

Options:

Server-mediated WebSocket vs E2E encryption.
Choice: Server-mediated messaging (TLS, server-side encryption at rest) for moderation, reporting, and easier features. Consider E2EE later.

Trade-off: E2EE improves privacy but prevents moderation and features like searching message history and safety auditing.

4 Verification & anonymity

Verify school email via OTP to reduce bad actors.

Provide anon_username (adjective-noun-####). Store mapping server-side; enforce uniqueness. Do not expose school email or real name.

Trade-off: Email verification increases friction but improves safety.





*********— Testing, metrics & A/B ideas**********

Unit & integration tests

Route sampling correctness

H3 indexing correctness

Matching scoring (unit tests for scoring with synthetic routes)

Concurrency tests for seat booking (transactions)

Metrics to monitor

Match rate (journeys with at least one suggested match)

Acceptance ratio (% suggested → accepted)

Average detour minutes for accepted matches

Seat fill rate (seats filled per driver)

Safety metrics: reports per 1000 chats

A/B experiments

Weight tuning for scoring (overlap vs detour)

Buffer distance in overlap calculation (50m vs 100m)

Suggestion UI variations (show top 3 vs map-first)

************— Deployment & infra (brief)***************

Containerized services (Docker) → Kubernetes

Postgres + PostGIS on managed DB (RDS/Azure), Redis cluster, OSRM fleet on nodes with precomputed tiles

CI/CD via GitHub Actions → image registry → deploy to K8s

Observability: Prometheus + Grafana, Sentry for errors

****************— Roadmap / release plan****************

Week 0–2: Prototype frontend + simple backend; use Mapbox/Google Directions for routing; implement email OTP signup + anon username.

Week 3–6: Journey CRUD + H3 prefilter + PostGIS refine + in-app chat (managed service).

Week 7–10: Improve scoring, add accept/book flow, safety reporting, metrics dashboard.

Month 3–6: Scale: self-host OSRM, more campuses, ML-based personalized suggestions, reputation system.

********************************************PSEUDOCODE**************************
# -------------------------
# Utilities
# -------------------------
FUNCTION generate_unique_username():
    WHILE TRUE:
        name = random_adjective() + "-" + random_noun() + "-" + rand_int(1000,9999)
        TRY:
            INSERT INTO users(anon_username=name, ...)  # atomic uniqueness check
            RETURN name
        EXCEPT UniqueViolation:
            CONTINUE

FUNCTION sample_route(route_line, step_m=50):
    # walk along LineString and emit points approx every step_m meters
    return list_of_latlngs

FUNCTION h3_index_points(points, resolution=9):
    cells = empty_set
    FOR p IN points:
        cells.add(h3.latlng_to_cell(p.lat, p.lng, resolution))
    RETURN cells

# -------------------------
# Create journey
# -------------------------
FUNCTION create_journey(user_id, origin, destination, dep_start, dep_end, seats, role):
    route_line = routing.get_route(origin, destination)
    route_length_m = compute_length(route_line)
    journey_id = DB.insert_journey(user_id, origin, destination, route_line, route_length_m, dep_start, dep_end, seats, role)

    points = sample_route(route_line, 50)
    cells = h3_index_points(points, resolution=9)
    FOR cell IN cells:
        REDIS.sadd("h3:"+cell, journey_id)

    RETURN journey_id

# -------------------------
# Find matches
# -------------------------
FUNCTION find_matches_for_journey(journey_id, top_k=5):
    new_j = DB.get_journey(journey_id)
    points = sample_route(new_j.route_line, 50)
    cells = h3_index_points(points)
    candidate_ids = REDIS.union([ "h3:"+c for c IN cells ])   # fast
    candidate_ids.remove(journey_id)

    # quick fetch + time/filter
    candidates = DB.fetch_journeys(candidate_ids WHERE active AND seats_available>0 AND time_overlap)

    scored = []
    FOR c IN candidates:
        overlap_len = POSTGIS.intersection_length(new_j.route_line, c.route_line, buffer_m=50)
        overlap_percent = overlap_len / min(new_j.route_length_m, c.route_length_m)

        # only for top shortlisted compute expensive detour via routing engine
        detour_minutes = cheap_estimate_detour(new_j, c)

        time_diff_minutes = compute_time_diff(new_j, c)

        score = 0.6*overlap_percent - 0.25*normalize(detour_minutes) - 0.1*normalize(time_diff_minutes)
        IF score > threshold:
            scored.append((c.id, score))

    sort scored by score desc
    RETURN scored[:top_k]

# -------------------------
# Accept match / book seat
# -------------------------
FUNCTION accept_and_book(match_id, acceptor_user_id):
    match = DB.get_match(match_id)
    # verify acceptor is part of match
    IF acceptor_user_id not in (match.participant_a, match.participant_b):
        RETURN error "not participant"

    BEGIN TRANSACTION
        driver_journey = resolve_driver_journey(match)
        IF driver_journey.seats_available <= 0:
            ROLLBACK
            RETURN "no seats"

        UPDATE journeys SET seats_available = seats_available - 1 WHERE id = driver_journey.id
        UPDATE matches SET status='accepted' WHERE id = match_id
    COMMIT

    notify both parties
    RETURN success


******************USE MERMAID FOR IMAGES AND DIAGRAM*******************
ARCHITECTURE DESIGN CODE:
flowchart TD
    subgraph Frontend[📱 Student App]
        A1[Enter Home/Destination]
        A2[View Suggested Routes]
        A3[Chat with Matches]
    end

    subgraph Backend[☁️ Backend Services]
        B1[User Service - registration & anon usernames]
        B2[Journey Service - store routes]
        B3[Matching Engine - compare overlaps]
        B4[Chat Service - WebSocket]
    end

    subgraph Database[🗄️ Data Layer]
        C1[(User DB)]
        C2[(Journey DB - PostGIS)]
        C3[(Redis/H3 index for geospatial search)]
        C4[(Chat DB)]
    end

    subgraph External[🛠 External APIs]
        D1[Map API - routes/distance]
        D2[Auth/Email verification]
    end

    Frontend -->|REST/GraphQL| Backend
    Backend --> Database
    Backend --> External


DATAMODEL CODE DESIGN:
flowchart TD
    subgraph Frontend[📱 Student App]
        A1[Enter Home/Destination]
        A2[View Suggested Routes]
        A3[Chat with Matches]
    end

    subgraph Backend[☁️ Backend Services]
        B1[User Service - registration & anon usernames]
        B2[Journey Service - store routes]
        B3[Matching Engine - compare overlaps]
        B4[Chat Service - WebSocket]
    end

    subgraph Database[🗄️ Data Layer]
        C1[(User DB)]
        C2[(Journey DB - PostGIS)]
        C3[(Redis/H3 index for geospatial search)]
        C4[(Chat DB)]
    end

    subgraph External[🛠 External APIs]
        D1[Map API - routes/distance]
        D2[Auth/Email verification]
    end

    Frontend -->|REST/GraphQL| Backend
    Backend --> Database
    Backend --> External


FLOWCHART DESIGN CODE:
flowchart TD
    subgraph Frontend[📱 Student App]
        A1[Enter Home/Destination]
        A2[View Suggested Routes]
        A3[Chat with Matches]
    end

    subgraph Backend[☁️ Backend Services]
        B1[User Service - registration & anon usernames]
        B2[Journey Service - store routes]
        B3[Matching Engine - compare overlaps]
        B4[Chat Service - WebSocket]
    end

    subgraph Database[🗄️ Data Layer]
        C1[(User DB)]
        C2[(Journey DB - PostGIS)]
        C3[(Redis/H3 index for geospatial search)]
        C4[(Chat DB)]
    end

    subgraph External[🛠 External APIs]
        D1[Map API - routes/distance]
        D2[Auth/Email verification]
    end

    Frontend -->|REST/GraphQL| Backend
    Backend --> Database
    Backend --> External

