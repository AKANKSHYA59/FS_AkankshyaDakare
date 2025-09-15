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
