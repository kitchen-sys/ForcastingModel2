# Build Plan 13: Backend API — Coding Agent Instructions

## Executive Summary

This is a step-by-step build plan for the FastAPI backend of Sentinel, a Palantir-inspired global conflict intelligence dashboard. A coding agent should follow this document sequentially to produce a working backend. All architecture decisions, schemas, and endpoint specs come from completed research (findings-09, findings-10, findings-12).

## Research Scope

Based on: findings-09-database-decision.md, findings-10-mvp-scope.md, findings-12-implementation-roadmap.md

---

## 1. Directory Structure

Create exactly this tree under `/home/user/ForcastingModel2/src/backend/`:

```
src/backend/
├── pyproject.toml
├── alembic.ini
├── Dockerfile
├── .env.example
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app factory, CORS, lifespan
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py              # Pydantic BaseSettings
│   │   ├── database.py            # Async SQLAlchemy engine + session
│   │   ├── redis.py               # Redis connection pool
│   │   └── dependencies.py        # get_db, get_redis FastAPI deps
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py                # DeclarativeBase + mixins
│   │   ├── event.py               # Event model (PostGIS, hypertable)
│   │   └── source.py              # Source/Feed model
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── event.py               # EventCreate, EventRead, EventFilter
│   │   ├── source.py              # SourceCreate, SourceRead, SourceHealth
│   │   └── common.py              # PaginatedResponse, ErrorResponse
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── events.py              # /api/v1/events/*
│   │   ├── sources.py             # /api/v1/sources/*
│   │   ├── health.py              # /api/v1/health
│   │   └── stream.py              # /api/v1/ws/events (WebSocket)
│   └── services/
│       ├── __init__.py
│       ├── event_service.py       # Event CRUD, filtering, aggregation
│       ├── source_service.py      # Source CRUD, health tracking
│       └── stream_service.py      # Redis pub/sub → WebSocket broadcast
├── migrations/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_events.py
    ├── test_sources.py
    └── fixtures/
        ├── sample_events.json
        └── sample_rss.xml
```

---

## 2. Dependencies (pyproject.toml)

```toml
[project]
name = "sentinel-backend"
version = "0.1.0"
description = "Sentinel — Global conflict intelligence backend"
requires-python = ">=3.12"

dependencies = [
    "fastapi>=0.115.0,<1.0",
    "uvicorn[standard]>=0.32.0,<1.0",
    "sqlalchemy[asyncio]>=2.0.36,<2.1",
    "asyncpg>=0.30.0,<1.0",
    "alembic>=1.14.0,<2.0",
    "geoalchemy2>=0.15.0,<1.0",
    "redis>=5.2.0,<6.2",
    "celery[redis]>=5.4.0,<6.0",
    "pydantic>=2.10.0,<3.0",
    "pydantic-settings>=2.6.0,<3.0",
    "feedparser>=6.0.11,<7.0",
    "httpx>=0.28.0,<1.0",
    "datasketch>=1.6.0,<2.0",
    "python-jose[cryptography]>=3.3.0,<4.0",
    "passlib[bcrypt]>=1.7.4,<2.0",
    "python-dateutil>=2.9.0,<3.0",
    "orjson>=3.10.0,<4.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0,<9.0",
    "pytest-asyncio>=0.24.0,<1.0",
    "pytest-cov>=6.0.0,<7.0",
    "httpx>=0.28.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"

[tool.ruff]
target-version = "py312"
line-length = 100
```

---

## 3. Core Configuration

### 3.1 config.py — Environment Settings

```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Database
    database_url: str = "postgresql+asyncpg://sentinel:sentinel_dev@localhost:5432/sentinel"
    database_url_sync: str = "postgresql://sentinel:sentinel_dev@localhost:5432/sentinel"

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    # App
    app_name: str = "Sentinel"
    app_env: str = "development"
    log_level: str = "INFO"
    cors_origins: str = "http://localhost:3000"

    # Security
    secret_key: str = "change-in-production"
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30

    # Polling
    default_poll_interval_minutes: int = 60
    min_poll_interval_minutes: int = 5
    max_poll_interval_minutes: int = 1440
    max_consecutive_errors: int = 5

    model_config = {"env_file": ".env", "case_sensitive": False}

settings = Settings()
```

### 3.2 database.py — Async SQLAlchemy

```python
# app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.core.config import settings

engine = create_async_engine(
    settings.database_url,
    echo=(settings.app_env == "development"),
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
)

async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session
```

### 3.3 redis.py

```python
# app/core/redis.py
import redis.asyncio as redis
from app.core.config import settings

redis_pool = redis.ConnectionPool.from_url(settings.redis_url)

async def get_redis() -> redis.Redis:
    return redis.Redis(connection_pool=redis_pool)
```

---

## 4. Database Schema (SQL)

### 4.1 Sources Table

```sql
CREATE TABLE sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL UNIQUE,
    source_type VARCHAR(20) NOT NULL DEFAULT 'rss',  -- 'rss', 'api', 'scraper'
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    poll_interval_minutes INTEGER NOT NULL DEFAULT 60,
    last_poll_at TIMESTAMPTZ,
    last_poll_status VARCHAR(20),  -- 'success', 'error', 'not_modified'
    last_poll_error TEXT,
    etag VARCHAR(255),
    last_modified VARCHAR(255),
    total_items_ingested INTEGER NOT NULL DEFAULT 0,
    error_count INTEGER NOT NULL DEFAULT 0,
    consecutive_errors INTEGER NOT NULL DEFAULT 0,
    next_poll_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reliability_score FLOAT NOT NULL DEFAULT 0.5,
    config JSONB DEFAULT '{}',  -- source-specific config (API keys, query params)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sources_next_poll ON sources (next_poll_at) WHERE enabled = TRUE;
CREATE INDEX idx_sources_type ON sources (source_type);
```

### 4.2 Events Table

```sql
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    description TEXT,
    event_type VARCHAR(100),  -- 'Battles', 'Explosions/Remote violence', 'Protests', etc.
    event_date DATE NOT NULL,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    geo GEOGRAPHY(POINT, 4326),  -- PostGIS geography column
    country VARCHAR(100),
    region VARCHAR(255),
    actors TEXT[],  -- PostgreSQL array
    fatalities INTEGER DEFAULT 0,
    source_name VARCHAR(255),
    source_url TEXT,
    source_id UUID REFERENCES sources(id) ON DELETE SET NULL,
    source_type VARCHAR(20),  -- 'rss', 'api', 'acled', 'gdelt'
    confidence_score FLOAT DEFAULT 0.5,
    content_hash VARCHAR(64),  -- SHA-256 for dedup
    raw_data JSONB,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Convert to TimescaleDB hypertable (partitioned by ingested_at)
SELECT create_hypertable('events', 'ingested_at', migrate_data => true);

-- Indexes
CREATE INDEX idx_events_date ON events (event_date DESC);
CREATE INDEX idx_events_type ON events (event_type);
CREATE INDEX idx_events_country ON events (country);
CREATE INDEX idx_events_geo ON events USING GIST (geo);
CREATE INDEX idx_events_content_hash ON events (content_hash);
CREATE INDEX idx_events_search ON events USING GIN (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, '')));
```

### 4.3 Source Rankings Table

```sql
CREATE TABLE source_rankings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uptime_score FLOAT NOT NULL DEFAULT 0.0,       -- 0-1: successful polls / total polls
    freshness_score FLOAT NOT NULL DEFAULT 0.0,     -- 0-1: how often feed has new content
    relevance_score FLOAT NOT NULL DEFAULT 0.0,     -- 0-1: conflict-relevant items / total items
    quality_score FLOAT NOT NULL DEFAULT 0.0,       -- 0-1: avg content richness
    composite_score FLOAT NOT NULL DEFAULT 0.0,     -- Weighted average of above
    items_last_24h INTEGER NOT NULL DEFAULT 0,
    errors_last_24h INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms INTEGER,
    UNIQUE(source_id, calculated_at)
);

SELECT create_hypertable('source_rankings', 'calculated_at', migrate_data => true);
CREATE INDEX idx_rankings_source ON source_rankings (source_id, calculated_at DESC);
```

---

## 5. Pydantic Schemas

### 5.1 Event Schemas

```python
# app/schemas/event.py
from pydantic import BaseModel, Field
from datetime import date, datetime
from uuid import UUID

class EventRead(BaseModel):
    id: UUID
    title: str
    description: str | None = None
    event_type: str | None = None
    event_date: date
    latitude: float | None = None
    longitude: float | None = None
    country: str | None = None
    region: str | None = None
    actors: list[str] = []
    fatalities: int = 0
    source_name: str | None = None
    source_url: str | None = None
    source_type: str | None = None
    confidence_score: float = 0.5
    ingested_at: datetime

    model_config = {"from_attributes": True}

class EventCreate(BaseModel):
    title: str
    description: str | None = None
    event_type: str | None = None
    event_date: date
    latitude: float | None = None
    longitude: float | None = None
    country: str | None = None
    region: str | None = None
    actors: list[str] = []
    fatalities: int = 0
    source_name: str | None = None
    source_url: str | None = None
    source_type: str | None = None
    confidence_score: float = 0.5
    raw_data: dict | None = None

class EventFilter(BaseModel):
    date_from: date | None = None
    date_to: date | None = None
    event_type: list[str] | None = None
    country: list[str] | None = None
    bbox: str | None = None  # "minLng,minLat,maxLng,maxLat"
    search: str | None = None
    sort: str = "-event_date"
    limit: int = Field(default=50, ge=1, le=500)
    offset: int = Field(default=0, ge=0)

class ClusterRead(BaseModel):
    cluster_id: int | None = None
    latitude: float
    longitude: float
    count: int
    dominant_type: str | None = None
    expansion_zoom: int | None = None
    event_id: UUID | None = None
    event_type: str | None = None
    title: str | None = None

class TimelineBucket(BaseModel):
    date: str
    count: int
    fatalities: int = 0
```

### 5.2 Source Schemas

```python
# app/schemas/source.py
from pydantic import BaseModel, HttpUrl
from datetime import datetime
from uuid import UUID

class SourceCreate(BaseModel):
    name: str
    url: str
    source_type: str = "rss"
    poll_interval_minutes: int = 60
    config: dict = {}

class SourceRead(BaseModel):
    id: UUID
    name: str
    url: str
    source_type: str
    enabled: bool
    poll_interval_minutes: int
    last_poll_at: datetime | None = None
    last_poll_status: str | None = None
    last_poll_error: str | None = None
    total_items_ingested: int
    error_count: int
    consecutive_errors: int
    reliability_score: float
    created_at: datetime

    model_config = {"from_attributes": True}

class SourceUpdate(BaseModel):
    name: str | None = None
    url: str | None = None
    enabled: bool | None = None
    poll_interval_minutes: int | None = None
    config: dict | None = None
```

### 5.3 Common Schemas

```python
# app/schemas/common.py
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    data: list[T]
    meta: dict  # {"total": int, "limit": int, "offset": int}

class SingleResponse(BaseModel, Generic[T]):
    data: T

class HealthResponse(BaseModel):
    status: str
    db: str
    redis: str
    sources_active: int

class StatsResponse(BaseModel):
    total_events: int
    events_today: int
    active_sources: int
    countries_covered: int
```

---

## 6. API Endpoints — Full Specification

### 6.1 Events Router

```python
# app/routers/events.py

# GET /api/v1/events — List events with filters
# Query params: date_from, date_to, event_type, country, bbox, search, sort, limit, offset
# Response: PaginatedResponse[EventRead]

# GET /api/v1/events/{id} — Get single event
# Response: SingleResponse[EventRead]

# GET /api/v1/events/export — Export filtered events as CSV
# Same query params as list
# Response: StreamingResponse (CSV file)

# GET /api/v1/events/clusters — Clustered markers for map
# Query: bbox, zoom, date_from, date_to, event_type, country
# Response: {"data": list[ClusterRead]}

# GET /api/v1/events/timeline — Aggregated counts for timeline chart
# Query: date_from, date_to, interval (day/week/month), event_type, country
# Response: {"data": list[TimelineBucket], "meta": {...}}
```

### 6.2 Sources Router

```python
# app/routers/sources.py

# GET    /api/v1/sources          — List all sources         → list[SourceRead]
# POST   /api/v1/sources          — Add new source           → SourceRead
# PATCH  /api/v1/sources/{id}     — Update source            → SourceRead
# DELETE /api/v1/sources/{id}     — Delete source            → {"success": true}
# POST   /api/v1/sources/{id}/poll — Force immediate poll    → {"status": "polling_started"}
# GET    /api/v1/sources/logs     — Recent ingestion logs    → list[LogEntry]
```

### 6.3 Health Router

```python
# app/routers/health.py

# GET /api/v1/health — System health check
# Response: {"status": "ok", "db": "ok", "redis": "ok", "sources_active": 6}

# GET /api/v1/stats — Dashboard summary
# Response: {"total_events": 12453, "events_today": 167, "active_sources": 6, "countries_covered": 42}
```

### 6.4 WebSocket Stream (Tier 2)

```python
# app/routers/stream.py

# WebSocket /api/v1/ws/events
# Server pushes:  {"type": "new_event", "data": EventRead}
# Client sends:   {"type": "subscribe", "filters": {...}} (optional filter)
```

---

## 7. Service Layer Function Signatures

```python
# app/services/event_service.py

async def list_events(db: AsyncSession, filters: EventFilter) -> tuple[list[Event], int]:
    """Query events with filters, return (events, total_count)."""

async def get_event(db: AsyncSession, event_id: UUID) -> Event | None:
    """Get single event by ID."""

async def create_event(db: AsyncSession, event: EventCreate) -> Event:
    """Insert a new event. Compute content_hash and geo point."""

async def bulk_create_events(db: AsyncSession, events: list[EventCreate]) -> int:
    """Bulk insert events. Returns count of inserted (deduped)."""

async def get_clusters(db: AsyncSession, bbox: str, zoom: int, filters: EventFilter) -> list[dict]:
    """Server-side clustering using ST_ClusterDBSCAN or grid-based approach."""

async def get_timeline(db: AsyncSession, filters: EventFilter, interval: str) -> list[dict]:
    """Aggregate event counts by time bucket using TimescaleDB time_bucket()."""

async def export_csv(db: AsyncSession, filters: EventFilter) -> AsyncGenerator[str, None]:
    """Stream CSV rows for filtered events."""
```

```python
# app/services/source_service.py

async def list_sources(db: AsyncSession, active_only: bool = False) -> list[Source]:
    """List all registered data sources."""

async def create_source(db: AsyncSession, source: SourceCreate) -> Source:
    """Register a new data source."""

async def update_source(db: AsyncSession, source_id: UUID, data: SourceUpdate) -> Source:
    """Update source settings."""

async def delete_source(db: AsyncSession, source_id: UUID) -> bool:
    """Remove a data source."""

async def trigger_poll(source_id: UUID) -> str:
    """Send Celery task to poll source immediately. Returns task ID."""

async def get_ingestion_logs(db: AsyncSession, source_id: UUID | None, limit: int) -> list[dict]:
    """Get recent ingestion log entries."""
```

---

## 8. main.py — App Factory

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings
from app.routers import events, sources, health, stream

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: verify DB connection, Redis connection
    yield
    # Shutdown: close pools

app = FastAPI(
    title=settings.app_name,
    version="0.1.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(events.router, prefix="/api/v1", tags=["events"])
app.include_router(sources.router, prefix="/api/v1", tags=["sources"])
app.include_router(health.router, prefix="/api/v1", tags=["health"])
app.include_router(stream.router, prefix="/api/v1", tags=["stream"])
```

---

## 9. Environment Variables (.env.example)

```bash
DATABASE_URL=postgresql+asyncpg://sentinel:sentinel_dev@localhost:5432/sentinel
DATABASE_URL_SYNC=postgresql://sentinel:sentinel_dev@localhost:5432/sentinel
REDIS_URL=redis://localhost:6379/0
CELERY_BROKER_URL=redis://localhost:6379/1
CELERY_RESULT_BACKEND=redis://localhost:6379/2
SECRET_KEY=change-this-in-production
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
APP_NAME=Sentinel
APP_ENV=development
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000
DEFAULT_POLL_INTERVAL_MINUTES=60
MIN_POLL_INTERVAL_MINUTES=5
MAX_POLL_INTERVAL_MINUTES=1440
MAX_CONSECUTIVE_ERRORS=5
```

---

## 10. Agent Build Instructions — Step by Step

A coding agent should follow these steps in order:

### Step 1: Create directory structure
Create all directories and empty `__init__.py` files.

### Step 2: Write pyproject.toml
Copy the dependencies section exactly.

### Step 3: Write .env.example and Dockerfile
These are static files — copy from above.

### Step 4: Write app/core/ modules
1. `config.py` — Settings class
2. `database.py` — Engine and session
3. `redis.py` — Redis pool
4. `dependencies.py` — FastAPI dependency injection

### Step 5: Write app/models/
1. `base.py` — DeclarativeBase with UUID pk mixin and timestamp mixin
2. `source.py` — Source ORM model matching the SQL schema
3. `event.py` — Event ORM model with GeoAlchemy2 Geography column

### Step 6: Write app/schemas/
1. `common.py` — PaginatedResponse, HealthResponse, StatsResponse
2. `event.py` — EventCreate, EventRead, EventFilter, ClusterRead, TimelineBucket
3. `source.py` — SourceCreate, SourceRead, SourceUpdate

### Step 7: Write app/services/
1. `event_service.py` — All event query logic
2. `source_service.py` — Source CRUD + Celery task trigger
3. `stream_service.py` — Redis pub/sub listener

### Step 8: Write app/routers/
1. `health.py` — Simple, test first
2. `sources.py` — CRUD endpoints
3. `events.py` — List, detail, export, clusters, timeline
4. `stream.py` — WebSocket endpoint

### Step 9: Write app/main.py
Wire everything together.

### Step 10: Set up Alembic
```bash
alembic init migrations
# Edit alembic.ini to use DATABASE_URL_SYNC
# Edit migrations/env.py to import models
# Generate initial migration
alembic revision --autogenerate -m "initial schema"
```

### Step 11: Write tests
1. `conftest.py` — Test DB, test client fixtures
2. `test_events.py` — Event CRUD tests
3. `test_sources.py` — Source CRUD tests

### Verification Checkpoint
After building, run:
```bash
docker compose up -d db redis
alembic upgrade head
uvicorn app.main:app --reload
# Test: curl http://localhost:8000/api/v1/health
# Expect: {"status": "ok", "db": "ok", "redis": "ok", "sources_active": 0}
```

---

## Comparison Tables

### ORM Library Choice

| Criteria | SQLAlchemy 2.0 Async | Tortoise ORM | Raw asyncpg |
|----------|---------------------|--------------|-------------|
| Maturity | Very High | Medium | High |
| PostGIS support | GeoAlchemy2 | None | Manual |
| Migration tool | Alembic | Aerich | Manual |
| Community | Largest | Smaller | N/A |
| **Recommendation** | **YES** | No | Fallback |

### API Framework Choice

| Criteria | FastAPI | Django REST | Flask |
|----------|---------|-------------|-------|
| Async native | Yes | Partial | No |
| Auto docs | Swagger + ReDoc | Browsable API | Manual |
| WebSocket | Native | Channels (addon) | Flask-SocketIO |
| Type safety | Pydantic | Serializers | Manual |
| **Recommendation** | **YES** | No | No |

---

## Priority Implementation Order

1. **Core config + database setup** — Everything depends on this
2. **Models + Alembic migrations** — Schema must exist before any data
3. **Health endpoint** — Verify the stack is working
4. **Source CRUD endpoints** — Need to register feeds before polling
5. **Event CRUD + filtering** — Core data access layer
6. **Timeline aggregation** — Powers the chart component
7. **Cluster endpoint** — Powers the map component
8. **CSV export** — Analyst requirement
9. **WebSocket stream** — Tier 2, after core REST works

---

## Open Questions

1. **Auth timing**: When to add JWT auth? Defer to Tier 3 or add skeleton now?
2. **Server-side clustering**: ST_ClusterDBSCAN vs Supercluster on frontend? Need benchmarking.
3. **TimescaleDB continuous aggregates**: Pre-compute timeline data or query live?
4. **Connection pooling**: PgBouncer needed for MVP or only at scale?
