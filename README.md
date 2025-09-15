# FS_AkankshyaDakare
Carpooling and route-sharing app concept for students - design, pseudocode, and system architecture.
Key design choices & trade-offs (thought process)
4.1 Geospatial approach

Options:

PostGIS geometry operations (accurate) vs H3 / geohash indexing (fast).
Choice: Hybrid — use H3 (or geohash) for fast prefilter in Redis; use PostGIS geometry (ST_Buffer / ST_Length / ST_Intersection) to compute precise overlap and intersection lengths for the top candidates.

Trade-off: H3 gives O(1)-like set union performance but coarser; PostGIS is precise but expensive. Hybrid yields speed + accuracy.

4.2 Routing engine

Options:

Use third-party Directions API (Google/Mapbox) vs host OSRM/GraphHopper.
Choice: Start with hosted Directions API for quicker MVP, switch to self-hosted OSRM at scale to reduce costs.

Trade-off: Hosted is faster to develop but expensive at scale. OSRM requires ops effort and tile hosting.

4.3 Messaging

Options:

Server-mediated WebSocket vs E2E encryption.
Choice: Server-mediated messaging (TLS, server-side encryption at rest) for moderation, reporting, and easier features. Consider E2EE later.

Trade-off: E2EE improves privacy but prevents moderation and features like searching message history and safety auditing.

4.4 Verification & anonymity

Verify school email via OTP to reduce bad actors.

Provide anon_username (adjective-noun-####). Store mapping server-side; enforce uniqueness. Do not expose school email or real name.

Trade-off: Email verification increases friction but improves safety.





10 — Testing, metrics & A/B ideas

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

11 — Deployment & infra (brief)

Containerized services (Docker) → Kubernetes

Postgres + PostGIS on managed DB (RDS/Azure), Redis cluster, OSRM fleet on nodes with precomputed tiles

CI/CD via GitHub Actions → image registry → deploy to K8s

Observability: Prometheus + Grafana, Sentry for errors

12 — Roadmap / release plan

Week 0–2: Prototype frontend + simple backend; use Mapbox/Google Directions for routing; implement email OTP signup + anon username.

Week 3–6: Journey CRUD + H3 prefilter + PostGIS refine + in-app chat (managed service).

Week 7–10: Improve scoring, add accept/book flow, safety reporting, metrics dashboard.

Month 3–6: Scale: self-host OSRM, more campuses, ML-based personalized suggestions, reputation system.
