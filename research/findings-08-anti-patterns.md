# Findings: Anti-Patterns, Pitfalls & Failed Approaches to Avoid

## Executive Summary

This research catalogs the most dangerous anti-patterns, real-world failures, and costly mistakes that teams make when building real-time monitoring dashboards, OSINT platforms, and data ingestion systems. The findings are organized into six domains: failed open-source projects, database anti-patterns (including Elasticsearch and Supabase pitfalls), RSS/feed ingestion mistakes, frontend dashboard traps, architecture overengineering, and Supabase-specific gotchas. The overarching lesson is clear: **start simple, ship fast, and add complexity only when concrete pain demands it.** Every anti-pattern documented here stems from the same root cause -- solving imaginary future problems instead of real present ones.

## Research Scope

This document responds to Research Brief 08, covering:
- Failed open-source OSINT/dashboard projects and why they died
- Database anti-patterns (Elasticsearch as primary store, Supabase scaling limits, PostgreSQL mistakes)
- RSS/feed ingestion anti-patterns (polling, parsing, backoff)
- Frontend dashboard anti-patterns (memory leaks, WebSocket issues, map rendering)
- Architecture anti-patterns (microservices for small teams, Kafka overkill, premature optimization)
- Supabase-specific gotchas and the self-hosted vs managed decision

---

## Detailed Findings

### 1. Failed Open-Source OSINT/Dashboard Projects

#### Why 95% of Open-Source Projects Die Within a Year

Academic research (Coelho & Valente, "Why Modern Open Source Projects Fail," ESEC/FSE 2017) surveyed maintainers of 104 deprecated GitHub projects and identified nine root causes. The most relevant to Sentinel:

| Failure Reason | Frequency | Relevance to Sentinel |
|---|---|---|
| **Usurped by competitor** | 30 out of 104 projects | High -- commercial platforms (Dataminr, Recorded Future) dominate this space |
| **Lack of time** | Very common | High -- a small team building an ambitious platform will hit this wall |
| **Project became obsolete** | Common | Medium -- data source APIs change frequently |
| **Scope creep beyond maintainer capacity** | Common | Critical -- dashboard projects are scope magnets |

#### The "Truck Factor" Problem

The "Truck Factor" is the minimum number of developers who must leave before a project becomes unsustainable. Most OSINT dashboard projects on GitHub (e.g., `kotIIT/ITMS448-osint-dashboard`, `campwill/osint-dashboard`) are academic class projects or single-developer efforts. They go dormant the moment the creator moves on. The pattern is predictable: ambitious README, initial burst of commits, then silence.

#### Concrete Lessons for Sentinel

1. **Do not build features nobody asked for.** Dashboard projects die from feature bloat, not feature scarcity.
2. **External API dependencies are a time bomb.** When Google changed its sync API, the Grive project died overnight. ACLED, GDELT, or ReliefWeb API changes could do the same to Sentinel. Build adapter layers, not tight couplings.
3. **CI/CD and contributing guidelines have a strong statistical association with project survival.** Set these up from day one.
4. **Plan the handoff from the start.** Document architecture decisions. Write ADRs (Architecture Decision Records). Make it possible for someone new to pick up the project.

---

### 2. Database Anti-Patterns

#### 2a. Elasticsearch as Primary Database -- Why It Fails

Elasticsearch was built as a search engine, not a database. Using it as a primary data store is one of the most common and costly anti-patterns in dashboard projects. Even Elastic's own documentation recommends keeping a separate source of truth.

**Why it breaks:**

| Problem | Detail | Impact on Sentinel |
|---|---|---|
| **No ACID transactions** | Atomicity is per-document only. A bulk insert that partially fails leaves data in an inconsistent state. No rollback. | Conflict events with related entities (actors, locations) cannot be atomically written. |
| **Eventual consistency** | Near-real-time indexing means queries can return stale or partial results. It is an AP system (CAP theorem), not CP. | Dashboard could show incorrect event counts or miss recent events. |
| **Data loss risk** | No transaction boundaries means failures can leave half-applied operations. Recovery will not roll them back. | Losing conflict event data silently is unacceptable for an intelligence platform. |
| **Schema changes require full reindex** | Changing a field type requires rebuilding the entire index from the source. | As Sentinel's data model evolves, schema changes become extremely painful. |
| **Deep pagination is expensive** | Distributed architecture makes paginating beyond ~10,000 results memory-intensive. Default limit exists. | Analysts querying historical conflict data will hit this wall. |
| **Operational complexity** | Already resource-heavy as a search engine; using it as primary store magnifies this. | More infrastructure to manage with no database guarantees in return. |

**The correct pattern:** Use PostgreSQL (with PostGIS and TimescaleDB) as the source of truth. Use Elasticsearch as a secondary search index, synced via a Change Data Capture (CDC) pipeline or periodic bulk indexing. This gives you ACID guarantees for writes and fast full-text search for reads.

**Anti-pattern to avoid:** "Let's just put everything in Elasticsearch since we need search anyway." This collapses your source of truth and your search index into one fragile system.

#### 2b. PostgreSQL Mistakes

Even with PostgreSQL as the primary store, common mistakes include:

1. **Missing indexes on high-query columns.** Conflict events queried by date range, location, and event type need composite indexes. Without them, queries degrade from milliseconds to seconds at scale.
2. **No table partitioning for time-series data.** At 10M+ events, unpartitioned tables become unwieldy. Use TimescaleDB hypertables or native PostgreSQL range partitioning on timestamp columns.
3. **N+1 query patterns.** Loading a list of events and then querying each event's related entities individually generates O(N) queries instead of O(1). Use JOINs or batch loading.
4. **Storing raw HTML/XML in the database.** RSS feed content stored as raw XML bloats the database, makes indexing impossible, and forces parsing at query time. Extract and store structured fields during ingestion.
5. **Not planning for data growth.** At 1M events: PostgreSQL handles it fine with basic indexing. At 10M events: you need partitioning and query optimization. At 100M events: you need archival strategies, materialized views, and possibly read replicas.

#### 2c. Supabase Scaling Pitfalls (Detailed in Section 6 below)

---

### 3. RSS/Feed Ingestion Anti-Patterns

RSS feed ingestion is deceptively simple. Teams consistently make the same mistakes.

#### Anti-Pattern 1: Polling Too Aggressively

**The mistake:** Setting a 1-minute polling interval for feeds that update once a day.

**What happens:** You get IP-banned, rate-limited, or flagged as abusive. Some feeds use bot-blocking technology that will reject frequent automated requests entirely.

**The fix:**
- Respect `Cache-Control`, `ETag`, and `Last-Modified` headers. Return `If-None-Match` / `If-Modified-Since` in subsequent requests. Handle `304 Not Modified` responses.
- Set per-feed polling intervals based on observed update frequency. A feed that updates hourly should be polled every 30-60 minutes, not every minute.
- Use WebSub (formerly PubSubHubbub) for feeds that support it -- this is push-based and eliminates polling entirely.

#### Anti-Pattern 2: No Exponential Backoff

**The mistake:** When a feed returns an error, immediately retrying at the same interval (or faster).

**What happens:** The "thundering herd" effect. If 50 feeds go down simultaneously (e.g., a CDN outage) and your system retries all 50 every 10 seconds, you create artificial traffic spikes that can compound the outage or get you permanently blocked.

**The fix:**
- Implement exponential backoff with jitter: `delay = min(base * 2^attempt + random_jitter, max_delay)`
- Track consecutive failures per feed. After N consecutive failures (e.g., 5), reduce polling frequency dramatically or suspend the feed and alert.
- Never retry all failed feeds at the same time. Stagger retries.

#### Anti-Pattern 3: Assuming Feeds Are Well-Formed XML

**The mistake:** Using a strict XML parser and crashing on malformed feeds.

**What happens:** Real-world RSS feeds are a mess. Feeds from conflict zones, small news agencies, and government sites routinely have encoding errors, unclosed tags, invalid characters, and mixed RSS/Atom formats. A strict parser will reject a significant percentage of the feeds Sentinel needs.

**The fix:**
- Use the `feedparser` library (Python). It handles RSS 0.9x, 1.0, 2.0, Atom, CDF, and JSON Feed. It gracefully handles malformed XML, encoding issues, and format inconsistencies. Do NOT build a custom parser.
- Wrap all feed parsing in try/except blocks. Log malformed feeds for investigation but do not crash the ingestion pipeline.
- Normalize all feed entries into a common internal schema during ingestion, regardless of source format.

#### Anti-Pattern 4: Storing Everything

**The mistake:** Storing the full raw feed XML, all metadata, all enclosures, and every field "just in case."

**What happens:** Database bloat. A single RSS feed entry with full HTML content can be 10-50KB. At 100 feeds polled hourly with 20 items each, that is 50-250MB/day of raw data, most of which is never queried.

**The fix:**
- Extract and store only the fields you need: title, link, published date, summary (plain text, not HTML), source, and a content hash for deduplication.
- Store a hash of each entry's unique identifier (usually `<link>` or `<guid>`) to detect duplicates before inserting.
- If you need the raw content for NLP later, store it in a separate blob store or cold storage, not in your primary event table.

#### Anti-Pattern 5: No Deduplication

**The mistake:** Inserting every feed item every time you poll, without checking for duplicates.

**What happens:** The same event appears dozens of times in your database. Analysts see duplicate events on the dashboard. Counts are inflated. Search results are polluted.

**The fix:**
- Use a combination of `feed_url + item_guid` (or `item_link` if no GUID) as a unique constraint.
- Check against a Bloom filter or Redis set for fast deduplication before hitting the database.
- Handle the case where a feed item's content changes but its GUID stays the same (update, don't insert).

---

### 4. Frontend Dashboard Anti-Patterns

#### Anti-Pattern 1: Rendering 10,000+ Map Markers Without Clustering

**The mistake:** Passing all conflict events as individual markers to Mapbox GL JS or Leaflet.

**What happens:** The browser freezes. Each marker is a DOM element or WebGL draw call. At 10,000 markers, frame rates drop below 10fps. At 50,000, the tab crashes.

**The fix:**
- Use Supercluster (for Mapbox GL JS) or marker clustering plugins. Group nearby markers at each zoom level.
- Use Deck.gl's ScatterplotLayer for WebGL-accelerated rendering if you need to show individual points at high density.
- Implement viewport-based loading: only fetch and render events within the current map bounds plus a buffer zone.
- Use server-side clustering for initial loads: send pre-aggregated cluster data from the API, not individual events.

#### Anti-Pattern 2: WebSocket Connections That Never Reconnect

**The mistake:** Opening a WebSocket connection for live event updates and assuming it stays open.

**What happens:** WebSocket connections die silently. Browser tabs in the background have their JavaScript timers throttled, preventing heartbeat pings. The server drops the connection after missing heartbeats. The client never knows. The dashboard shows stale data indefinitely.

**Real-world data:** Supabase Realtime subscriptions that "dropped every 30 minutes" were a documented production issue. Firefox had bugs where WebSocket connections leaked ~100MB per 10 minutes due to un-garbage-collected blobs.

**The fix:**
- Implement reconnection with exponential backoff and jitter. Track `readyState` and fire reconnections on `close` and `error` events.
- Use Web Workers to manage WebSocket connections. This isolates them from the main thread's timer throttling and prevents orphaned event listeners from leaking memory on the main thread.
- Implement a heartbeat mechanism independent of the WebSocket protocol's built-in ping/pong. Send application-level heartbeats every 30 seconds. If no response within 10 seconds, assume disconnection and reconnect.
- On reconnection, request a state sync from the server (events since last received timestamp) to fill gaps.

#### Anti-Pattern 3: Memory Leaks from Real-Time Updates

**The mistake:** Appending every incoming event to a React state array without bounds.

**What happens:** After hours of operation, the browser tab consumes gigabytes of memory. The dashboard becomes unresponsive. Garbage collection pauses cause visible jank.

**Real-world data:** WebSocket server tests showed that 20,000 connections consumed ~2.86GB. After closing and reconnecting, memory grew to ~4.14GB due to un-freed resources -- a classic memory leak pattern.

**The fix:**
- Maintain a bounded buffer for real-time events (e.g., last 500 events in memory). Older events are available via paginated API calls, not in-memory state.
- Use `useEffect` cleanup functions in React to properly close WebSocket connections and remove event listeners when components unmount.
- Implement periodic state pruning: every N minutes, trim in-memory collections to their maximum allowed size.
- Profile memory usage during development using Chrome DevTools' Memory tab. Take heap snapshots before and after extended operation.

#### Anti-Pattern 4: Loading All Data on Initial Page Load

**The mistake:** Fetching the entire event database on dashboard load to "have everything ready."

**What happens:** 30-second initial load times. Massive payload sizes. Users leave before the dashboard renders.

**The fix:**
- Load only the current viewport's data. Use time-range filters and geospatial bounding boxes in API queries.
- Implement progressive loading: show the map and skeleton UI immediately, then load data in chunks.
- Use React Query (TanStack Query) for caching, deduplication, and background refetching. Never manually manage API response caching.

#### Anti-Pattern 5: Using D3.js for Everything

**The mistake:** Building every chart, graph, and visualization from scratch with D3.js because "it's more customizable."

**What happens:** Weeks of development time for charts that Recharts or Tremor can render in 10 lines of code. The D3 code is brittle, hard to maintain, and has subtle rendering bugs.

**The fix:**
- Use Recharts or Tremor for standard charts (bar, line, area, pie). These cover 90% of dashboard visualization needs.
- Reserve D3.js for truly custom visualizations that no library supports (e.g., custom force-directed entity graphs, novel timeline visualizations).
- If using D3 with React, use a library like `@visx/visx` that provides React-compatible D3 primitives instead of fighting the DOM.

---

### 5. Architecture Anti-Patterns

#### Anti-Pattern 1: Microservices for a Small Team

**The mistake:** Splitting Sentinel into 7+ microservices (ingestion service, NLP service, geo service, auth service, API gateway, notification service, etc.) from day one.

**What happens:** A 2024 DZone study found teams spent 35% more time debugging in microservices architectures compared to modular monoliths. A modular monolith requires 1-2 ops engineers; equivalent microservices require 2-4 platform engineers plus distributed ops burden.

**Real-world war story:** A fintech startup with 6 developers built a microservice architecture with Kafka at the core. Months were spent fine-tuning brokers, partitioning strategies, and event schemas. By launch, they had a bulletproof pipeline -- but no users. When they pivoted, the system was so coupled to Kafka that rewriting took more time than building from scratch.

**Another war story:** A developer built a web app for a local bakery and split it into seven microservices after reading a book about the pattern. Standard tracking of orders, inventory, and email receipts did not warrant this complexity.

**The industry consensus (2024-2025):** Start with a monolith. Martin Fowler: "Almost all the successful microservice stories have started with a monolith that got too big and was broken up." Fewer than 5% of applications truly benefit from microservices initially. Basecamp has run as a Rails monolith for nearly two decades. Shopify uses a modular monolith, extracting microservices only for specific needs like checkout and fraud detection.

**The fix for Sentinel:**
- Build a **modular monolith** in FastAPI. Use Python modules/packages with clear boundaries (ingestion, analytics, API, etc.) but deploy as a single application.
- Extract services ONLY when a specific module has demonstrably different scaling needs (e.g., the NLP pipeline needs GPU resources that the REST API does not).
- Use clear internal interfaces between modules so extraction is possible later without rewriting.

#### Anti-Pattern 2: Kafka When Redis Streams Would Suffice

**The mistake:** Adding Apache Kafka to handle event streaming between ingestion and processing.

**What happens:** Kafka requires a minimum of 3 brokers for production reliability. It demands ZooKeeper (or KRaft). It needs careful partition strategy, consumer group management, retention configuration, and monitoring. For a dashboard that processes thousands of events per day (not millions per second), this is orders of magnitude more infrastructure than needed.

**War story:** An enterprise team added Kafka to stream data between two internal tools used once daily. The setup required 3 brokers, had a 7-day retention period, and generated constant consumer lag alerts. A single cron job dumping to a database would have sufficed.

**The fix for Sentinel:**
- Use **Redis Streams** for the ingestion pipeline. Redis is already in the stack (for caching). Redis Streams provides consumer groups, acknowledgment, and persistence -- everything Sentinel needs for event routing.
- If Redis Streams becomes a bottleneck (unlikely at Sentinel's scale), migrate to Kafka then. The consumer group abstraction is similar enough that migration is straightforward.
- For simple background job processing (feed polling, NLP analysis), use **Celery with Redis broker**. This is already in the planned tech stack and handles the use case without additional infrastructure.

#### Anti-Pattern 3: GraphQL When REST Is Simpler

**The mistake:** Implementing a GraphQL API because "it's more flexible" for dashboard queries.

**What happens:** GraphQL adds complexity to the backend (resolvers, schema definition, N+1 query prevention with DataLoader), requires a different caching strategy (no HTTP caching by default), and makes rate limiting harder (one "query" can fetch arbitrary amounts of data). For a dashboard with a known, stable set of views, GraphQL's flexibility is unnecessary.

**The fix for Sentinel:**
- Use **REST endpoints** designed for each dashboard view. The dashboard has a finite number of views (map view, timeline view, entity view, etc.). Each view needs 1-3 API calls. REST handles this cleanly.
- If the dashboard later needs ad-hoc querying (e.g., analyst-defined custom queries), consider GraphQL as an addition, not a replacement.

#### Anti-Pattern 4: Building Custom Authentication

**The mistake:** Building a custom user authentication system with password hashing, session management, JWT handling, password reset flows, and email verification.

**What happens:** Security vulnerabilities. Authentication is one of the hardest things to get right, and custom implementations routinely have timing attacks, session fixation, CSRF vulnerabilities, or weak token generation. Even experienced teams get it wrong.

**The fix for Sentinel:**
- Use an existing auth solution. Options: Auth.js (NextAuth) for the Next.js frontend, or FastAPI-Users / python-jose for the backend. If using Supabase, use Supabase Auth (GoTrue).
- Do not roll your own JWT implementation. Do not roll your own password hashing. Do not roll your own session management.

#### Anti-Pattern 5: Premature Optimization

**The mistake:** Spending weeks optimizing database queries, caching strategies, and rendering performance before the dashboard has any users.

**What happens:** You optimize for hypothetical workloads that may not match reality. The optimizations add complexity that slows down feature development. You ship later, with fewer features, and the optimizations may not even address the actual bottlenecks.

**War story:** A team spent eight months building a "future-proof" CMS with seventeen abstraction layers. By launch, their competitor had shipped three major feature updates and captured the market.

**The fix for Sentinel:**
- Build the simplest working version first. Measure. Then optimize the actual bottlenecks.
- The warning signs you have gone too far: (1) you are solving problems you do not have yet, (2) you cannot explain the architecture to a new team member in under 20 minutes, (3) architecture discussion dominates product discussion.

---

### 6. Supabase-Specific Gotchas

Supabase is in the Sentinel tech stack conversation, so understanding its real-world failure modes is critical.

#### Gotcha 1: Real-Time Subscription Instability

**The problem:** Supabase Realtime connections drop silently when browser tabs are backgrounded (JavaScript timer throttling prevents heartbeat pings). Mobile apps lose connections when the screen locks. One developer reported switching to Pusher because of this. As recently as February 5, 2026, Supabase reported elevated errors and latency in their Realtime service.

**The mitigation:** Use the `worker: true` option to run heartbeats in a Web Worker immune to timer throttling. Implement `heartbeatCallback` as a fallback. Build application-level reconnection logic -- do not rely solely on the Supabase client's built-in reconnection.

#### Gotcha 2: Connection Limits Hit Faster Than Expected

**The problem:** A developer's project crashed at 5,000 users -- queries took 8+ seconds, connection limits were hit, and real-time subscriptions dropped. Supavisor (the connection pooler) has hard-coded limits tied to compute size. The "Max client connections reached" error is common during traffic surges.

**The mitigation:** Use connection pooling from the start (Supavisor in transaction mode). Set up monitoring for connection counts and query duration. Be prepared to upgrade compute tiers -- the $25/month "Pro" plan is marketing; real production apps cost $125+/month.

#### Gotcha 3: No Built-In API Rate Limiting

**The problem:** PostgREST (Supabase's API layer) has no built-in rate limiting. Row-Level Security (RLS) prevents data leakage but does not prevent a malicious or buggy client from hammering the database with unlimited queries. Some developers consider this a blocker for production use from the frontend.

**The mitigation:** Implement rate limiting at the application level (e.g., in your FastAPI middleware) or use an API gateway in front of Supabase.

#### Gotcha 4: Cost Surprises at Scale

**The problem:** The Pro plan advertises $25/month but real production costs include:
- Compute upgrades for connection limits and consistent performance
- MAU overages at $3.25 per 1,000 users ($325/month at 200K MAU)
- Storage and egress costs that grow with data volume

**The mitigation:** Budget for $100-300/month for a real production deployment. Monitor usage dashboards. Set billing alerts.

#### Gotcha 5: Self-Hosting Is Harder Than It Looks

**The problem:** Self-hosting Supabase means maintaining PostgreSQL, RealtimeDB, GoTrue auth, Storage, and Vector services. A typical self-hosted team spends 1-2 FTE on operations ($120K-$240K/year). Feature parity with hosted Supabase lags behind. For teams under 50 people, the ROI on self-hosting is poor.

**The recommendation for Sentinel:** Use managed Supabase for rapid prototyping and MVP. Plan a migration path to self-hosted PostgreSQL (with PostGIS + TimescaleDB) if Sentinel grows beyond Supabase's performance or cost ceiling. Since Supabase uses standard PostgreSQL, migration is straightforward via `pg_dump`.

---

## Comparison Tables

### Architecture Approach Comparison

| Criteria | Microservices | Modular Monolith | Simple Monolith | Recommendation for Sentinel |
|---|---|---|---|---|
| **Team size needed** | 10+ developers | 2-5 developers | 1-3 developers | Modular Monolith |
| **Ops complexity** | Very High (2-4 platform engineers) | Low (1-2 engineers) | Minimal | Modular Monolith |
| **Debug time** | 35% more than monolith (DZone 2024) | Baseline | Baseline | Modular Monolith |
| **Deployment complexity** | High (orchestration needed) | Low (single deploy) | Low (single deploy) | Modular Monolith |
| **Scaling flexibility** | High (per-service scaling) | Medium (can extract services) | Low (scale entire app) | Modular Monolith |
| **Time to first feature** | Slowest | Fast | Fastest | Modular Monolith |

### Message Queue Comparison for Sentinel's Scale

| Criteria | Apache Kafka | Redis Streams | Celery + Redis | Recommendation |
|---|---|---|---|---|
| **Min infrastructure** | 3 brokers + ZooKeeper/KRaft | 1 Redis instance (already in stack) | 1 Redis instance (already in stack) | Celery + Redis |
| **Throughput** | Millions/sec | Hundreds of thousands/sec | Thousands/sec | Celery + Redis (sufficient) |
| **Ops complexity** | Very High | Low | Low | Celery + Redis |
| **Consumer groups** | Yes | Yes | Via task routing | Celery + Redis |
| **Persistence** | Yes (configurable retention) | Yes (with AOF) | Via result backend | Celery + Redis |
| **When to upgrade** | >100K events/sec sustained | >10K events/sec sustained | -- | Only if proven necessary |

### Database Role Comparison

| Role | PostgreSQL + PostGIS + TimescaleDB | Elasticsearch | Supabase (Managed PG) | Recommendation |
|---|---|---|---|---|
| **Primary data store** | Excellent (ACID, transactions) | Dangerous (no transactions, data loss risk) | Good (is PostgreSQL) | PostgreSQL |
| **Full-text search** | Adequate (tsvector) | Excellent | Adequate (via PG) | Elasticsearch as secondary index |
| **Geospatial queries** | Excellent (PostGIS) | Limited | Good (PostGIS available) | PostgreSQL with PostGIS |
| **Time-series data** | Excellent (TimescaleDB) | Adequate | Limited extension support | PostgreSQL with TimescaleDB |
| **Schema evolution** | Alembic migrations | Full reindex required | Alembic migrations | PostgreSQL |
| **Operational complexity** | Medium | High | Low (managed) | Depends on scale |

---

## Priority Implementation Order

1. **Set up a modular monolith structure in FastAPI from day one.** Define clear module boundaries (ingestion, analytics, API, auth) with explicit interfaces. This costs zero extra time and saves massive refactoring later. Do NOT create separate services or repositories.

2. **Implement feed ingestion with `feedparser` + exponential backoff + deduplication before adding any other data sources.** RSS feeds are the fastest path to having real data in the system. Get this right first. Use `feedparser` (do not build a custom parser). Implement conditional requests (`ETag`/`Last-Modified`). Add deduplication via content hashing.

3. **Use PostgreSQL as the sole database initially.** Add PostGIS for geospatial, TimescaleDB for time-series. Do NOT add Elasticsearch until you have a working system with real data and can measure actual search performance. PostgreSQL's built-in `tsvector` full-text search may be sufficient for the MVP.

4. **Use Celery + Redis for all background tasks.** Feed polling, NLP processing, geocoding -- all through Celery. Do NOT add Kafka, RabbitMQ, or any other message broker. Redis is already in the stack for caching.

5. **Implement WebSocket reconnection with exponential backoff and Web Workers from the start.** This is not premature optimization -- it is a known reliability requirement. Every real-time dashboard that skips this fails in production.

6. **Add Elasticsearch only after the PostgreSQL-based system is working and search performance is measured.** Set it up as a secondary index synced from PostgreSQL, never as a primary store.

7. **Use an existing auth library (Auth.js or FastAPI-Users).** Do not build custom authentication. This is a solved problem.

8. **Implement map marker clustering from the first map render.** Again, not premature optimization -- rendering thousands of markers without clustering is a guaranteed crash. Use Supercluster or Deck.gl.

---

## Cost Analysis

### Self-Hosted PostgreSQL Stack

| Component | Free/OSS | Managed Alternative | Managed Cost |
|---|---|---|---|
| PostgreSQL | Free | AWS RDS / Supabase Pro | $25-200/month |
| PostGIS extension | Free | Included in managed PG | Included |
| TimescaleDB extension | Free (Apache 2.0 edition) | Timescale Cloud | $29-450/month |
| Redis | Free | AWS ElastiCache / Upstash | $0-50/month |
| Elasticsearch | Free (OSS) | Elastic Cloud / Bonsai | $95-500/month |
| Total (self-hosted) | **$0** (+ server costs) | -- | -- |
| Total (managed) | -- | -- | **$150-1,200/month** |

### Supabase Managed Costs at Scale

| Scale | Estimated Monthly Cost | Notes |
|---|---|---|
| MVP / Development | $0 (free tier) | 500MB database, 50K MAU |
| Small production (1K users) | $25 | Pro plan base |
| Medium production (10K users) | $75-150 | Compute upgrades needed |
| Large production (50K users) | $200-500 | MAU overages, storage, egress |
| Scale production (200K users) | $500-1,000+ | $325/month MAU overages alone |

### Infrastructure Comparison: Simple vs Overengineered

| Approach | Monthly Infrastructure Cost | Engineering Time (First 3 Months) | Recommendation |
|---|---|---|---|
| **Simple:** Monolith + PG + Redis on single VPS | $20-50/month | 80% on features | This one |
| **Medium:** Monolith + PG + Redis + Elasticsearch on 2-3 VPS | $60-150/month | 70% on features | After MVP |
| **Overengineered:** Microservices + Kafka + PG + ES + K8s | $300-1,000/month | 35% on features, 65% on infra | Never (at this team size) |

---

## Open Questions

1. **What is the actual event volume from target data sources?** The choice between Redis Streams and Celery (or needing Kafka) depends on actual throughput. ACLED releases data weekly (~50K events/year). GDELT generates ~250K events/day. The architecture should be validated against real ingestion rates. **Next step:** Run a 1-week pilot ingestion from top 5 sources and measure actual event volumes.

2. **How frequently do ACLED, GDELT, and ReliefWeb APIs change their schemas or rate limits?** This determines how much effort to invest in adapter layers. **Next step:** Check API changelogs and community forums for each source. Subscribe to their developer mailing lists.

3. **Is PostgreSQL `tsvector` full-text search sufficient for Sentinel's search needs, or will Elasticsearch be required from the start?** This depends on the types of queries analysts will run and acceptable latency. **Next step:** Benchmark PostgreSQL full-text search against a sample dataset of 100K conflict events with realistic queries before adding Elasticsearch.

4. **What is the realistic user count for Sentinel in the first 6 months?** This determines whether Supabase managed hosting or self-hosted PostgreSQL is more cost-effective. If under 1,000 users, Supabase is cheaper. If over 5,000, self-hosted may be necessary anyway. **Next step:** Define target audience and estimate user count.

5. **How will Sentinel handle the "data quality problem" from heterogeneous sources?** Different sources have different schema, different definitions of "conflict event," different geolocation precision, and different update frequencies. The ingestion pipeline needs a normalization strategy. **Next step:** Define a canonical event schema and build source-specific normalizers during Phase 2.

6. **What happens when a critical data source goes permanently offline or paywalled?** ACLED moved to a freemium model. GDELT is free but could change. Sentinel needs a data source redundancy strategy. **Next step:** Identify backup sources for each primary source. Ensure no single source is a single point of failure.

---

## Sources & References

### Academic Research
- Coelho & Valente, "Why Modern Open Source Projects Fail" (ESEC/FSE 2017): https://arxiv.org/abs/1707.02327
- OpenPledge, "From Thriving to Forgotten: Dynamics of Abandoned OSS Projects": https://openpledge.io/abandoned-open-source-projects.html
- Handsontable, "Most Common Causes of Failed Open-Source Projects": https://handsontable.com/blog/the-most-common-causes-of-failed-open-source-software-projects

### Elasticsearch Anti-Patterns
- Bonsai, "Why Elasticsearch Should Not Be Your Primary Data Store": https://bonsai.io/blog/why-elasticsearch-should-not-be-your-primary-data-store/
- ParadeDB, "Elasticsearch Was Never a Database": https://www.paradedb.com/blog/elasticsearch-was-never-a-database
- BigData Boutique, "Using Elasticsearch as Your Primary Datastore": https://bigdataboutique.com/blog/using-elasticsearch-or-opensearch-as-your-primary-datastore-1e5178

### Supabase Issues & Documentation
- Supabase Realtime Limits: https://supabase.com/docs/guides/realtime/limits
- Supabase Realtime Troubleshooting: https://supabase.com/docs/guides/realtime/troubleshooting
- Supabase Silent Disconnections: https://supabase.com/docs/guides/troubleshooting/realtime-handling-silent-disconnections-in-backgrounded-applications-592794
- Supabase Realtime Reconnection Issues: https://github.com/supabase/realtime-js/issues/463
- Supabase Connection Scaling Discussion: https://github.com/orgs/supabase/discussions/5975
- Supabase Self-Hosting Discussion: https://github.com/orgs/supabase/discussions/39820

### Supabase vs Self-Hosted PostgreSQL
- Leanware, "Supabase vs Postgres Deployment Guide": https://www.leanware.co/insights/postgresql-vs-supabase-deployment-guide-startups
- Vela/Simplyblock, "Self-hosting Supabase: Is It Worth It?": https://vela.simplyblock.io/articles/self-hosting-supabase-worth-it/
- PeerDB, "Comparing Postgres Managed Services": https://blog.peerdb.io/comparing-postgres-managed-services-aws-azure-gcp-and-supabase

### RSS Feed Best Practices
- Kevin Cox, "RSS Feed Best Practices": https://kevincox.ca/2022/05/06/rss-feed-best-practices/
- Gravitee, "API Rate Limiting at Scale": https://www.gravitee.io/blog/rate-limiting-apis-scale-patterns-strategies
- MoldStud, "API Rate Limiting in RSS Feed Management": https://moldstud.com/articles/p-api-rate-limiting-in-rss-feed-management-for-developers

### WebSocket & Memory Leak Issues
- GitHub websockets/ws Memory Leak Issue #804: https://github.com/websockets/ws/issues/804
- xjavascript.com, "Optimize WebSocket Connections Using Web Workers": https://www.xjavascript.com/blog/run-websocket-in-web-worker-or-service-worker-javascript/
- OneUptime, "How to Fix WebSocket Performance Issues": https://oneuptime.com/blog/post/2026-01-24-websocket-performance/view
- Mozilla Bug 1153907 (WebSocket Memory Leak): https://bugzilla.mozilla.org/show_bug.cgi?id=1153907

### Architecture & Overengineering
- AlgoMaster, "Why You Should NEVER Start With Microservices": https://blog.algomaster.io/p/why-you-should-never-start-with-microservices
- DEV Community, "The Microservices Backlash": https://dev.to/aryanmehrotra/the-microservices-backlash-over-engineering-or-misunderstood-architecture-51bm
- Medium, "Kafka Is Overkill for 80% of Projects": https://medium.com/@techInFocus/kafka-is-overkill-for-80-of-projects-prove-me-wrong-0d966988b58d
- Hemaks, "Why Overengineering Can Sometimes Be the Right Choice": https://hemaks.org/posts/why-overengineering-can-sometimes-be-the-right-choice/
- Java Code Geeks, "Microservices vs Modular Monoliths in 2025": https://www.javacodegeeks.com/2025/12/microservices-vs-modular-monoliths-in-2025-when-each-approach-wins.html
- Graphite, "Microservices vs Monolith Pros Cons Best Practices": https://graphite.dev/guides/microservices-vs-monolith
