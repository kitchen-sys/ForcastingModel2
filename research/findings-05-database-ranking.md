# Findings: Database Architecture for RSS Source Management & Ranking

## Executive Summary

For the Sentinel conflict monitoring dashboard, **self-hosted PostgreSQL with a custom FastAPI backend is the recommended approach** over Supabase. While Supabase offers rapid prototyping with built-in real-time subscriptions and auth, it introduces scaling bottlenecks (single-threaded Postgres Changes, connection limits, vendor lock-in) and cost unpredictability that conflict with a production intelligence platform. The project already specifies FastAPI + PostgreSQL + PostGIS + TimescaleDB in its tech stack, which Supabase cannot fully support. For RSS source ranking, a weighted composite scoring system modeled after NewsGuard's 0-100 methodology -- combining feed reliability, content relevance, source authority, and freshness metrics -- provides the best framework. pgvector for semantic search is viable on self-hosted PostgreSQL up to ~50 million vectors and should be integrated from the start.

## Research Scope

This research responds to Brief 05, covering:
1. Supabase vs custom PostgreSQL for the Sentinel platform
2. RSS source ranking and reliability scoring systems
3. Database schema design for source management and health tracking
4. Data storage architecture for ingested conflict events
5. pgvector for semantic search capabilities and practical limits

---

## Detailed Findings

### 1. Supabase: Capabilities, Pricing, and Limits

#### Free Tier (2025-2026)

| Resource | Free Tier Limit |
|----------|----------------|
| Projects | 2 |
| Database Storage | 500 MB |
| Database Egress | 2 GB |
| Monthly Active Users (Auth) | 50,000 |
| File Storage | 1 GB |
| Storage Egress | 2 GB |
| Edge Function Invocations | 500,000/month |
| Row Limit | **None** (bounded by 500 MB storage) |
| Project Pausing | After 7 days of inactivity |

**Critical limitation**: Free tier projects are automatically paused after 7 days of inactivity. This makes the free tier completely unsuitable for a 24/7 conflict monitoring dashboard that must run continuously.

#### Paid Plans

| Plan | Monthly Cost | Database Storage | Notes |
|------|-------------|-----------------|-------|
| Pro | $25+ | 8 GB | Usage-based overages |
| Team | $599+ | Pro limits + collaboration | Team features |
| Enterprise | Custom | Custom | Dedicated support, HIPAA |

#### Real-Time Subscriptions Limits

Supabase Realtime is built with Elixir/Phoenix and uses PostgreSQL logical replication to broadcast changes via WebSockets. Key limits:

- **Single-thread bottleneck**: Postgres Changes are processed on a single thread to maintain change order. Compute upgrades do not significantly improve throughput for change subscriptions.
- **Channel limit**: Up to 100 channels per connection on most plans.
- **Scaling math is brutal**: If 100 users subscribe to a table and 1 insert occurs, it triggers 100 "reads" (one per user for RLS authorization). This multiplicative effect causes bottlenecks at scale.
- **Pricing**: $2.50 per million messages beyond plan quota. Pro plan includes 5 million messages.
- **Recommendation from Supabase docs**: For scale, use public tables without RLS and use Broadcast instead of Postgres Changes.

#### Production Issues (Community Feedback)

Real-world complaints gathered from GitHub Discussions, Hacker News, Medium, and Reddit:

1. **Backup limitations**: Backups only cover the database, not storage buckets. File deletions are immediate and not recoverable from DB backups.
2. **Branching is dev-only**: Database branching copies schema but not data, making it impractical for staging environments.
3. **Connection saturation**: One developer's project crashed at 5,000 users with 8+ second queries, connection limit hits, and dropped real-time subscriptions.
4. **Migration nightmare**: As of 2025, Supabase no longer allows table definition exports. Manually created tables make migration away from Supabase painful.
5. **Vendor lock-in risk**: Heavy use of Supabase-specific features (database functions, RLS policies, Supabase Auth) creates coupling that makes future migration difficult.
6. **Local dev philosophy mismatch**: Supabase docs assume remote development, which does not provide the isolation needed for team-based development.

### 2. Self-Hosted PostgreSQL: The Case for Custom

#### Why Self-Hosted PostgreSQL Wins for Sentinel

| Factor | Supabase (Hosted) | Self-Hosted PostgreSQL | Winner |
|--------|-------------------|----------------------|--------|
| PostGIS support | Yes (basic) | Full control, latest version | Self-hosted |
| TimescaleDB | Not available | Full support | Self-hosted |
| Real-time | Built-in (single-thread bottleneck) | Custom WebSocket server (full control) | Self-hosted |
| pgvector | Yes | Yes (with pgvectorscale) | Tie |
| Connection pooling | PgBouncer included | PgBouncer or pgcat (configurable) | Self-hosted |
| Cost at scale | $25+/mo growing with usage | ~$50/mo on Hetzner (8 vCPU, 32 GB) | Self-hosted |
| Setup time | Minutes | 2-4 weeks for production-ready | Supabase |
| Ops burden | Managed | Self-managed (or use managed PG) | Supabase |
| Full-text search | Basic | Elasticsearch integration | Self-hosted |
| Backup control | Limited | Full (pg_dump, WAL archiving, PITR) | Self-hosted |

**The decisive factor**: Sentinel's tech stack already requires PostGIS + TimescaleDB + Elasticsearch + Redis + Celery. Supabase cannot provide TimescaleDB, and the project needs a custom FastAPI backend anyway (not PostgREST). Supabase would be an extra dependency without replacing any existing stack component.

#### Real-Time Without Supabase

For self-hosted real-time capabilities, the recommended approach:

1. **PostgreSQL LISTEN/NOTIFY** for database change events (lightweight, built-in).
2. **Redis Pub/Sub or Redis Streams** for cross-service event distribution (already in the stack for Celery).
3. **FastAPI WebSocket endpoints** for client-side push (already planned in the architecture).
4. **Server-Sent Events (SSE)** as a simpler alternative for one-way event streaming.

This stack gives full control over real-time behavior without the single-thread bottleneck of Supabase's Postgres Changes.

### 3. RSS Source Ranking System

#### Scoring Framework: Weighted Composite Score (0-100)

Modeled after NewsGuard's methodology (used for 35,000+ news sources covering 95%+ of online engagement), the Sentinel source ranking system uses a weighted composite score across four dimensions:

##### Dimension 1: Feed Reliability (30 points max)

| Metric | Points | How Measured |
|--------|--------|-------------|
| Uptime percentage (30-day rolling) | 0-10 | Automated health checks every 15 min |
| Average response latency | 0-5 | Track p50, p95, p99 latency |
| Error rate (HTTP 4xx/5xx) | 0-5 | Count errors / total checks |
| SSL/TLS validity | 0-3 | Certificate chain validation |
| Feed format compliance | 0-4 | RSS/Atom spec compliance check |
| Content encoding consistency | 0-3 | Detect encoding issues, broken XML |

##### Dimension 2: Content Quality (30 points max)

| Metric | Points | How Measured |
|--------|--------|-------------|
| Conflict relevance rate | 0-10 | % of items matching conflict keywords/entities |
| Entity richness | 0-5 | Avg named entities per item (locations, actors, events) |
| Duplication rate | 0-5 | Dedup against other sources using MinHash/SimHash |
| Content freshness | 0-5 | Avg time from event to publication |
| Geographic specificity | 0-5 | % of items with extractable geolocation |

##### Dimension 3: Source Authority (25 points max)

| Metric | Points | How Measured |
|--------|--------|-------------|
| Organization type | 0-8 | Wire service=8, Major outlet=6, Regional=4, Blog=1 |
| Editorial standards | 0-5 | Manual assessment (corrections policy, bylines, sourcing) |
| Historical accuracy | 0-5 | Track claim verification over time |
| Cross-reference rate | 0-4 | How often this source's events appear in other sources |
| Domain authority/age | 0-3 | Domain registration age + web authority metrics |

##### Dimension 4: Operational Value (15 points max)

| Metric | Points | How Measured |
|--------|--------|-------------|
| Update frequency | 0-5 | Items/day matching expected cadence |
| Geographic coverage breadth | 0-3 | Number of distinct regions covered |
| Unique content rate | 0-4 | % of items not found in higher-ranked sources |
| API/feed accessibility | 0-3 | Rate limiting friendliness, structured data availability |

#### Composite Score Calculation

```python
composite_score = (
    reliability_score * 0.30 +    # 30% weight
    content_quality_score * 0.30 + # 30% weight
    authority_score * 0.25 +       # 25% weight
    operational_score * 0.15       # 15% weight
)
```

Weights can be adjusted per use case. For example, a "breaking news" mode might weight freshness and update frequency higher, while an "analysis" mode weights authority and content quality higher.

#### Score Tiers

| Tier | Score Range | Label | Treatment |
|------|-------------|-------|-----------|
| Tier 1 | 80-100 | Gold | Primary sources, highest polling frequency |
| Tier 2 | 60-79 | Silver | Secondary sources, standard polling |
| Tier 3 | 40-59 | Bronze | Supplementary, lower polling frequency |
| Tier 4 | 20-39 | Watch | Monitored but deprioritized |
| Tier 5 | 0-19 | Suspended | Inactive, flagged for review/removal |

#### Time-Decay for Score Freshness

Source scores should incorporate time-decay so that recent performance carries more weight:

```python
def time_weighted_score(recent_score, historical_score, decay_factor=0.7):
    """
    recent_score: computed from last 7 days of data
    historical_score: computed from last 90 days of data
    decay_factor: weight given to recent performance (0.0-1.0)
    """
    return (recent_score * decay_factor) + (historical_score * (1 - decay_factor))
```

### 4. Database Schema Design

#### Core Tables

```sql
-- Sources: Master table for all data sources
CREATE TABLE sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL UNIQUE,
    feed_url TEXT,                           -- RSS/Atom feed URL (if applicable)
    source_type VARCHAR(50) NOT NULL,        -- 'rss', 'api', 'scraper', 'manual'
    category VARCHAR(100),                   -- 'wire_service', 'ngo', 'government', 'academic', 'media'
    region_focus VARCHAR(100)[],             -- Array of regions this source covers
    language VARCHAR(10) DEFAULT 'en',

    -- Scoring fields (denormalized for fast queries)
    composite_score DECIMAL(5,2) DEFAULT 0,
    reliability_score DECIMAL(5,2) DEFAULT 0,
    content_quality_score DECIMAL(5,2) DEFAULT 0,
    authority_score DECIMAL(5,2) DEFAULT 0,
    operational_score DECIMAL(5,2) DEFAULT 0,
    score_tier VARCHAR(20) DEFAULT 'watch',

    -- Operational metadata
    polling_interval_seconds INTEGER DEFAULT 900,  -- 15 min default
    last_checked_at TIMESTAMPTZ,
    last_successful_fetch_at TIMESTAMPTZ,
    last_error_at TIMESTAMPTZ,
    last_error_message TEXT,
    consecutive_errors INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    is_manually_curated BOOLEAN DEFAULT false,

    -- Authority metadata (manually assigned)
    editorial_standards_score DECIMAL(3,1),
    organization_type VARCHAR(50),
    notes TEXT,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sources_composite_score ON sources(composite_score DESC);
CREATE INDEX idx_sources_active ON sources(is_active) WHERE is_active = true;
CREATE INDEX idx_sources_type ON sources(source_type);
CREATE INDEX idx_sources_tier ON sources(score_tier);

-- Feed Health Checks: Time-series of polling results
CREATE TABLE feed_health_checks (
    id BIGSERIAL PRIMARY KEY,
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    checked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Response metrics
    http_status_code INTEGER,
    response_time_ms INTEGER,
    content_length_bytes INTEGER,
    was_successful BOOLEAN NOT NULL,
    error_type VARCHAR(50),        -- 'timeout', 'dns_failure', 'http_error', 'parse_error', 'ssl_error'
    error_detail TEXT,

    -- Content metrics (per check)
    items_found INTEGER DEFAULT 0,
    new_items INTEGER DEFAULT 0,
    duplicate_items INTEGER DEFAULT 0,
    items_with_geodata INTEGER DEFAULT 0,
    avg_item_age_hours DECIMAL(8,2)
);

-- Convert to TimescaleDB hypertable for efficient time-series queries
SELECT create_hypertable('feed_health_checks', 'checked_at');

CREATE INDEX idx_health_source_time ON feed_health_checks(source_id, checked_at DESC);

-- Source Score History: Track score changes over time
CREATE TABLE source_score_history (
    id BIGSERIAL PRIMARY KEY,
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    composite_score DECIMAL(5,2),
    reliability_score DECIMAL(5,2),
    content_quality_score DECIMAL(5,2),
    authority_score DECIMAL(5,2),
    operational_score DECIMAL(5,2),
    score_tier VARCHAR(20),
    calculation_metadata JSONB    -- Store the raw inputs used for scoring
);

SELECT create_hypertable('source_score_history', 'recorded_at');

-- Conflict Events: Normalized events from all sources
CREATE TABLE conflict_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID REFERENCES sources(id),
    external_id VARCHAR(255),               -- ID from original source (ACLED event_id, etc.)

    -- Core event data
    title TEXT NOT NULL,
    description TEXT,
    event_type VARCHAR(100),                -- 'battle', 'protest', 'explosion', 'strategic_development'
    event_date TIMESTAMPTZ,
    published_at TIMESTAMPTZ,
    ingested_at TIMESTAMPTZ DEFAULT NOW(),

    -- Geospatial
    location_name TEXT,
    country VARCHAR(100),
    admin1 VARCHAR(100),                    -- State/province
    admin2 VARCHAR(100),                    -- District/county
    coordinates GEOMETRY(POINT, 4326),      -- PostGIS point
    geo_precision VARCHAR(20),              -- 'exact', 'approximate', 'country_level'

    -- Actors and entities (JSONB for flexibility)
    actors JSONB DEFAULT '[]',              -- [{name, type, side}]
    entities JSONB DEFAULT '[]',            -- Named entities extracted via NLP

    -- Classification
    severity INTEGER CHECK (severity BETWEEN 1 AND 10),
    fatalities INTEGER,
    tags VARCHAR(100)[],

    -- Content
    source_url TEXT,
    raw_content TEXT,
    content_hash VARCHAR(64),               -- SHA-256 for deduplication

    -- Semantic search
    embedding VECTOR(384),                  -- pgvector embedding (384d for efficiency)

    -- Quality tracking
    confidence_score DECIMAL(3,2),          -- 0.00-1.00
    is_verified BOOLEAN DEFAULT false,
    duplicate_of UUID REFERENCES conflict_events(id),

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- TimescaleDB hypertable for time-series queries on events
SELECT create_hypertable('conflict_events', 'ingested_at');

-- Indexes
CREATE INDEX idx_events_source ON conflict_events(source_id);
CREATE INDEX idx_events_type ON conflict_events(event_type);
CREATE INDEX idx_events_date ON conflict_events(event_date DESC);
CREATE INDEX idx_events_country ON conflict_events(country);
CREATE INDEX idx_events_hash ON conflict_events(content_hash);
CREATE INDEX idx_events_geo ON conflict_events USING GIST(coordinates);
CREATE INDEX idx_events_embedding ON conflict_events USING hnsw(embedding vector_cosine_ops);

-- Event-Source Cross-References: Track which sources reported same event
CREATE TABLE event_cross_references (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL REFERENCES conflict_events(id) ON DELETE CASCADE,
    corroborating_source_id UUID NOT NULL REFERENCES sources(id),
    corroborating_event_id UUID REFERENCES conflict_events(id),
    similarity_score DECIMAL(3,2),           -- 0.00-1.00
    match_type VARCHAR(50),                  -- 'exact_duplicate', 'same_event', 'related'
    detected_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_crossref_event ON event_cross_references(event_id);
CREATE INDEX idx_crossref_source ON event_cross_references(corroborating_source_id);
```

#### Key Schema Design Decisions

1. **Denormalized scores on sources table**: Composite and dimension scores are stored directly on the sources table for fast ranked queries. Historical scores go in `source_score_history` for trend analysis.

2. **TimescaleDB hypertables**: Both `feed_health_checks` and `conflict_events` use TimescaleDB hypertables for efficient time-range queries and automatic partitioning. This handles the continuous append-only nature of health checks and event ingestion.

3. **PostGIS for geospatial**: The `coordinates` column uses PostGIS GEOMETRY type with SRID 4326 (WGS84) for spatial indexing and queries like "find all events within 50km of Kyiv."

4. **pgvector with 384 dimensions**: Using 384-dimensional embeddings (e.g., from `all-MiniLM-L6-v2`) instead of 1536 (OpenAI ada-002) provides 200%+ throughput improvement with minimal accuracy loss. HNSW index for approximate nearest neighbor search.

5. **JSONB for flexible entity storage**: Actors and entities use JSONB to accommodate varying structures from different sources without rigid schema constraints.

6. **Content hash for deduplication**: SHA-256 hash of normalized content enables fast duplicate detection across sources.

### 5. pgvector for Semantic Search

#### Practical Limits and Recommendations

| Factor | Detail |
|--------|--------|
| Maximum practical scale | ~50 million vectors with pgvectorscale |
| Recommended dimensions | 384 (all-MiniLM-L6-v2) over 1536 (ada-002) |
| Latency (10M vectors, 768d) | ~68ms on Supabase, ~45ms on Pinecone |
| Throughput ceiling | ~2,000 QPS before needing vertical scaling |
| Index type recommendation | HNSW for <10M vectors, DiskANN (pgvectorscale) for >10M |
| Memory requirement | HNSW index should fit in shared_buffers for best performance |
| Index build time | ~40 min for 10M vectors on mid-tier instance (DiskANN) |

#### For Sentinel Specifically

The conflict events table will likely contain 1-10 million events after 1-2 years of operation. At 384 dimensions, each vector consumes ~1.5 KB. For 10M events, that is ~15 GB of vector data, well within PostgreSQL's comfort zone.

Use cases for semantic search in Sentinel:
- "Find events similar to this one" (event deduplication and clustering)
- "Find reports about militia activity in eastern Congo" (semantic query)
- Cross-language event matching (embed multilingual content in same vector space)

**Recommendation**: Use self-hosted PostgreSQL with pgvector + pgvectorscale. Start with HNSW indexing. Switch to DiskANN if vector count exceeds 10M and memory becomes constrained.

---

## Comparison Tables

### Supabase vs Self-Hosted PostgreSQL for Sentinel

| Criteria | Supabase (Hosted) | Self-Hosted PostgreSQL | Recommendation |
|----------|-------------------|----------------------|----------------|
| Monthly cost (production) | $25-$100+ (scales with usage) | ~$50 (Hetzner 8vCPU/32GB) | **Self-hosted** |
| TimescaleDB support | Not available | Full support | **Self-hosted** |
| PostGIS support | Basic | Full, latest version | **Self-hosted** |
| pgvector support | Yes | Yes (with pgvectorscale) | Tie |
| Real-time capability | Built-in (single-thread bottleneck) | Custom (LISTEN/NOTIFY + WebSockets) | **Self-hosted** |
| Elasticsearch integration | Not native | Full integration | **Self-hosted** |
| Connection management | PgBouncer, managed | Full control (PgBouncer/pgcat) | **Self-hosted** |
| Backup control | Limited, no bucket backup | Full (WAL, PITR, pg_dump) | **Self-hosted** |
| Setup time | Minutes | 2-4 weeks | **Supabase** |
| Auth system | Built-in (GoTrue) | Must implement separately | **Supabase** |
| Ops overhead | Minimal | Significant (or use managed PG) | **Supabase** |
| Vendor lock-in risk | High (migrations difficult) | None | **Self-hosted** |
| FastAPI compatibility | PostgREST (different paradigm) | Native async SQLAlchemy/asyncpg | **Self-hosted** |

**Verdict**: Self-hosted PostgreSQL wins 9-4 for Sentinel's requirements. Supabase only wins on setup speed, managed ops, and built-in auth -- none of which are blockers for a project that already plans custom FastAPI + Docker infrastructure.

### Source Ranking Dimension Comparison

| Ranking Dimension | Weight | Auto-Measurable? | Update Frequency | Data Source |
|-------------------|--------|-------------------|------------------|-------------|
| Feed Reliability | 30% | Yes (fully automated) | Every health check | feed_health_checks table |
| Content Quality | 30% | Mostly (NLP pipeline) | Per ingestion batch | NLP analytics engine |
| Source Authority | 25% | Partially (manual + automated) | Weekly/monthly | Manual curation + cross-ref analysis |
| Operational Value | 15% | Yes (fully automated) | Daily | Ingestion pipeline metrics |

---

## Priority Implementation Order

1. **PostgreSQL + PostGIS + TimescaleDB base setup** -- This is the foundation everything else depends on. Containerize with Docker, set up Alembic migrations, and create the core schema (sources, feed_health_checks, conflict_events tables). Estimated: 2-3 days.

2. **Sources table with basic CRUD API** -- FastAPI endpoints for creating, reading, updating, and managing data sources. Include bulk import capability for initial source seeding (ACLED, GDELT, RSS feeds from the data sources research). Estimated: 1-2 days.

3. **Feed health check system** -- Automated polling of all active sources on their configured intervals. Record HTTP status, latency, error type, and items found into the feed_health_checks TimescaleDB hypertable. This provides the raw data for reliability scoring. Estimated: 2-3 days.

4. **Reliability scoring (Dimension 1)** -- Implement automated calculation of feed reliability scores from health check data: uptime %, average latency, error rate, SSL validity, format compliance. Update source scores on a rolling window. Estimated: 1-2 days.

5. **Content quality scoring (Dimension 2)** -- Integrate with the NLP pipeline to assess conflict relevance, entity richness, duplication rate, freshness, and geographic specificity of ingested content. This depends on the NLP analytics engine being available. Estimated: 2-3 days.

6. **Composite score calculation and tier assignment** -- Combine all four dimensions with configurable weights to produce final 0-100 composite scores. Assign tiers. Use scores to dynamically adjust polling intervals (higher-ranked sources get polled more frequently). Estimated: 1 day.

7. **pgvector integration for semantic search** -- Add vector embedding column to conflict_events, set up embedding generation (using all-MiniLM-L6-v2 or similar), create HNSW index, and build semantic search API endpoints. Estimated: 2-3 days.

8. **Source authority scoring (Dimension 3)** -- Build admin interface for manual authority assessment (organization type, editorial standards). Implement automated cross-reference detection to measure how often a source's events are corroborated by other sources. Estimated: 3-4 days.

9. **Score history and trend analysis** -- Record score snapshots in source_score_history. Build API endpoints for score trend visualization (7-day, 30-day, 90-day trends). Alert on significant score drops. Estimated: 1-2 days.

10. **Event deduplication pipeline** -- Use content hashing (SHA-256) for exact duplicates and pgvector cosine similarity for semantic duplicates. Link duplicate events via event_cross_references table. Estimated: 2-3 days.

---

## Cost Analysis

### Self-Hosted PostgreSQL (Recommended)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| Hetzner VPS (8 vCPU, 32 GB RAM, 240 GB NVMe) | ~$50 | Primary database server |
| Hetzner VPS (4 vCPU, 16 GB RAM) | ~$25 | Redis + Elasticsearch |
| Backups (Hetzner snapshots + S3) | ~$10 | Daily automated backups |
| Domain + SSL (Let's Encrypt) | $0-15 | Free SSL with Let's Encrypt |
| **Total** | **~$85-100/mo** | |

### Supabase (Not Recommended, for comparison)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| Supabase Pro plan | $25 | Base fee |
| Compute upgrade (for real-time + pgvector) | $50-150 | Depends on usage |
| Realtime messages overage | $2.50/M messages | Conflict events could generate millions |
| Storage overages | Variable | Grows with event data |
| Still need separate Elasticsearch | ~$25+ | Supabase lacks full-text search at this level |
| Still need separate Redis/Celery | ~$25+ | Task queue not provided by Supabase |
| Still need separate TimescaleDB | ~$25+ | Not available in Supabase |
| **Total** | **~$175-350+/mo** | More expensive AND less capable |

### Managed PostgreSQL Alternative (Middle Ground)

| Provider | Monthly Cost | Notes |
|----------|-------------|-------|
| DigitalOcean Managed PostgreSQL (4 vCPU, 8 GB) | ~$60 | Includes backups, HA option |
| AWS RDS PostgreSQL (db.t3.medium) | ~$65 | Highest benchmark performance |
| Render Managed PostgreSQL | ~$50 | Simple, developer-friendly |
| Neon Serverless PostgreSQL | $19+ | Serverless scaling, good for variable load |

**Recommendation**: Start with Hetzner VPS for maximum control and lowest cost. Migrate to managed PostgreSQL (DigitalOcean or AWS RDS) if ops burden becomes significant.

---

## Open Questions

1. **TimescaleDB licensing for production** -- TimescaleDB changed its license in recent years. Need to verify current licensing terms for the specific features needed (hypertables, continuous aggregates, compression). The Apache 2.0 licensed features may be sufficient, or the Timescale License may be needed.

2. **Optimal health check interval vs. source load** -- Polling 100+ sources every 15 minutes means ~10,000 checks/day. Is this sustainable for all source types? Some sources (e.g., government sites) may rate-limit aggressive polling. Need to test and potentially implement adaptive polling intervals.

3. **NLP pipeline dependency for content quality scoring** -- Dimension 2 (Content Quality) requires the NLP analytics engine to be operational. What is the fallback scoring mechanism if NLP is not yet available? Suggest using keyword-based proxy scoring initially.

4. **Cross-source event deduplication accuracy** -- Semantic similarity thresholds for event deduplication need empirical tuning. What cosine similarity threshold (0.85? 0.90? 0.95?) correctly identifies same-event reports vs. genuinely different events? This requires testing with real conflict data.

5. **Authority score objectivity** -- Manual authority scoring introduces human bias. How do we ensure consistent scoring across curators? Consider developing a rubric with examples for each score level.

6. **pgvector index rebuild frequency** -- HNSW indexes need to be rebuilt or updated as new vectors are added. What is the performance impact of continuous inserts vs. periodic batch index rebuilds? pgvector 0.5+ supports concurrent index builds, but the impact on query performance during rebuilds needs testing.

7. **Embedding model selection** -- The schema specifies 384 dimensions (all-MiniLM-L6-v2), but multilingual conflict data may benefit from a multilingual model (e.g., paraphrase-multilingual-MiniLM-L12-v2, also 384d). Need to evaluate accuracy on conflict-specific text in multiple languages.

8. **Score gaming prevention** -- If sources discover the ranking algorithm, could they game it (e.g., publishing high-volume low-quality content to inflate operational scores)? Consider implementing anomaly detection on score changes.

---

## Sources & References

### Supabase Pricing & Limits
- [Supabase Pricing Page](https://supabase.com/pricing)
- [Supabase Billing Documentation](https://supabase.com/docs/guides/platform/billing-on-supabase)
- [Supabase Pricing Breakdown (Metacto)](https://www.metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance)
- [Supabase Pricing Analysis (UI Bakery)](https://uibakery.io/blog/supabase-pricing)

### Supabase Real-Time
- [Supabase Realtime Limits Documentation](https://supabase.com/docs/guides/realtime/limits)
- [Supabase Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)
- [Supabase Realtime Pricing](https://supabase.com/docs/guides/realtime/pricing)
- [Supabase Scaling Discussion (GitHub)](https://github.com/orgs/supabase/discussions/14597)

### Supabase Production Issues
- [Is Supabase Truly Production Ready? (GitHub Discussion)](https://github.com/orgs/supabase/discussions/28377)
- [Supabase Scaling Issues (Hacker News)](https://news.ycombinator.com/item?id=38659806)
- [Cannot Fully Recommend Supabase (Medium)](https://bombillazo.medium.com/why-i-cannot-fully-recommend-supabase-yet-f8e994201804)
- [3 Biggest Mistakes Using Supabase (Medium)](https://medium.com/@lior_amsalem/3-biggest-mistakes-using-supabase-854fe45712e3)
- [Scale Supabase to 100K+ Users Guide](https://www.princenocode.com/blog/scale-supabase-production-guide)

### PostgreSQL & Self-Hosting
- [Postgres vs Supabase Benchmarks (pgbench.com)](https://pgbench.com/comparisons/postgres-vs-supabase/)
- [Self-Hosting Supabase Analysis (Vela/Simplyblock)](https://vela.simplyblock.io/articles/self-hosting-supabase-worth-it/)
- [Comparing Managed Postgres Services (PeerDB)](https://blog.peerdb.io/comparing-postgres-managed-services-aws-azure-gcp-and-supabase)
- [Supabase vs PostgreSQL Deployment Guide (Leanware)](https://www.leanware.co/insights/postgresql-vs-supabase-deployment-guide-startups)

### pgvector & Semantic Search
- [Supabase pgvector Documentation](https://supabase.com/docs/guides/database/extensions/pgvector)
- [Optimizing Vector Search at Scale (Medium)](https://medium.com/@dikhyantkrishnadalai/optimizing-vector-search-at-scale-lessons-from-pgvector-supabase-performance-tuning-ce4ada4ba2ed)
- [Pinecone vs Supabase pgvector Performance Test 2026](https://geetopadesha.com/vector-search-in-2026-pinecone-vs-supabase-pgvector-performance-test/)
- [Postgres Vector Search Benchmarks (Medium)](https://medium.com/@DataCraft-Innovations/postgres-vector-search-with-pgvector-benchmarks-costs-and-reality-check-f839a4d2b66f)
- [Supabase Vector Module](https://supabase.com/modules/vector)

### Source Credibility & Ranking
- [NewsGuard Rating Process & Criteria](https://www.newsguardtech.com/ratings/rating-process-criteria/)
- [NewsGuard News Reliability Ratings](https://www.newsguardtech.com/solutions/news-reliability-ratings/)
- [Ad Fontes Media Methodology](https://adfontesmedia.com/methodology/)
- [Feed Ranking Algorithms in System Design (JavaTechOnline)](https://javatechonline.com/feed-ranking-algorithms-in-system-design/)
- [Meta News Feed Ranking with ML](https://engineering.fb.com/2021/01/26/ml-applications/news-feed-ranking/)
- [Predicting News Source Credibility (ResearchGate)](https://www.researchgate.net/publication/335082931_Predicting_News_Source_Credibility)
- [Reducing Unreliable News via Algorithms (Nature)](https://www.nature.com/articles/s41598-023-38277-5)

### Feed System Design
- [Designing a Scalable News Feed System (AlgoMaster)](https://blog.algomaster.io/p/designing-a-scalable-news-feed-system)
- [News Feed System Design (Medium)](https://medium.com/@ishwarya1011.hidkimath/system-design-feedback-system-88a67b81a8b3)
