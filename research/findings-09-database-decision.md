# Findings: Database Decision -- Self-Hosted PostgreSQL (Final)

## Executive Summary

**Decision: Self-hosted PostgreSQL using the `timescale/timescaledb-ha` Docker image.** This single image bundles PostgreSQL 17, TimescaleDB, PostGIS, pgvector, and pgvectorscale out of the box. Supabase is rejected for Sentinel because it cannot run TimescaleDB, its real-time system has a single-threaded bottleneck incompatible with high-frequency conflict event ingestion, its PostGIS support requires awkward `rpc()` workarounds, and it would actually cost more than self-hosted once you add the separate Redis, Elasticsearch, and TimescaleDB instances that Supabase cannot provide. This document provides the complete Docker Compose configuration, CREATE TABLE statements for `sources`, `events`, and `rankings`, and a migration escape plan.

## Research Scope

This document responds to Brief 09 and builds on the analysis in findings-05-database-ranking.md. It covers:
1. Final evaluation of Supabase (hosted) for Sentinel's requirements
2. Final evaluation of self-hosted PostgreSQL for Sentinel's requirements
3. Rejection of the hybrid approach as unnecessary complexity
4. Complete Docker Compose configuration for the chosen stack
5. Production-ready CREATE TABLE statements for core tables
6. Migration/escape plan if the decision needs to change

---

## Detailed Findings

### 1. Why Supabase Is Rejected

Supabase is a good product for MVPs, internal tools, and apps that need auth + real-time + storage out of the box quickly. It is not the right choice for Sentinel. Here is the concrete evidence:

#### 1.1 TimescaleDB Is Not Available on Supabase

Sentinel's architecture requires TimescaleDB hypertables for time-series data: feed health checks (thousands per day), conflict events (continuous append-only ingestion), and score history tracking. TimescaleDB provides automatic partitioning, compression, and continuous aggregates that are essential for querying time-range data efficiently. Supabase does not offer TimescaleDB as an extension. This alone is disqualifying.

#### 1.2 Real-Time Single-Thread Bottleneck

Supabase Realtime processes Postgres Changes on a single thread to maintain change order. For a conflict monitoring dashboard ingesting events from dozens of sources simultaneously:

- **Default limits**: 100 events per second per tenant, 100 channels, 200 concurrent users per channel.
- **Quadratic scaling**: If 50 dashboard users subscribe to the events table and 1 insert occurs, that triggers 50 database authorization checks (one per subscriber for RLS). At 10 inserts/second from ACLED + GDELT + RSS feeds, that is 500 authorization queries/second from the real-time system alone.
- **WAL replication lag**: High insert rates cause replication lag in the logical decoding pipeline, delaying real-time delivery.
- **Mitigation requires abandoning the feature**: Supabase recommends using Broadcast instead of Postgres Changes for high-throughput scenarios, but Broadcast does not automatically capture database changes -- you would need to manually publish events, which means building the same LISTEN/NOTIFY + WebSocket pipeline that self-hosted PostgreSQL uses natively.

#### 1.3 PostGIS Is Awkward on Supabase

Supabase supports PostGIS as an extension, but with significant friction:

- **No native client library support**: PostGIS geometry types render as hex blobs (`0101000020E6100000...`) in the Supabase dashboard and client libraries. You must create database functions using `st_astext()`, `st_y()`, `st_x()` to convert them.
- **All geo queries require `rpc()` calls**: You cannot use PostGIS operators like `<->` (nearest neighbor) or `ST_Distance` through the standard Supabase query builder. Every geospatial query must be wrapped in a Postgres function and invoked via `rpc()`.
- **Schema configuration pitfalls**: PostGIS must be created in a separate schema (e.g., `gis`), and geography columns require the schema prefix. Documentation is inconsistent.
- **Self-hosted bugs**: Reported issues where PostGIS functions fail in Supabase self-hosted deployments (GitHub issue #27295).

Since Sentinel is a geospatial intelligence platform where every event has coordinates and spatial queries are core functionality, this friction is unacceptable.

#### 1.4 Free Tier Is Unsuitable for 24/7 Monitoring

- **500 MB database storage**: A single conflict event row with embeddings, JSONB fields, and text content is roughly 2-4 KB. At 500 MB, that is approximately 125,000-250,000 events before hitting the limit. At an ingestion rate of even 1,000 events/day, the free tier is exhausted in 4-8 months.
- **Auto-pause after 7 days of inactivity**: Free tier projects pause after 7 days without activity. A 24/7 conflict monitoring dashboard cannot tolerate pausing.
- **2 GB data transfer**: Dashboard queries, real-time subscriptions, and API calls consume egress. 2 GB/month is trivially exceeded by a multi-user intelligence platform.

#### 1.5 Total Cost Is Higher Than Self-Hosted

Supabase Pro at $25/month does not include Redis, Elasticsearch, TimescaleDB, or Celery. You still need separate infrastructure for all of those:

| Component | Supabase Path | Self-Hosted Path |
|-----------|--------------|-----------------|
| Primary database | $25-150/mo (Supabase Pro + compute) | Included in VPS |
| TimescaleDB | Not available (need separate DB) ~$25+/mo | Included (same PostgreSQL) |
| Redis (cache + Celery broker) | Separate service ~$25/mo | Included in VPS |
| Elasticsearch | Separate service ~$25+/mo | Included in VPS |
| Real-time messages | $2.50/million beyond quota | Free (LISTEN/NOTIFY) |
| **Total** | **$100-250+/mo** | **$50-85/mo** (single VPS) |

Self-hosted is cheaper AND more capable.

### 2. Why Self-Hosted PostgreSQL Wins

#### 2.1 One Image, All Extensions

The `timescale/timescaledb-ha` Docker image includes:
- **PostgreSQL 17** (latest stable)
- **TimescaleDB** (hypertables, continuous aggregates, compression)
- **PostGIS 3** (full geospatial support, GIST indexes, spatial queries)
- **pgvector** (vector similarity search, HNSW indexes)
- **pgvectorscale** (DiskANN for large-scale vector search)
- **pg_cron** (scheduled tasks inside the database)
- **pgaudit** (audit logging)
- **pgrouting** (network analysis)

No need to build custom images or manage extension compatibility. One image, one container, all capabilities.

#### 2.2 Full Real-Time Control

Self-hosted gives us a real-time architecture with no artificial bottlenecks:

1. **PostgreSQL LISTEN/NOTIFY**: Built-in, zero-config pub/sub for database change notifications. No single-thread bottleneck, no WAL replication lag for notifications.
2. **Redis Pub/Sub**: Already in the stack for Celery. Use it to fan out events across multiple FastAPI WebSocket workers.
3. **FastAPI WebSocket endpoints**: Full control over authentication, filtering, rate limiting, and message formatting.
4. **Server-Sent Events (SSE)**: Simpler alternative for one-way streaming to dashboard clients.

This stack handles thousands of inserts/second with real-time delivery to hundreds of concurrent dashboard users, without the quadratic scaling problem of Supabase Realtime.

#### 2.3 Direct FastAPI Integration

Sentinel already specifies FastAPI as the backend framework. Self-hosted PostgreSQL integrates natively:

- **asyncpg**: High-performance async PostgreSQL driver (not going through PostgREST)
- **SQLAlchemy 2.0 async**: ORM with full PostGIS and TimescaleDB support via GeoAlchemy2
- **Alembic migrations**: Full schema version control (Supabase migration tooling is limited)
- **Connection pooling**: PgBouncer or pgcat with full configuration control

No impedance mismatch, no `rpc()` workarounds, no PostgREST abstraction layer.

### 3. Why the Hybrid Approach Is Rejected

A hybrid approach (Supabase for auth + real-time, self-hosted PostgreSQL for heavy data) was considered and rejected:

- **Added complexity**: Two database systems means two connection pools, two sets of credentials, two backup strategies, data synchronization between them, and doubled monitoring.
- **Auth is easy without Supabase**: FastAPI has mature auth libraries (python-jose for JWT, passlib for hashing, or FastAPI-Users as a complete solution). Adding Supabase solely for auth is not worth the dependency.
- **Real-time is better without Supabase**: As discussed above, LISTEN/NOTIFY + Redis + FastAPI WebSockets outperforms Supabase Realtime for this use case.
- **No clear benefit**: Every capability Supabase provides (auth, real-time, storage, auto-generated APIs) has a self-hosted equivalent that integrates more cleanly with the existing tech stack.

The hybrid approach adds cost and complexity without removing any limitations.

---

## Docker Compose Configuration

This is the production-ready Docker Compose for Sentinel's data layer. Pin all image tags. Never use `:latest` in production.

```yaml
version: "3.8"

services:
  # ============================================================
  # PRIMARY DATABASE: PostgreSQL + TimescaleDB + PostGIS + pgvector
  # ============================================================
  sentinel-db:
    image: timescale/timescaledb-ha:pg17-ts2.18.0
    container_name: sentinel-db
    environment:
      POSTGRES_DB: sentinel
      POSTGRES_USER: sentinel
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - sentinel_pgdata:/home/postgres/pgdata/data
      - ./docker/init-db:/docker-entrypoint-initdb.d
    shm_size: "512mb"
    command:
      - "postgres"
      - "-c"
      - "shared_preload_libraries=timescaledb"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=2GB"
      - "-c"
      - "work_mem=64MB"
      - "-c"
      - "maintenance_work_mem=512MB"
      - "-c"
      - "effective_cache_size=6GB"
      - "-c"
      - "max_wal_size=2GB"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "max_worker_processes=16"
      - "-c"
      - "max_parallel_workers_per_gather=4"
      - "-c"
      - "timescaledb.max_background_workers=8"
      - "-c"
      - "log_min_duration_statement=1000"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sentinel -d sentinel"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - sentinel-net

  # ============================================================
  # REDIS: Cache, Celery broker, pub/sub for real-time
  # ============================================================
  sentinel-redis:
    image: redis:7.4-alpine
    container_name: sentinel-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - sentinel_redis_data:/data
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - sentinel-net

  # ============================================================
  # ELASTICSEARCH: Full-text search across events and entities
  # ============================================================
  sentinel-elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: sentinel-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - cluster.name=sentinel-search
    ports:
      - "${ES_PORT:-9200}:9200"
    volumes:
      - sentinel_es_data:/usr/share/elasticsearch/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - sentinel-net

  # ============================================================
  # PGBOUNCER: Connection pooling for PostgreSQL
  # ============================================================
  sentinel-pgbouncer:
    image: edoburu/pgbouncer:1.23.1
    container_name: sentinel-pgbouncer
    environment:
      DATABASE_URL: "postgres://sentinel:${POSTGRES_PASSWORD}@sentinel-db:5432/sentinel"
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 500
      DEFAULT_POOL_SIZE: 25
      MIN_POOL_SIZE: 5
      RESERVE_POOL_SIZE: 5
      RESERVE_POOL_TIMEOUT: 3
    ports:
      - "${PGBOUNCER_PORT:-6432}:5432"
    depends_on:
      sentinel-db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - sentinel-net

volumes:
  sentinel_pgdata:
    driver: local
  sentinel_redis_data:
    driver: local
  sentinel_es_data:
    driver: local

networks:
  sentinel-net:
    driver: bridge
```

### Initialization Script

Place this file at `docker/init-db/001-extensions.sql`. PostgreSQL executes scripts in `/docker-entrypoint-initdb.d/` alphabetically on first container startup.

```sql
-- ============================================================
-- Sentinel Database Initialization
-- Extensions must be created before any tables that use them
-- ============================================================

-- Time-series partitioning and compression
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- Geospatial data types, indexes, and queries
CREATE EXTENSION IF NOT EXISTS postgis CASCADE;

-- Vector similarity search (semantic search, deduplication)
CREATE EXTENSION IF NOT EXISTS vector;

-- UUID generation
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Trigram similarity for fuzzy text matching
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Scheduled jobs inside the database
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Confirm extensions loaded
SELECT extname, extversion FROM pg_extension ORDER BY extname;
```

---

## CREATE TABLE Statements

### Table 1: `sources`

```sql
-- ============================================================
-- SOURCES: Master registry of all data sources
-- Tracks RSS feeds, APIs, scrapers, and manual sources
-- ============================================================
CREATE TABLE sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Identity
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,           -- URL-safe identifier
    url TEXT NOT NULL,                            -- Source homepage
    feed_url TEXT,                                -- RSS/Atom feed URL
    api_endpoint TEXT,                            -- API base URL (for ACLED, GDELT, etc.)
    source_type VARCHAR(50) NOT NULL              -- 'rss', 'api', 'scraper', 'manual'
        CHECK (source_type IN ('rss', 'api', 'scraper', 'manual')),
    category VARCHAR(100) NOT NULL               -- 'wire_service', 'ngo', 'government', 'academic', 'media', 'osint'
        CHECK (category IN ('wire_service', 'ngo', 'government', 'academic', 'media', 'osint', 'other')),

    -- Coverage
    region_focus TEXT[] DEFAULT '{}',             -- e.g., {'middle_east', 'sub_saharan_africa'}
    language VARCHAR(10) DEFAULT 'en',
    country_focus TEXT[] DEFAULT '{}',            -- ISO 3166-1 alpha-2 codes

    -- Ranking scores (denormalized from rankings table for fast queries)
    composite_score NUMERIC(5,2) DEFAULT 0.00
        CHECK (composite_score >= 0 AND composite_score <= 100),
    score_tier VARCHAR(20) DEFAULT 'watch'
        CHECK (score_tier IN ('gold', 'silver', 'bronze', 'watch', 'suspended')),

    -- Polling configuration
    polling_interval_seconds INTEGER DEFAULT 900, -- 15 min default
    max_items_per_fetch INTEGER DEFAULT 100,
    rate_limit_rpm INTEGER,                       -- Requests per minute allowed by source

    -- Operational state
    is_active BOOLEAN DEFAULT true,
    is_manually_curated BOOLEAN DEFAULT false,
    last_checked_at TIMESTAMPTZ,
    last_successful_fetch_at TIMESTAMPTZ,
    last_error_at TIMESTAMPTZ,
    last_error_message TEXT,
    consecutive_errors INTEGER DEFAULT 0,
    total_events_ingested BIGINT DEFAULT 0,

    -- Metadata
    notes TEXT,
    config JSONB DEFAULT '{}',                   -- Source-specific config (API keys, params, etc.)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes for sources
CREATE INDEX idx_sources_active ON sources (is_active) WHERE is_active = true;
CREATE INDEX idx_sources_type ON sources (source_type);
CREATE INDEX idx_sources_category ON sources (category);
CREATE INDEX idx_sources_composite_score ON sources (composite_score DESC);
CREATE INDEX idx_sources_tier ON sources (score_tier);
CREATE INDEX idx_sources_next_check ON sources (last_checked_at ASC NULLS FIRST)
    WHERE is_active = true;
CREATE INDEX idx_sources_slug ON sources (slug);

-- Trigger to auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sources_updated_at
    BEFORE UPDATE ON sources
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Table 2: `events`

```sql
-- ============================================================
-- EVENTS: Normalized conflict events from all sources
-- Uses TimescaleDB hypertable for time-series partitioning
-- Uses PostGIS for geospatial indexing
-- Uses pgvector for semantic search
-- ============================================================
CREATE TABLE events (
    -- Primary key is a composite of id + ingested_at for TimescaleDB partitioning
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Source linkage
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE SET NULL,
    external_id VARCHAR(255),                    -- ID from original source (ACLED event_id_cnty, etc.)

    -- Core event data
    title TEXT NOT NULL,
    description TEXT,
    raw_content TEXT,                             -- Original unprocessed content
    event_type VARCHAR(100),                     -- 'battle', 'protest', 'explosion', 'strategic_development', etc.
    event_date TIMESTAMPTZ,                      -- When the event actually occurred
    published_at TIMESTAMPTZ,                    -- When the source published it

    -- Geospatial (PostGIS)
    location_name TEXT,
    country_code CHAR(2),                        -- ISO 3166-1 alpha-2
    country_name VARCHAR(100),
    admin1 VARCHAR(100),                         -- State/province
    admin2 VARCHAR(100),                         -- District/county
    coordinates GEOMETRY(POINT, 4326),           -- PostGIS WGS84 point
    geo_precision VARCHAR(20) DEFAULT 'unknown'  -- 'exact', 'approximate', 'admin1', 'country'
        CHECK (geo_precision IN ('exact', 'approximate', 'admin1', 'admin2', 'country', 'unknown')),

    -- Actors and entities
    actors JSONB DEFAULT '[]',                   -- [{name, type, side}]
    entities JSONB DEFAULT '[]',                 -- NLP-extracted named entities

    -- Classification and severity
    severity INTEGER CHECK (severity BETWEEN 1 AND 10),
    fatalities INTEGER DEFAULT 0,
    fatalities_estimate VARCHAR(20),             -- 'exact', 'estimated', 'unknown'
    tags TEXT[] DEFAULT '{}',

    -- Deduplication
    content_hash VARCHAR(64) NOT NULL,           -- SHA-256 of normalized title + description
    duplicate_of UUID,                           -- Points to canonical event if duplicate
    is_duplicate BOOLEAN DEFAULT false,

    -- Semantic search (pgvector)
    embedding VECTOR(384),                       -- all-MiniLM-L6-v2 embeddings

    -- Quality and provenance
    confidence_score NUMERIC(3,2) DEFAULT 0.50   -- 0.00-1.00
        CHECK (confidence_score >= 0 AND confidence_score <= 1),
    is_verified BOOLEAN DEFAULT false,
    verification_source TEXT,
    source_url TEXT,

    -- Composite primary key required for TimescaleDB
    PRIMARY KEY (id, ingested_at)
);

-- Convert to TimescaleDB hypertable partitioned by ingested_at
-- chunk_time_interval of 1 day means one partition per day
SELECT create_hypertable('events', 'ingested_at',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Indexes for events
CREATE INDEX idx_events_source ON events (source_id, ingested_at DESC);
CREATE INDEX idx_events_type ON events (event_type);
CREATE INDEX idx_events_date ON events (event_date DESC);
CREATE INDEX idx_events_country ON events (country_code);
CREATE INDEX idx_events_hash ON events (content_hash);
CREATE INDEX idx_events_duplicate ON events (is_duplicate) WHERE is_duplicate = false;
CREATE INDEX idx_events_geo ON events USING GIST (coordinates);
CREATE INDEX idx_events_severity ON events (severity DESC) WHERE severity >= 7;
CREATE INDEX idx_events_tags ON events USING GIN (tags);
CREATE INDEX idx_events_actors ON events USING GIN (actors);
CREATE INDEX idx_events_embedding ON events USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Enable compression on older chunks (events older than 30 days)
ALTER TABLE events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'source_id, country_code',
    timescaledb.compress_orderby = 'ingested_at DESC'
);

-- Auto-compress chunks older than 30 days
SELECT add_compression_policy('events', INTERVAL '30 days');

-- Auto-drop chunks older than 2 years (configurable retention)
SELECT add_retention_policy('events', INTERVAL '2 years');
```

### Table 3: `rankings`

```sql
-- ============================================================
-- RANKINGS: Source reliability and quality scores over time
-- Uses TimescaleDB hypertable for score trend analysis
-- ============================================================
CREATE TABLE rankings (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Source linkage
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,

    -- Composite score (0-100)
    composite_score NUMERIC(5,2) NOT NULL
        CHECK (composite_score >= 0 AND composite_score <= 100),
    score_tier VARCHAR(20) NOT NULL
        CHECK (score_tier IN ('gold', 'silver', 'bronze', 'watch', 'suspended')),

    -- Dimension 1: Feed Reliability (30% weight, 0-100 normalized)
    reliability_score NUMERIC(5,2) DEFAULT 0
        CHECK (reliability_score >= 0 AND reliability_score <= 100),
    reliability_uptime_pct NUMERIC(5,2),         -- 30-day rolling uptime %
    reliability_avg_latency_ms INTEGER,           -- Average response time
    reliability_error_rate NUMERIC(5,4),          -- Errors / total checks
    reliability_ssl_valid BOOLEAN,
    reliability_format_compliant BOOLEAN,

    -- Dimension 2: Content Quality (30% weight, 0-100 normalized)
    content_quality_score NUMERIC(5,2) DEFAULT 0
        CHECK (content_quality_score >= 0 AND content_quality_score <= 100),
    content_relevance_rate NUMERIC(5,4),          -- % items matching conflict keywords
    content_entity_richness NUMERIC(5,2),         -- Avg named entities per item
    content_duplication_rate NUMERIC(5,4),         -- % duplicate content
    content_freshness_hours NUMERIC(8,2),          -- Avg time from event to publication
    content_geo_specificity NUMERIC(5,4),          -- % items with extractable location

    -- Dimension 3: Source Authority (25% weight, 0-100 normalized)
    authority_score NUMERIC(5,2) DEFAULT 0
        CHECK (authority_score >= 0 AND authority_score <= 100),
    authority_org_type_score NUMERIC(5,2),
    authority_editorial_score NUMERIC(5,2),
    authority_historical_accuracy NUMERIC(5,2),
    authority_crossref_rate NUMERIC(5,4),          -- How often corroborated by other sources

    -- Dimension 4: Operational Value (15% weight, 0-100 normalized)
    operational_score NUMERIC(5,2) DEFAULT 0
        CHECK (operational_score >= 0 AND operational_score <= 100),
    operational_update_freq NUMERIC(8,2),          -- Items per day
    operational_coverage_breadth INTEGER,           -- Distinct regions covered
    operational_unique_content_rate NUMERIC(5,4),   -- % not in higher-ranked sources

    -- Calculation metadata
    calculation_window_days INTEGER DEFAULT 30,    -- How many days of data used
    events_analyzed INTEGER DEFAULT 0,             -- Number of events in calculation
    health_checks_analyzed INTEGER DEFAULT 0,      -- Number of health checks in window

    PRIMARY KEY (id, recorded_at)
);

-- Convert to TimescaleDB hypertable
SELECT create_hypertable('rankings', 'recorded_at',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

-- Indexes for rankings
CREATE INDEX idx_rankings_source ON rankings (source_id, recorded_at DESC);
CREATE INDEX idx_rankings_composite ON rankings (composite_score DESC);
CREATE INDEX idx_rankings_tier ON rankings (score_tier);

-- Continuous aggregate: daily average scores per source
CREATE MATERIALIZED VIEW rankings_daily
WITH (timescaledb.continuous) AS
SELECT
    source_id,
    time_bucket('1 day', recorded_at) AS day,
    AVG(composite_score) AS avg_composite,
    AVG(reliability_score) AS avg_reliability,
    AVG(content_quality_score) AS avg_content_quality,
    AVG(authority_score) AS avg_authority,
    AVG(operational_score) AS avg_operational,
    COUNT(*) AS num_calculations
FROM rankings
GROUP BY source_id, time_bucket('1 day', recorded_at)
WITH NO DATA;

-- Refresh policy: update daily aggregate every hour
SELECT add_continuous_aggregate_policy('rankings_daily',
    start_offset => INTERVAL '7 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Enable compression on older ranking data
ALTER TABLE rankings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'source_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);

SELECT add_compression_policy('rankings', INTERVAL '90 days');
```

---

## Comparison Tables

### Final Decision Matrix: Supabase vs Self-Hosted vs Hybrid

| Criteria | Supabase (Hosted) | Self-Hosted PostgreSQL | Hybrid | Winner |
|----------|-------------------|----------------------|--------|--------|
| TimescaleDB support | Not available | Full support | Partial (self-hosted only) | **Self-hosted** |
| PostGIS usability | Via rpc() only | Native SQL | Split between systems | **Self-hosted** |
| pgvector support | Yes | Yes (+ pgvectorscale) | Yes | **Self-hosted** |
| Real-time at scale | Single-thread bottleneck | LISTEN/NOTIFY + Redis + WS | Mixed | **Self-hosted** |
| FastAPI integration | PostgREST (mismatch) | asyncpg/SQLAlchemy (native) | Two drivers | **Self-hosted** |
| Monthly cost (production) | $100-250+ | $50-85 | $75-175 | **Self-hosted** |
| Setup time | Minutes | 1-2 days (with Docker Compose above) | 2-3 days | **Supabase** |
| Ops burden | Minimal | Moderate (Docker + backups) | High (two systems) | **Supabase** |
| Vendor lock-in | High | None | Medium | **Self-hosted** |
| Auth solution | Built-in GoTrue | FastAPI-Users or custom JWT | GoTrue + custom | **Supabase** |
| Backup control | Limited | Full (pg_dump, WAL-G, PITR) | Split | **Self-hosted** |
| Extension flexibility | Limited catalog | Any PG extension | Split | **Self-hosted** |
| **Score (wins)** | **2** | **10** | **0** | **Self-hosted** |

### Storage Capacity Estimates (Self-Hosted)

| Data Type | Avg Row Size | Rows/Year (est.) | Storage/Year | 2-Year Total |
|-----------|-------------|-------------------|--------------|--------------|
| Events (without embeddings) | ~2 KB | 500,000 | ~1 GB | ~2 GB |
| Events (with 384d embeddings) | ~3.5 KB | 500,000 | ~1.75 GB | ~3.5 GB |
| Rankings (score snapshots) | ~500 bytes | 365,000 (1/day/source x 1000 sources) | ~180 MB | ~360 MB |
| Feed health checks | ~200 bytes | 3,500,000 (96/day x 100 sources) | ~700 MB | ~1.4 GB |
| Indexes + overhead | ~30% of data | -- | ~800 MB | ~1.6 GB |
| **Total** | | | **~3.4 GB/year** | **~6.9 GB** |

An 8 GB database easily holds 2+ years of data. A 240 GB NVMe VPS has decades of runway. TimescaleDB compression on older chunks reduces actual storage by 80-95%.

---

## Priority Implementation Order

1. **Docker Compose stack startup** -- Deploy the Docker Compose configuration above. Verify all four services (sentinel-db, sentinel-redis, sentinel-elasticsearch, sentinel-pgbouncer) start and pass health checks. Run the initialization SQL to create extensions. This unblocks all subsequent work. Estimated: 0.5 days.

2. **Create core schema (sources, events, rankings tables)** -- Run the CREATE TABLE statements above via Alembic migration. Verify hypertables are created, indexes exist, and compression policies are active. Estimated: 0.5 days.

3. **Seed initial sources** -- Bulk insert the known data sources from findings-01 (ACLED, GDELT, UCDP, ReliefWeb, RSS feeds). Populate name, url, feed_url, source_type, category, region_focus. Estimated: 0.5 days.

4. **Build FastAPI CRUD for sources** -- RESTful endpoints: GET /sources, GET /sources/{id}, POST /sources, PUT /sources/{id}, DELETE /sources/{id}. Include filtering by type, category, tier, and active status. Estimated: 1 day.

5. **Implement feed health check ingestion** -- Celery task that polls each active source on its configured interval, records HTTP status, latency, error type, and item count into the feed_health_checks table (or a dedicated hypertable). Estimated: 2 days.

6. **Implement event ingestion pipeline** -- Parse RSS/Atom feeds and API responses into normalized event rows. Compute content_hash for deduplication. Extract coordinates via geocoding or source metadata. Insert into events hypertable. Estimated: 3 days.

7. **Implement reliability scoring** -- Calculate Dimension 1 scores from feed_health_checks data (uptime %, latency, error rate). Write scores to rankings table and denormalize to sources.composite_score. Estimated: 1 day.

8. **Add pgvector embeddings** -- Generate 384-dimensional embeddings for event titles + descriptions using all-MiniLM-L6-v2. Store in events.embedding column. Build HNSW index. Create semantic search API endpoint. Estimated: 2 days.

9. **Implement remaining scoring dimensions** -- Content quality (Dimension 2), authority (Dimension 3), and operational value (Dimension 4). Calculate composite scores with configurable weights. Estimated: 3 days.

10. **Set up backup and monitoring** -- Configure pg_dump daily backups to object storage (S3/Backblaze B2). Set up Prometheus + Grafana for database metrics (connections, query latency, table sizes, replication lag). Estimated: 1 day.

---

## Migration/Escape Plan

If self-hosted PostgreSQL needs to change (e.g., team grows and ops burden becomes too heavy, or a managed service becomes more attractive):

### Escape Route 1: Move to Managed PostgreSQL (Easiest)

**Target**: DigitalOcean Managed PostgreSQL, AWS RDS, or Neon.

**Steps**:
1. `pg_dump --format=custom --no-owner sentinel > sentinel_backup.dump`
2. Create managed PostgreSQL instance with PostGIS extension enabled.
3. `pg_restore --no-owner -d sentinel sentinel_backup.dump`
4. Install TimescaleDB extension (available on Aiven, Timescale Cloud, some others -- NOT on all managed providers).
5. Install pgvector extension.
6. Update connection strings in FastAPI config. Update PgBouncer target.
7. Test all queries and hypertable operations.
8. Cut over DNS/connection strings.

**Estimated effort**: 1-2 days. **Risk**: Low. Standard PostgreSQL migration.

**Caveat**: Not all managed PostgreSQL providers support TimescaleDB. Timescale Cloud is the safest target if TimescaleDB is required. If the target does not support TimescaleDB, you must convert hypertables back to regular tables (losing automatic partitioning and compression) or use partitioned tables with native PostgreSQL declarative partitioning.

### Escape Route 2: Move to Supabase (If Requirements Change)

**Target**: Supabase hosted.

**Steps**:
1. Export schema without TimescaleDB-specific features (hypertables, compression policies, continuous aggregates).
2. Convert hypertables back to regular PostgreSQL tables with manual partitioning.
3. Remove pgvectorscale usage (pgvector is available on Supabase, but not pgvectorscale).
4. Create Supabase project, enable PostGIS and pgvector extensions.
5. Import data via `pg_dump`/`pg_restore` or Supabase CLI.
6. Rewrite real-time layer from LISTEN/NOTIFY to Supabase Realtime subscriptions.
7. Rewrite auth layer from FastAPI-Users to Supabase GoTrue.
8. Wrap all PostGIS queries in database functions callable via `rpc()`.

**Estimated effort**: 1-2 weeks. **Risk**: Medium-high. Significant application code changes required.

### Escape Route 3: Move to Timescale Cloud (Best Managed Option)

**Target**: Timescale Cloud (managed TimescaleDB service).

**Steps**:
1. Create Timescale Cloud instance.
2. Use `pg_dump`/`pg_restore` -- full compatibility since Timescale Cloud runs the same extensions.
3. Update connection strings.
4. All hypertables, compression policies, continuous aggregates, and pgvector indexes transfer directly.

**Estimated effort**: 0.5-1 day. **Risk**: Very low. Near-identical environment.

**Cost**: Timescale Cloud starts at ~$29/month for a small instance. Production instances run $50-200/month depending on compute and storage.

### Migration Safety Net

To ensure we can always migrate:

1. **Use Alembic for all schema changes** -- never modify the database manually. Every migration is versioned and reversible.
2. **Avoid stored procedures for business logic** -- keep logic in FastAPI Python code, not in PL/pgSQL. Database functions should only be used for triggers, aggregations, and performance-critical queries.
3. **Daily automated backups** -- `pg_dump` to object storage with 30-day retention. Test restores monthly.
4. **Document all TimescaleDB-specific features** -- maintain a list of what would need to change if moving to standard PostgreSQL.
5. **Use standard SQL where possible** -- avoid TimescaleDB-specific SQL syntax except for hypertable creation, compression policies, and continuous aggregates.

---

## Cost Analysis

### Self-Hosted Path (Recommended) -- Monthly Costs

| Component | Provider | Spec | Cost/Month |
|-----------|----------|------|-----------|
| Primary VPS | Hetzner CX42 | 8 vCPU, 16 GB RAM, 160 GB NVMe | ~$30 |
| OR Primary VPS | Hetzner CX52 | 16 vCPU, 32 GB RAM, 320 GB NVMe | ~$55 |
| Backup storage | Backblaze B2 | 50 GB | ~$0.25 |
| Domain + DNS | Cloudflare | Free tier | $0 |
| SSL certificates | Let's Encrypt | Free | $0 |
| **Total (small)** | | | **~$30/mo** |
| **Total (production)** | | | **~$55/mo** |

All services (PostgreSQL, Redis, Elasticsearch, PgBouncer) run on the same VPS via Docker Compose. For high availability, add a second VPS as a streaming replica (~$30/mo extra).

### Supabase Path (Rejected) -- Monthly Costs

| Component | Cost/Month | Notes |
|-----------|-----------|-------|
| Supabase Pro | $25 | Base plan |
| Compute upgrade (4 vCPU) | $50-100 | Needed for pgvector + real-time |
| Realtime message overages | $5-25 | ~2-10M messages/month |
| Separate TimescaleDB (Timescale Cloud) | $29+ | Not available on Supabase |
| Separate Redis (Upstash or Railway) | $10-25 | For Celery + caching |
| Separate Elasticsearch (Elastic Cloud) | $25-95 | For full-text search |
| **Total** | **$144-295/mo** | 3-5x more expensive |

### Year-One Cost Comparison

| Path | Monthly | Annual | Notes |
|------|---------|--------|-------|
| Self-hosted (small) | $30 | $360 | All-in-one VPS |
| Self-hosted (production) | $55 | $660 | Larger VPS |
| Self-hosted (HA) | $85 | $1,020 | Primary + replica |
| Supabase (minimum viable) | $144 | $1,728 | Pro + supplements |
| Supabase (realistic) | $220 | $2,640 | With compute upgrade |

---

## Open Questions

1. **TimescaleDB Community vs Licensed Edition** -- The `timescaledb-ha` image includes the Apache 2.0 licensed community edition by default. Continuous aggregates and compression are available in the community edition as of TimescaleDB 2.x. Verify that all features we need (particularly `add_compression_policy` and `add_continuous_aggregate_policy`) remain in the community edition and have not been moved behind the Timescale License in recent releases.

2. **pgvectorscale AVX2 requirement** -- pgvectorscale (DiskANN) requires AVX2 and FMA CPU instructions. Most modern bare-metal servers support this, but some VPS providers (especially budget ones) may not expose these instructions to guest VMs. Need to verify that the chosen Hetzner instance type supports AVX2 before relying on pgvectorscale. If not available, standard pgvector HNSW indexes are sufficient for our scale (<10M vectors).

3. **Elasticsearch vs PostgreSQL full-text search** -- PostgreSQL has built-in full-text search (tsvector/tsquery) that may be sufficient for Sentinel's needs without a separate Elasticsearch instance. This would reduce complexity and cost. Need to benchmark: can PostgreSQL FTS handle multi-language full-text search across 1M+ events with acceptable latency (<100ms)? If yes, Elasticsearch can be deferred.

4. **PgBouncer transaction mode and prepared statements** -- PgBouncer in transaction pooling mode does not support prepared statements (used by some PostgreSQL drivers by default). Need to verify that asyncpg and SQLAlchemy async work correctly with PgBouncer transaction mode, or use session pooling instead.

5. **Backup restoration testing** -- The migration plan assumes pg_dump/pg_restore works cleanly with TimescaleDB hypertables. TimescaleDB has specific backup/restore procedures (`timescaledb_pre_restore()` and `timescaledb_post_restore()` functions). Need to test and document the exact backup/restore procedure.

6. **Hypertable primary key constraint** -- TimescaleDB requires that the partitioning column (`ingested_at`) be part of any unique constraint or primary key. This means the events table uses a composite primary key `(id, ingested_at)` instead of just `(id)`. Foreign keys from other tables to events must include both columns, or we need a workaround (e.g., a unique index on `id` alone that is not a primary key constraint).

---

## Sources & References

### Supabase Real-Time Performance
- [Supabase Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)
- [Supabase Realtime Limits](https://supabase.com/docs/guides/realtime/limits)
- [Supabase Postgres Changes Documentation](https://supabase.com/docs/guides/realtime/postgres-changes)
- [Supabase is Supaslow (Medium)](https://medium.com/@turingvang/supabase-is-supaslow-cee686f15372)
- [Supabase Realtime High Latency Diagnosis](https://drdroid.io/stack-diagnosis/supabase-realtime-high-latency-on-the-server-side-is-affecting-performance)

### Supabase PostGIS
- [Supabase PostGIS Documentation](https://supabase.com/docs/guides/database/extensions/postgis)
- [PostGIS self-hosted bug -- GitHub Issue #27295](https://github.com/supabase/supabase/issues/27295)
- [Supabase PostGIS geo queries discussion](https://github.com/orgs/supabase/discussions/5390)

### Supabase Pricing & Limits
- [Supabase Pricing Page](https://supabase.com/pricing)
- [Supabase True Cost Breakdown (Metacto)](https://www.metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance)
- [Supabase Pricing 2026 (UI Bakery)](https://uibakery.io/blog/supabase-pricing)
- [Supabase Database Size Documentation](https://supabase.com/docs/guides/platform/database-size)

### Supabase vs Self-Hosted Recommendations
- [PostgreSQL vs Supabase Deployment Guide (Leanware)](https://www.leanware.co/insights/postgresql-vs-supabase-deployment-guide-startups)
- [Postgres vs Supabase Benchmarks (pgbench.com)](https://pgbench.com/comparisons/postgres-vs-supabase/)
- [Self-hosting Supabase: Worth It? (Vela/Simplyblock)](https://vela.simplyblock.io/articles/self-hosting-supabase-worth-it/)
- [Self-hosted Supabase Discussion (GitHub)](https://github.com/orgs/supabase/discussions/39820)
- [HN: Self-hosted Supabase for Healthcare](https://news.ycombinator.com/item?id=42366172)

### Docker Compose & TimescaleDB Setup
- [TimescaleDB Docker Installation Docs](https://docs.timescale.com/self-hosted/latest/install/installation-docker/)
- [TimescaleDB Docker Configuration](https://docs.timescale.com/self-hosted/latest/configuration/docker-config/)
- [timescale/timescaledb-ha Dockerfile (GitHub)](https://github.com/timescale/timescaledb-docker-ha/blob/master/Dockerfile)
- [kartoza/docker-postgis (GitHub)](https://github.com/kartoza/docker-postgis)
- [binakot/PostgreSQL-PostGIS-TimescaleDB (GitHub)](https://github.com/binakot/PostgreSQL-PostGIS-TimescaleDB)
- [TimescaleDB Docker Hub](https://hub.docker.com/r/timescale/timescaledb)
- [PostGIS with Docker Compose (Florian Neukirchen)](https://www.riannek.de/2024/postgis-with-docker-compose/)

### pgvector & Semantic Search
- [pgvectorscale (GitHub)](https://github.com/timescale/pgvectorscale)
- [TimescaleDB Vector Search on Kubernetes (Tiger Data)](https://www.tigerdata.com/blog/deploying-timescaledb-vector-search-cloudnativepg-kubernetes-operator)
- [PostgreSQL vectorscale and timescale (Medium)](https://gregorylmagnusson.medium.com/pgvectorscale-39761c4ae8b2)

### Production PostgreSQL
- [Optimizing Geospatial and Time-Series Queries with TimescaleDB + PostGIS (Medium)](https://medium.com/@marcoscedenillabonet/optimizing-geospatial-and-time-series-queries-with-timescaledb-and-postgis-4978ea2ef8af)
- [Quick Guide to Running TimescaleDB on Docker (Composite Code)](https://compositecode.blog/2025/03/28/setup-timescaledb-with-docker-compose-a-step-by-step-guide/)
