# Research Findings 02: System Architecture & Tech Stack

## Executive Summary

This document presents comprehensive research findings for building a Palantir-like global conflict monitoring dashboard. After analyzing Palantir's architecture, competing intelligence platforms (Dataminr, Recorded Future), modern streaming architectures, and best-of-breed database technologies, we recommend a **modular, event-driven architecture** built on PostgreSQL (with PostGIS and TimescaleDB extensions) as the primary datastore, React for the frontend, FastAPI/Python for the backend, Redis for caching, and optional Elasticsearch for advanced search -- with a clear MVP-first approach that can scale to production-grade intelligence tooling.

---

## 1. Palantir Architecture Analysis

### How Palantir Works

Palantir's platforms (Gotham for intelligence/defense, Foundry for enterprise) are built around a central concept called the **Ontology** -- a unified semantic and operational layer that maps real-world entities into digital objects with properties, links, actions, and security controls.

#### Three-Layer Architecture
1. **Language Layer**: Models semantic objects, links, properties, along with kinetic actions and automations. Defines the logic for how actions operate and interact with external systems.
2. **Engine Layer**: Substantiates the Language. Provides modular read architecture (high-scale SQL, real-time subscriptions, materializations) and scalable write architecture (atomic transactions, batch mutations, streaming, Change Data Capture).
3. **Toolchain Layer**: User-facing analytical and operational tools -- Object Explorer, Quiver (analysis), Workshop (applications), and dashboards.

#### Key Design Principles from Palantir
- **Ontology-First**: Everything maps to real-world entities (people, organizations, places, events) with typed relationships
- **Data Fusion**: Ingest from any source (ERPs, CRMs, sensors, documents, geospatial repos) and unify into coherent objects
- **Microservices Backend**: Object Data Funnel (write orchestration), Ontology Metadata Service (entity definitions), Object Storage V2 (horizontally scalable read/write separation)
- **Fine-Grained Security**: Every data element tethered to its source with attribute-level access controls
- **Gotham Specifics**: Transforms structured and unstructured data into ontology objects representing people, organizations, places, documents, and events -- exactly our use case

#### What Makes It "Intelligence-Grade"
- Entity resolution across disparate sources
- Link analysis revealing hidden connections
- Temporal analysis showing evolution over time
- Geospatial layering of all entities
- Classification-aware security throughout

### Our Takeaway
We should adopt Palantir's ontology-inspired approach: define a clear entity model (conflicts, actors, locations, events, sources) with typed relationships, and build our architecture around this model. We do NOT need to replicate Palantir's full complexity -- we can achieve 80% of the intelligence-grade feel with 20% of the infrastructure by focusing on entity modeling, geospatial visualization, temporal analysis, and clean data fusion.

---

## 2. Competing Platform Analysis

### Dataminr
- **Architecture**: AI-powered real-time event detection platform
- **Scale**: 50+ proprietary LLMs, processes massive daily data flows, detects ~500,000 daily events
- **Key Innovation**: Agentic AI -- autonomous agents that seek critical context 24/7
- **ReGenAI**: Delivers dynamic event briefs automatically
- **Users**: 100+ US government agencies, 20+ international governments
- **Lesson for Us**: Focus on automated event detection and contextual enrichment; filter noise aggressively

### Recorded Future
- **Architecture**: Enterprise AI platform with nine-module architecture
- **Scale**: 50+ proprietary language models, terabytes of data in 150+ languages
- **Key Innovation**: Deep web and dark web source integration; cuts analysis cycles from hours to minutes
- **Acquisition**: Purchased by Mastercard for $2.65B -- validates the market
- **Lesson for Us**: Multi-language processing is critical for global conflict monitoring; modular architecture allows specialization

### OSINT Market Context
- Market valued at $8.69B in 2024, projected $46.12B by 2034 (18% CAGR)
- Hybrid cloud patterns emerging: unclassified feeds processed in cloud, enriched data ported behind firewalls
- Zero-trust architecture becoming standard

### Our Positioning
We are building an open-source/academic-grade tool, not competing with $2.65B platforms. Our advantage is transparency, customizability, and focus on conflict/geopolitical analysis specifically. We should aim for the "80% of Recorded Future at 1% of the cost" sweet spot.

---

## 3. Recommended Tech Stack

### 3.1 Frontend: React (SPA) with Vite

#### Decision: React SPA over Next.js

| Criteria | React SPA | Next.js |
|---|---|---|
| SEO needs | Not needed (behind login) | SSR/SSG strength, but irrelevant here |
| Real-time updates | Native, simple | SSR adds hydration complexity |
| State management | Full control | More constrained with Server Components |
| Dashboard complexity | Ideal -- purpose-built for interactive UIs | Adds unnecessary SSR overhead |
| Developer ecosystem | 39.5% adoption (Stack Overflow 2024) | 27% adoption |
| Learning curve | Lower for dashboard-focused work | Higher with SSR/SSG concepts |
| Bundle size control | Full control | Framework overhead |

**Justification**: React is the industry standard for complex, interactive dashboards. SEO is irrelevant for an intelligence dashboard behind authentication. Next.js's SSR/SSG features add complexity without benefit. React gives us full control over state management and real-time data patterns.

#### Frontend Stack Details
- **Build Tool**: Vite (fast HMR, modern ESM)
- **State Management**: Zustand (lightweight) + React Query/TanStack Query (server state)
- **Mapping**: Mapbox GL JS or Deck.gl (WebGL-powered geospatial visualization)
- **Charts**: D3.js (custom visualizations) + Recharts (standard charts)
- **Component Library**: Shadcn/ui (Tailwind-based, customizable) or Ant Design (enterprise-grade)
- **Real-time**: EventSource API (SSE) for dashboard updates
- **Data Grid**: AG Grid or TanStack Table (for large datasets)
- **Timeline**: vis-timeline or custom D3 implementation

### 3.2 Backend: Python/FastAPI

| Criteria | FastAPI (Python) | Express (Node.js) | Go |
|---|---|---|---|
| ML/NLP ecosystem | Excellent (spaCy, transformers, scikit-learn) | Limited | Limited |
| Async support | Native async/await | Native | Native goroutines |
| API documentation | Auto-generated OpenAPI/Swagger | Manual or plugins | Manual |
| Data processing | pandas, numpy, geopandas | Limited | Limited |
| Development speed | Very fast | Fast | Moderate |
| Type safety | Pydantic models, type hints | TypeScript required | Built-in |
| WebSocket/SSE | Native support | Native support | Native support |
| Community for OSINT | Strong (most OSINT tools are Python) | Moderate | Small |

**Justification**: Python is the lingua franca of data science, NLP, and OSINT tooling. FastAPI provides async performance comparable to Node.js while giving access to the entire Python ML ecosystem. This is critical for entity extraction, sentiment analysis, and predictive modeling down the line.

#### Backend Stack Details
- **Framework**: FastAPI with Uvicorn (ASGI server)
- **ORM**: SQLAlchemy 2.0 (async) + GeoAlchemy2 (PostGIS support)
- **Task Queue**: Celery with Redis broker (background feed polling, analysis pipelines)
- **Caching**: Redis (dashboard state, session management, rate limiting)
- **API Style**: REST for CRUD operations, SSE for real-time streaming, GraphQL (optional, via Strawberry) for complex entity queries
- **Authentication**: JWT tokens + OAuth2 (FastAPI built-in support)
- **NLP Pipeline**: spaCy (entity extraction), transformers (classification)

### 3.3 Database Architecture

This is the most critical architectural decision. Rather than using five separate databases, we recommend a **PostgreSQL-centric approach** with targeted extensions.

#### Primary Database: PostgreSQL 16+ with Extensions

| Extension | Purpose | Why Not a Separate DB |
|---|---|---|
| **PostGIS** | Geospatial queries, conflict mapping | Eliminates need for separate geo DB; mature, battle-tested |
| **TimescaleDB** | Time-series data for events | SQL-compatible, works within same PostgreSQL instance |
| **pg_trgm** | Fuzzy text search | Handles moderate search needs without Elasticsearch |
| **pgvector** | Vector embeddings for semantic search | ML-ready without separate vector DB |

#### Database Decision Matrix

| Database Option | Recommendation | Justification |
|---|---|---|
| **PostgreSQL + PostGIS + TimescaleDB** | PRIMARY - Use this | Single database handles 90% of needs: relational data, geospatial, time-series. SQL everywhere. One backup strategy. One connection pool. Massive ecosystem. |
| **Redis** | USE - Required | Caching layer, task queue broker, real-time pub/sub for SSE. Not a primary data store. |
| **Elasticsearch** | DEFER to Phase 2 | PostgreSQL full-text search handles <1M records well. Add Elasticsearch only when search becomes a bottleneck. Saves massive operational overhead. |
| **Neo4j** | DEFER to Phase 3 | Entity relationships can be modeled with PostgreSQL junction tables + recursive CTEs initially. Add Neo4j only for complex graph traversals (6+ hops). |
| **InfluxDB** | SKIP | TimescaleDB provides time-series within PostgreSQL. No need for separate time-series DB. TimescaleDB outperforms InfluxDB on complex queries and high-cardinality data. |
| **MongoDB** | SKIP | PostgreSQL JSONB columns handle document storage. No need for separate document store. |

#### TimescaleDB vs InfluxDB Deep Dive

| Criteria | TimescaleDB | InfluxDB |
|---|---|---|
| Query language | Full SQL | InfluxQL / Flux (proprietary) |
| Complex queries | Vastly outperforms InfluxDB | Limited analytical capabilities |
| Reliability | PostgreSQL-grade (decades of testing) | Community reports of data loss issues |
| High cardinality | Better performance at scale | Performance degrades with cardinality |
| Integration | PostGIS, pgvector, all PG extensions | Separate ecosystem |
| Operational cost | Same PostgreSQL instance | Separate infrastructure |
| Learning curve | SQL (universal) | Custom languages |
| InfluxDB 3.0 limits | N/A | 72h retention, 5 DB limit (free tier) |

**Winner: TimescaleDB** -- runs within PostgreSQL, full SQL, better complex query performance, one operational surface.

#### Elasticsearch vs PostgreSQL Full-Text Search

| Criteria | PostgreSQL FTS | Elasticsearch | pg_search (ParadeDB) |
|---|---|---|---|
| Performance (<1M rows) | Good (13-16ms optimized) | Excellent (5-30ms) | Excellent (matches ES) |
| Performance (>5M rows) | Degrades | Excellent | Excellent |
| Relevance ranking | Basic | Gold standard (BM25+) | Good (Tantivy/BM25) |
| Fuzzy matching | pg_trgm extension | Built-in, excellent | Built-in |
| Operational overhead | Zero (same DB) | High (separate cluster, ETL) | Low (PG extension) |
| Data freshness | Real-time | Lags behind (ETL delay) | Real-time |
| Cost | Free | Significant infrastructure cost | Free |

**Recommendation**: Start with PostgreSQL FTS + pg_trgm. Evaluate pg_search (ParadeDB) as a drop-in upgrade. Only add Elasticsearch at >5M records if search quality is insufficient.

---

## 4. System Architecture

### 4.1 High-Level Architecture (Textual Diagram)

```
                                    USERS
                                      |
                                  [Nginx/Caddy]
                                   /        \
                              [React SPA]   [FastAPI Backend]
                                              /    |     \
                                          [Redis] [Celery Workers] [PostgreSQL]
                                                    |               (PostGIS +
                                              [Feed Pollers]        TimescaleDB +
                                              [NLP Pipeline]        pgvector)
                                              [Alert Engine]
```

### 4.2 Detailed Component Architecture

```
+------------------------------------------------------------------+
|                        FRONTEND (React SPA)                       |
|                                                                    |
|  +------------+  +------------+  +-----------+  +---------------+ |
|  | Map View   |  | Timeline   |  | Entity    |  | Alert         | |
|  | (Mapbox/   |  | (D3/vis)   |  | Explorer  |  | Dashboard     | |
|  |  Deck.gl)  |  |            |  | (Network) |  |               | |
|  +------------+  +------------+  +-----------+  +---------------+ |
|                                                                    |
|  State: Zustand + TanStack Query | Real-time: SSE EventSource    |
+------------------------------------------------------------------+
                              |
                         HTTPS / SSE
                              |
+------------------------------------------------------------------+
|                     API GATEWAY (FastAPI)                          |
|                                                                    |
|  +---------------+  +---------------+  +------------------------+ |
|  | REST API      |  | SSE Streaming |  | Auth (JWT + OAuth2)    | |
|  | /api/v1/*     |  | /stream/*     |  |                        | |
|  +---------------+  +---------------+  +------------------------+ |
|                                                                    |
|  +---------------+  +---------------+  +------------------------+ |
|  | Event Router  |  | Rate Limiter  |  | Request Validation     | |
|  | (pub/sub)     |  | (Redis)       |  | (Pydantic)             | |
|  +---------------+  +---------------+  +------------------------+ |
+------------------------------------------------------------------+
                              |
              +---------------+------------------+
              |                                  |
+---------------------------+    +----------------------------------+
|     WORKER LAYER          |    |      DATA LAYER                  |
|     (Celery + Redis)      |    |                                  |
|                           |    |  +---------------------------+   |
|  +--------------------+   |    |  | PostgreSQL 16+            |   |
|  | Feed Pollers       |   |    |  |                           |   |
|  | (RSS, API, scrape) |   |    |  |  +-- PostGIS (geo)       |   |
|  +--------------------+   |    |  |  +-- TimescaleDB (time)   |   |
|                           |    |  |  +-- pgvector (embeddings)|   |
|  +--------------------+   |    |  |  +-- pg_trgm (search)     |   |
|  | NLP Pipeline       |   |    |  +---------------------------+   |
|  | (spaCy, BERT)      |   |    |                                  |
|  +--------------------+   |    |  +---------------------------+   |
|                           |    |  | Redis                     |   |
|  +--------------------+   |    |  | (cache, pubsub, queues)   |   |
|  | Entity Resolver    |   |    |  +---------------------------+   |
|  | (dedup, linking)   |   |    |                                  |
|  +--------------------+   |    |  +---------------------------+   |
|                           |    |  | File Storage (S3/MinIO)   |   |
|  +--------------------+   |    |  | (raw docs, images)        |   |
|  | Alert Engine       |   |    |  +---------------------------+   |
|  | (threshold, ML)    |   |    |                                  |
|  +--------------------+   |    +----------------------------------+
+---------------------------+
```

### 4.3 Data Flow

1. **Ingestion**: Celery workers poll RSS feeds, APIs (ACLED, GDELT, news APIs), and web sources on configurable schedules
2. **Processing**: NLP pipeline extracts entities (people, places, organizations), classifies event types, assigns severity scores
3. **Storage**: Processed events stored in PostgreSQL with geospatial coordinates (PostGIS), timestamps (TimescaleDB hypertables), and entity relationships
4. **Notification**: New events published to Redis pub/sub channel
5. **Streaming**: FastAPI SSE endpoints subscribe to Redis pub/sub, push updates to connected frontend clients
6. **Query**: Frontend queries REST API for historical data, filters, aggregations; receives real-time updates via SSE

---

## 5. Real-Time Streaming Architecture

### Decision: SSE over WebSockets for Dashboard Updates

| Criteria | SSE (Server-Sent Events) | WebSocket |
|---|---|---|
| Direction needed | Server -> Client (our use case) | Bidirectional |
| Complexity | Simple HTTP-based | Custom protocol (ws://) |
| Reconnection | Automatic (built into browser API) | Manual implementation required |
| HTTP/2 compatible | Yes (eliminates connection limits) | Separate protocol |
| Firewall friendly | Standard HTTPS | Can be blocked by corporate firewalls |
| Infrastructure | Standard HTTP load balancers | Requires WebSocket-aware proxies |
| Performance | Comparable for server-push scenarios | Slightly lower latency |
| Binary data | No (UTF-8 only) | Yes |

**Decision**: Use **SSE for dashboard data streaming** (event updates, alert notifications, metric changes). These are all server-to-client flows. Use standard HTTP POST/PUT for the rare client-to-server actions (creating alerts, saving views). Reserve WebSockets only if we add collaborative features (shared cursors, real-time chat) in the future.

### SSE Implementation Pattern
```
Client: EventSource('/api/stream/events?region=middle-east&severity=high')
Server: FastAPI SSE endpoint subscribes to Redis pub/sub with filters
Server: Yields formatted events as they arrive
Client: TanStack Query invalidation on SSE message -> re-fetch affected data
```

---

## 6. API Design

### REST API Structure

```
/api/v1/
  /auth/
    POST   /login              # JWT token
    POST   /refresh             # Refresh token
    GET    /me                  # Current user

  /events/
    GET    /                    # List events (paginated, filtered)
    GET    /:id                 # Single event detail
    POST   /search              # Full-text search with filters
    GET    /timeline            # Time-bucketed aggregation
    GET    /heatmap             # Geospatial density data

  /entities/
    GET    /                    # List entities (actors, orgs, locations)
    GET    /:id                 # Entity detail with relationships
    GET    /:id/network         # Entity relationship graph
    GET    /:id/timeline        # Entity activity over time

  /regions/
    GET    /                    # List monitored regions
    GET    /:id/summary         # Region conflict summary
    GET    /:id/events          # Events in region (geospatial query)

  /alerts/
    GET    /                    # User's alert configurations
    POST   /                    # Create alert rule
    PUT    /:id                 # Update alert rule
    DELETE /:id                 # Remove alert rule

  /dashboards/
    GET    /                    # User's saved dashboards
    POST   /                    # Create dashboard layout
    PUT    /:id                 # Update dashboard

  /sources/
    GET    /                    # Data source status/health
    GET    /:id/feed            # Recent items from source

  /analytics/
    GET    /severity-trend      # Severity over time
    GET    /actor-network       # Actor co-occurrence
    GET    /predictions         # ML model predictions

/api/stream/
    GET    /events              # SSE: real-time event stream
    GET    /alerts              # SSE: alert notifications
    GET    /metrics             # SSE: dashboard metric updates
```

### Query Parameter Standards
```
# Pagination
?page=1&per_page=50

# Filtering
?severity=high,critical&region=middle-east&date_from=2024-01-01&date_to=2024-12-31

# Sorting
?sort_by=severity&sort_order=desc

# Geospatial
?lat=33.8&lon=35.5&radius_km=500

# Time aggregation
?bucket=day&from=2024-01-01&to=2024-06-01

# Full-text search
?q=military+operation&fuzzy=true
```

---

## 7. Entity Model (Ontology-Inspired)

Drawing from Palantir's ontology concept, our core entity model:

```
EVENT
  - id, title, description, event_type, severity (1-10)
  - location (PostGIS geometry point/polygon)
  - timestamp (TimescaleDB hypertable)
  - source_id, source_url, raw_content
  - embedding (pgvector for semantic search)
  - entities[] (extracted via NLP)

ACTOR
  - id, name, aliases[], type (state/non-state/individual/organization)
  - description, ideology, capabilities
  - locations[] (known operating areas)

LOCATION
  - id, name, country, admin_level
  - geometry (PostGIS polygon/point)
  - population, strategic_significance

CONFLICT
  - id, name, status (active/frozen/resolved)
  - start_date, end_date
  - region, description
  - actors[] (involved parties)
  - events[] (timeline of events)

SOURCE
  - id, name, type (rss/api/scraper), url
  - reliability_score, bias_rating
  - poll_interval, last_polled, status

RELATIONSHIP (junction table approach)
  - entity_a_type, entity_a_id
  - entity_b_type, entity_b_id
  - relationship_type (allied_with, opposed_to, operates_in, etc.)
  - confidence_score, source_id
  - valid_from, valid_to
```

---

## 8. MVP vs Full-Scale Architecture

### Phase 1: MVP (Weeks 1-6)

**Goal**: Working dashboard with real data, core visualizations, basic analysis

| Component | MVP Choice | Notes |
|---|---|---|
| Frontend | React + Vite + Mapbox + Recharts | Basic map, timeline, event list |
| Backend | FastAPI (monolithic) | Single service, no microservices yet |
| Database | PostgreSQL + PostGIS | No TimescaleDB yet, standard tables |
| Cache | Redis | Essential from day 1 |
| Search | PostgreSQL pg_trgm | Good enough for <100K records |
| Real-time | SSE (simple polling fallback) | Basic event streaming |
| Auth | JWT (simple) | No OAuth2 yet |
| Data Sources | 3-5 RSS feeds + ACLED API | Manual curation initially |
| NLP | spaCy (basic NER) | Location/person/org extraction |
| Deployment | Docker Compose (single server) | Simple, reproducible |
| Task Queue | Celery + Redis | Feed polling on schedule |

**MVP Architecture**: Monolithic FastAPI application + Celery workers + PostgreSQL + Redis, all in Docker Compose.

### Phase 2: Enhanced (Weeks 7-14)

| Component | Upgrade | Trigger |
|---|---|---|
| Database | Add TimescaleDB extension | When events table exceeds 500K rows |
| Search | Add pg_search (ParadeDB) or Elasticsearch | When full-text search quality is insufficient |
| NLP | Add transformer models (BERT/LLM) | When basic NER quality is insufficient |
| Frontend | Add D3 custom visualizations, network graph | User feedback on visualization needs |
| Entity Resolution | Deduplication pipeline | When duplicate entities become a problem |
| Alerts | Rule-based alert engine | User demand for notifications |
| Auth | OAuth2 + role-based access | Multi-user deployment |
| Deployment | Docker Compose with replicas | Performance needs |

### Phase 3: Production Scale (Weeks 15+)

| Component | Upgrade | Trigger |
|---|---|---|
| Architecture | Extract to microservices | When monolith becomes hard to maintain |
| Database | Add Neo4j for graph analysis | When relationship queries need 6+ hops |
| Database | Add pgvector embeddings | When semantic search is needed |
| Search | Elasticsearch cluster | When >5M records need search |
| Streaming | Kafka / Redis Streams | When >100 concurrent users need real-time |
| ML | Prediction models, anomaly detection | When historical data is sufficient for training |
| Deployment | Kubernetes | When horizontal scaling is required |
| Monitoring | Prometheus + Grafana + Sentry | Production observability |
| CDN | CloudFlare / AWS CloudFront | Global user base |

### Architecture Evolution Summary

```
MVP:        [React] -> [FastAPI Monolith] -> [PostgreSQL + Redis]
                              |
                        [Celery Workers]

Phase 2:    [React] -> [FastAPI + SSE] -> [PostgreSQL+PostGIS+TimescaleDB + Redis]
                              |                     + optional Elasticsearch
                        [Celery Workers]
                        [NLP Pipeline]

Phase 3:    [React] -> [API Gateway] -> [Event Service]    -> [PostgreSQL+PostGIS+Timescale]
                              |         [Entity Service]    -> [Neo4j (graph)]
                              |         [Search Service]    -> [Elasticsearch]
                              |         [Analytics Service] -> [Redis + ML models]
                              |         [Ingestion Service] -> [Kafka -> workers]
                              |
                        [Kubernetes orchestration]
```

---

## 9. Event-Driven Architecture Patterns

Based on research into microservices and EDA patterns, our recommended patterns:

### For MVP (Simplified Event-Driven)
- **Pub/Sub via Redis**: Celery workers publish events to Redis channels; FastAPI SSE endpoints subscribe and stream to clients
- **Task Queue Pattern**: Celery Beat schedules feed polling; workers process independently
- **Simple Event Flow**: Ingest -> Process -> Store -> Notify

### For Production Scale
- **Event Sourcing**: Store all raw events immutably; derive current state from event log
- **CQRS**: Separate write path (ingestion pipeline) from read path (dashboard queries)
- **Change Data Capture**: PostgreSQL logical replication to feed Elasticsearch/Neo4j
- **Choreography**: Services react to events independently (e.g., NLP service processes new events without central orchestration)

### Recommended Message Broker Progression
1. **MVP**: Redis pub/sub (already deployed for caching)
2. **Scale**: Redis Streams (persistent, consumer groups)
3. **Production**: Apache Kafka (durability, replayability, high throughput)

---

## 10. Deployment Strategy

### Development Environment
```yaml
# docker-compose.yml
services:
  frontend:      # React dev server (Vite)
  backend:       # FastAPI with hot reload
  worker:        # Celery worker
  scheduler:     # Celery Beat
  db:            # PostgreSQL 16 + PostGIS + TimescaleDB
  redis:         # Redis 7
  # Optional:
  pgadmin:       # Database admin UI
  flower:        # Celery monitoring
```

### Production Deployment Options

| Option | Cost | Complexity | Best For |
|---|---|---|---|
| **Single VPS + Docker Compose** | $20-50/mo | Low | MVP, demos, small teams |
| **Managed Services (AWS/GCP)** | $100-300/mo | Medium | Production, reliability |
| **Kubernetes** | $200-500/mo | High | Scale, multi-service |

### Recommended Production Stack (Phase 2)
- **Compute**: AWS ECS Fargate or DigitalOcean App Platform (managed containers)
- **Database**: AWS RDS PostgreSQL with PostGIS (managed, automated backups)
- **Cache**: AWS ElastiCache Redis or managed Redis
- **Storage**: S3 for raw documents and media
- **CDN**: CloudFlare (free tier is excellent)
- **CI/CD**: GitHub Actions (free for public repos)
- **Monitoring**: Sentry (error tracking) + UptimeRobot (availability)
- **SSL**: Let's Encrypt via Caddy or CloudFlare

### Security Considerations
- JWT tokens with short expiry (15min access, 7d refresh)
- Rate limiting on all API endpoints (Redis-backed)
- CORS configuration for frontend origin only
- Input validation via Pydantic models (prevents injection)
- PostgreSQL row-level security for multi-tenant scenarios
- Secrets management via environment variables (never in code)
- HTTPS everywhere (enforce via HSTS)

---

## 11. Comparison Tables Summary

### Full Technology Comparison

| Category | Recommended | Runner-Up | Rejected | Rationale |
|---|---|---|---|---|
| **Frontend Framework** | React + Vite | Next.js | Vue, Angular | Dashboard = SPA; no SEO needed; largest ecosystem |
| **State Management** | Zustand + TanStack Query | Redux Toolkit | MobX, Recoil | Lightweight + excellent server-state caching |
| **Mapping Library** | Mapbox GL JS | Deck.gl | Leaflet, Google Maps | WebGL performance, 3D support, custom styling |
| **Backend Framework** | FastAPI (Python) | Express (Node.js) | Django, Flask | Async, auto-docs, Pydantic, ML ecosystem |
| **Primary Database** | PostgreSQL 16 | -- | MySQL, MariaDB | Extensions ecosystem (PostGIS, TimescaleDB, pgvector) |
| **Geospatial** | PostGIS | -- | MongoDB geospatial | Industry standard, mature, powerful spatial indexing |
| **Time-Series** | TimescaleDB | InfluxDB | QuestDB | SQL-native, same PG instance, better complex queries |
| **Cache/Broker** | Redis 7 | -- | Memcached | Pub/sub + cache + queue broker in one |
| **Search (MVP)** | PostgreSQL FTS + pg_trgm | pg_search (ParadeDB) | Elasticsearch | Zero operational overhead; upgrade path clear |
| **Search (Scale)** | Elasticsearch | Meilisearch | Solr | Gold standard relevance; add only when needed |
| **Graph (Scale)** | Neo4j | -- | ArangoDB | Best graph DB; add only for complex traversals |
| **Task Queue** | Celery | Dramatiq | Bull (Node) | Python ecosystem, mature, Redis broker |
| **Real-Time** | SSE | WebSocket | Long Polling | Server->client only; auto-reconnect; HTTP compatible |
| **NLP** | spaCy + transformers | -- | NLTK, CoreNLP | Fast, production-ready, excellent NER |
| **Containerization** | Docker + Compose | -- | Podman | Industry standard, simple local dev |
| **Orchestration** | Docker Compose (MVP) -> K8s | ECS Fargate | Docker Swarm | Progressive complexity |
| **CI/CD** | GitHub Actions | GitLab CI | Jenkins | Free, integrated, YAML-based |

---

## 12. Key Architectural Decisions Record (ADR)

### ADR-001: PostgreSQL as Primary Polyglot Database
**Decision**: Use PostgreSQL with extensions instead of multiple specialized databases.
**Rationale**: Reduces operational complexity by 80%. One backup strategy, one connection pool, one query language. PostGIS, TimescaleDB, pgvector, and pg_trgm cover geospatial, time-series, vector, and search needs within a single deployment. Specialized databases (Neo4j, Elasticsearch) added only when PostgreSQL demonstrably cannot meet requirements.

### ADR-002: Python/FastAPI Backend
**Decision**: Python over Node.js or Go for backend.
**Rationale**: Python is the dominant language for NLP, ML, data processing, and OSINT tooling. FastAPI's async performance is within 10-20% of Node.js while providing auto-generated API docs, Pydantic validation, and native access to spaCy, transformers, pandas, and geopandas.

### ADR-003: React SPA over Next.js
**Decision**: Plain React SPA with Vite, not Next.js.
**Rationale**: Dashboard applications do not benefit from SSR/SSG. Next.js adds hydration complexity and server-rendering overhead. React SPA gives full control over real-time state management and client-side rendering optimized for interactive dashboards.

### ADR-004: SSE over WebSockets for Real-Time
**Decision**: Server-Sent Events as primary real-time transport.
**Rationale**: Dashboard data flow is exclusively server-to-client. SSE provides automatic reconnection, HTTP/2 compatibility, firewall friendliness, and simpler infrastructure. WebSockets reserved for future bidirectional features (collaboration).

### ADR-005: Monolith-First Architecture
**Decision**: Start with a monolithic FastAPI application, extract microservices only when needed.
**Rationale**: Premature microservices decomposition adds network complexity, deployment overhead, and debugging difficulty without clear benefit at MVP scale. The monolith can be cleanly separated later because FastAPI's router/dependency injection system naturally creates module boundaries.

---

## 13. Risk Mitigation

| Risk | Mitigation |
|---|---|
| PostgreSQL FTS insufficient at scale | Clear upgrade path to pg_search or Elasticsearch; abstracted search interface |
| Single database bottleneck | Read replicas, connection pooling (PgBouncer), TimescaleDB compression |
| Real-time latency | Redis pub/sub provides <10ms publish-to-client; SSE connection pooling |
| NLP accuracy | Start with spaCy (fast, decent); upgrade to transformer models per-entity-type |
| Data source reliability | Multiple redundant sources per region; health monitoring; graceful degradation |
| Frontend performance with large datasets | Virtual scrolling, pagination, WebGL rendering (Mapbox/Deck.gl), query-level aggregation |

---

## Sources

- [Palantir Ontology Overview](https://www.palantir.com/docs/foundry/ontology/overview)
- [Palantir Ontology Architecture](https://www.palantir.com/docs/foundry/architecture-center/ontology-system)
- [Understanding Palantir's Ontology Layers](https://pythonebasta.medium.com/understanding-palantirs-ontology-semantic-kinetic-and-dynamic-layers-explained-c1c25b39ea3c)
- [Confluent: Real-Time Streaming Architecture](https://www.confluent.io/learn/real-time-streaming-architecture-examples/)
- [AWS: Streaming Architecture Patterns](https://docs.aws.amazon.com/whitepapers/latest/build-modern-data-streaming-analytics-architectures/streaming-analytics-architecture-patterns-using-a-modern-data-architecture.html)
- [Next.js vs React Comparison 2025](https://www.tactionsoft.com/guide/next-js-vs-react-comparison/)
- [React vs Next.js 2026 Guide](https://sam-solutions.com/blog/react-vs-nextjs/)
- [TimescaleDB vs InfluxDB](https://www.timescale.com/blog/timescaledb-vs-influxdb-for-time-series-data-timescale-influx-sql-nosql-36489299877)
- [ClickHouse vs TimescaleDB vs InfluxDB 2025 Benchmarks](https://sanj.dev/post/clickhouse-timescaledb-influxdb-time-series-comparison)
- [InfluxDB vs TimescaleDB Detailed Comparison](https://risingwave.com/blog/influxdb-vs-timescale-a-detailed-comparison/)
- [Neo4j Knowledge Graph](https://neo4j.com/use-cases/knowledge-graph/)
- [Neo4j Graph Intelligence Platform](https://research.isg-one.com/analyst-perspectives/neo4j-expands-data-platform-for-graph-intelligence)
- [PostGIS Spatial Queries Documentation](https://postgis.net/docs/using_postgis_query.html)
- [PostgreSQL Geospatial Applications](https://dev.to/pawnsapprentice/postgresql-in-geospatial-applications-unleashing-the-power-of-location-data-4jan)
- [Postgres vs Elasticsearch Full-Text Search](https://www.myscale.com/blog/postgres-vs-elasticsearch-comparison-full-text-search/)
- [Elasticsearch vs Postgres Alternatives (ParadeDB)](https://www.paradedb.com/blog/elasticsearch-vs-postgres)
- [Postgres FTS vs Elasticsearch (Xata)](https://xata.io/blog/postgres-full-text-search-postgres-vs-elasticsearch)
- [Neon: Postgres vs ElasticSearch vs pg_search](https://neon.com/blog/postgres-full-text-search-vs-elasticsearch)
- [SSE Beat WebSockets for 95% of Apps](https://dev.to/polliog/server-sent-events-beat-websockets-for-95-of-real-time-apps-heres-why-a4l)
- [WebSockets vs SSE Comparison (Ably)](https://ably.com/blog/websockets-vs-sse)
- [WebSocket vs SSE Performance (Timeplus)](https://www.timeplus.com/post/websocket-vs-sse)
- [Dataminr AI Platform](https://www.dataminr.com/ai-platform/)
- [OSINT Market Analysis (Mordor Intelligence)](https://www.mordorintelligence.com/industry-reports/open-source-intelligence-market)
- [Event-Driven Architecture Patterns](https://microservices.io/patterns/data/event-driven-architecture.html)
- [Event-Driven Architecture (Confluent)](https://www.confluent.io/learn/event-driven-architecture/)
- [Azure Event-Driven Architecture Style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
