# Findings: Implementation Roadmap — Phase 2 & 3 Build Plan

## Executive Summary

This document is the bridge from research to code for Sentinel. It provides the exact directory structure, pinned dependency versions, Docker Compose configuration, step-by-step component breakdown for the ingestion pipeline, FastAPI project architecture, testing strategy, and a three-sprint plan with specific files and functions to build. Every recommendation is grounded in decisions from findings-02 (architecture), findings-06 (RSS ingestion), and findings-08 (anti-patterns). A developer reading this document should be able to open a terminal and start building immediately.

## Research Scope

This document responds to Research Brief 12, covering:
1. Project scaffolding — exact directory tree, dependency files, Docker Compose, environment variables
2. Data ingestion pipeline — five components broken down step by step
3. Backend API — FastAPI project structure with routers, models, services, schemas
4. Testing strategy — what to test, pytest fixtures, RSS feed mocking
5. Sprint plan — three sprints with specific files, functions, and definitions of done

---

## Detailed Findings

### 1. Project Scaffolding

#### 1.1 Codebase Directory Structure

The project uses a **feature-organized modular monolith** pattern inside FastAPI, with a separate Next.js frontend. This structure follows the 2025 consensus on FastAPI best practices: separate routers (thin), services (business logic), models (ORM), and schemas (Pydantic) per domain.

```
/home/user/ForcastingModel2/
├── CLAUDE.md
├── AGENT_RULES.md
├── research/                          # Research artifacts (existing)
│
├── src/
│   ├── backend/                       # Python FastAPI backend
│   │   ├── pyproject.toml             # Python dependencies (pinned)
│   │   ├── alembic.ini                # Alembic migration config
│   │   ├── Dockerfile                 # Backend Docker image
│   │   ├── .env.example               # Environment variable template
│   │   │
│   │   ├── app/
│   │   │   ├── __init__.py
│   │   │   ├── main.py                # FastAPI app factory, middleware, lifespan
│   │   │   │
│   │   │   ├── core/                  # Cross-cutting concerns
│   │   │   │   ├── __init__.py
│   │   │   │   ├── config.py          # Pydantic BaseSettings (env vars)
│   │   │   │   ├── database.py        # Async SQLAlchemy engine + session factory
│   │   │   │   ├── redis.py           # Redis connection pool
│   │   │   │   ├── security.py        # JWT token creation/validation
│   │   │   │   └── dependencies.py    # Shared FastAPI dependencies (get_db, get_redis)
│   │   │   │
│   │   │   ├── models/                # SQLAlchemy ORM models
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py            # DeclarativeBase, common mixins (timestamps, id)
│   │   │   │   ├── event.py           # Event model (PostGIS geometry, hypertable)
│   │   │   │   ├── source.py          # Source/Feed model
│   │   │   │   ├── actor.py           # Actor model
│   │   │   │   ├── location.py        # Location model (PostGIS)
│   │   │   │   └── conflict.py        # Conflict model
│   │   │   │
│   │   │   ├── schemas/               # Pydantic request/response schemas
│   │   │   │   ├── __init__.py
│   │   │   │   ├── event.py           # EventCreate, EventRead, EventList
│   │   │   │   ├── source.py          # SourceCreate, SourceRead, SourceHealth
│   │   │   │   ├── actor.py           # ActorCreate, ActorRead
│   │   │   │   ├── search.py          # SearchQuery, SearchResult
│   │   │   │   └── common.py          # PaginatedResponse, ErrorResponse
│   │   │   │
│   │   │   ├── routers/               # Thin API route handlers
│   │   │   │   ├── __init__.py
│   │   │   │   ├── events.py          # /api/v1/events/*
│   │   │   │   ├── sources.py         # /api/v1/sources/*
│   │   │   │   ├── actors.py          # /api/v1/actors/*
│   │   │   │   ├── search.py          # /api/v1/search/*
│   │   │   │   ├── health.py          # /api/v1/health (liveness/readiness)
│   │   │   │   └── stream.py          # /api/v1/stream/* (SSE endpoints)
│   │   │   │
│   │   │   ├── services/              # Business logic layer
│   │   │   │   ├── __init__.py
│   │   │   │   ├── event_service.py   # Event CRUD, filtering, aggregation
│   │   │   │   ├── source_service.py  # Source CRUD, health checks
│   │   │   │   ├── search_service.py  # Full-text search via PostgreSQL tsvector
│   │   │   │   └── stream_service.py  # SSE event broadcasting via Redis pub/sub
│   │   │   │
│   │   │   └── ingestion/             # Data ingestion pipeline
│   │   │       ├── __init__.py
│   │   │       ├── celery_app.py      # Celery app configuration
│   │   │       ├── feed_registry.py   # Feed CRUD operations
│   │   │       ├── feed_poller.py     # Celery tasks: fetch feeds with ETags
│   │   │       ├── feed_parser.py     # Parse with fastfeedparser/feedparser fallback
│   │   │       ├── dedup.py           # 3-tier dedup: SHA256 + URL norm + MinHash LSH
│   │   │       ├── normalizer.py      # Normalize feed entries to Event schema
│   │   │       └── scheduler.py       # Celery Beat dynamic schedule (adaptive intervals)
│   │   │
│   │   ├── migrations/                # Alembic migrations
│   │   │   ├── env.py
│   │   │   ├── script.py.mako
│   │   │   └── versions/              # Migration scripts
│   │   │
│   │   └── tests/
│   │       ├── __init__.py
│   │       ├── conftest.py            # Shared fixtures (async db, test client, mock feeds)
│   │       ├── test_events.py         # Event API tests
│   │       ├── test_sources.py        # Source API tests
│   │       ├── test_ingestion.py      # Feed poller + parser + dedup tests
│   │       ├── test_search.py         # Search endpoint tests
│   │       └── fixtures/
│   │           ├── sample_rss.xml     # Well-formed test RSS feed
│   │           ├── malformed_rss.xml  # Malformed feed for error handling tests
│   │           └── sample_events.json # Seed event data
│   │
│   └── frontend/                      # TypeScript Next.js frontend (Phase 4)
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── Dockerfile
│       └── src/
│           ├── app/                   # Next.js App Router
│           │   ├── layout.tsx         # Root layout (dark theme, fonts)
│           │   ├── page.tsx           # Dashboard home
│           │   ├── events/
│           │   │   └── page.tsx       # Event list view
│           │   └── map/
│           │       └── page.tsx       # Map view
│           ├── components/            # Reusable UI components
│           │   ├── ui/               # Base design system (shadcn/ui)
│           │   ├── map/              # Mapbox GL components
│           │   ├── charts/           # Recharts wrappers
│           │   └── events/           # Event list, detail, filters
│           ├── lib/                   # Utilities, API client, types
│           │   ├── api.ts            # Fetch wrapper for backend API
│           │   ├── sse.ts            # SSE client for real-time streams
│           │   └── types.ts          # Shared TypeScript interfaces
│           └── stores/               # Zustand state stores
│               ├── event-store.ts
│               └── filter-store.ts
│
├── docker/
│   ├── docker-compose.yml             # Local dev environment
│   ├── docker-compose.prod.yml        # Production overrides
│   └── postgres/
│       └── init.sql                   # PostGIS + pg_trgm extension setup
│
└── docs/                              # Documentation (created as needed)
```

#### 1.2 Python Dependencies (`pyproject.toml`)

Critical compatibility note: Celery 5.6.x depends on Kombu, which constrains `redis<6.2`. Therefore, redis-py 7.x cannot be used with Celery. Pin redis to `>=5.2.0,<6.2`.

```toml
[project]
name = "sentinel-backend"
version = "0.1.0"
description = "Sentinel — Global conflict intelligence backend"
requires-python = ">=3.12"

dependencies = [
    # Web framework
    "fastapi>=0.128.0,<0.129",
    "uvicorn[standard]>=0.32.0,<0.33",
    "gunicorn>=23.0.0,<24",

    # Database
    "sqlalchemy[asyncio]>=2.0.46,<2.1",
    "asyncpg>=0.31.0,<0.32",
    "alembic>=1.14.0,<1.15",
    "geoalchemy2>=0.17.0,<0.18",

    # Cache and message broker
    "redis>=5.2.0,<6.2",

    # Task queue
    "celery[redis]>=5.6.0,<5.7",
    "redbeat>=2.2.0,<3",

    # Data validation
    "pydantic>=2.10.0,<3",
    "pydantic-settings>=2.6.0,<3",

    # RSS parsing
    "feedparser>=6.0.11,<7",
    "fastfeedparser>=0.4.5,<0.5",

    # HTTP client
    "httpx>=0.28.0,<0.29",

    # Deduplication
    "datasketch>=1.6.0,<2",
    "url-normalize>=2.2.0,<3",

    # Security
    "python-jose[cryptography]>=3.3.0,<4",
    "passlib[bcrypt]>=1.7.4,<2",

    # NLP (Phase 5 - include early for schema planning)
    "spacy>=3.8.0,<4",

    # Utilities
    "python-dateutil>=2.9.0,<3",
    "orjson>=3.10.0,<4",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0,<9",
    "pytest-asyncio>=0.24.0,<1",
    "pytest-cov>=6.0.0,<7",
    "httpx>=0.28.0",
    "factory-boy>=3.3.0,<4",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "pre-commit>=4.0.0",
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

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy"]
```

#### 1.3 Frontend Dependencies (`package.json`)

The frontend is Phase 4 work. This is included for planning and to lock dependency decisions.

```json
{
  "name": "sentinel-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zustand": "^5.0.0",
    "@tanstack/react-query": "^5.62.0",
    "mapbox-gl": "^3.9.0",
    "recharts": "^2.15.0",
    "date-fns": "^4.1.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.6.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "tailwindcss": "^4.0.0",
    "@tailwindcss/postcss": "^4.0.0",
    "postcss": "^8.5.0",
    "eslint": "^9.17.0",
    "eslint-config-next": "^15.1.0",
    "@types/mapbox-gl": "^3.4.0"
  }
}
```

#### 1.4 Docker Compose for Local Development

```yaml
# docker/docker-compose.yml
version: "3.9"

services:
  # --- PostgreSQL with PostGIS ---
  db:
    image: postgis/postgis:16-3.5
    container_name: sentinel-db
    environment:
      POSTGRES_USER: sentinel
      POSTGRES_PASSWORD: sentinel_dev
      POSTGRES_DB: sentinel
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sentinel"]
      interval: 5s
      timeout: 5s
      retries: 5

  # --- Redis ---
  redis:
    image: redis:7-alpine
    container_name: sentinel-redis
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # --- FastAPI Backend ---
  backend:
    build:
      context: ../src/backend
      dockerfile: Dockerfile
    container_name: sentinel-backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - "8000:8000"
    volumes:
      - ../src/backend:/app
    env_file:
      - ../src/backend/.env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  # --- Celery Worker ---
  celery-worker:
    build:
      context: ../src/backend
      dockerfile: Dockerfile
    container_name: sentinel-celery-worker
    command: celery -A app.ingestion.celery_app worker --loglevel=info --concurrency=4
    volumes:
      - ../src/backend:/app
    env_file:
      - ../src/backend/.env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  # --- Celery Beat Scheduler ---
  celery-beat:
    build:
      context: ../src/backend
      dockerfile: Dockerfile
    container_name: sentinel-celery-beat
    command: celery -A app.ingestion.celery_app beat --loglevel=info -S redbeat.RedBeatScheduler
    volumes:
      - ../src/backend:/app
    env_file:
      - ../src/backend/.env
    depends_on:
      redis:
        condition: service_healthy

  # --- Flower (Celery monitoring, optional) ---
  flower:
    build:
      context: ../src/backend
      dockerfile: Dockerfile
    container_name: sentinel-flower
    command: celery -A app.ingestion.celery_app flower --port=5555
    ports:
      - "5555:5555"
    env_file:
      - ../src/backend/.env
    depends_on:
      redis:
        condition: service_healthy

volumes:
  pgdata:
  redisdata:
```

#### 1.5 PostgreSQL Init Script

```sql
-- docker/postgres/init.sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

#### 1.6 Environment Variables (`.env.example`)

```bash
# Database
DATABASE_URL=postgresql+asyncpg://sentinel:sentinel_dev@db:5432/sentinel
DATABASE_URL_SYNC=postgresql://sentinel:sentinel_dev@db:5432/sentinel

# Redis
REDIS_URL=redis://redis:6379/0

# Celery
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

# Security
SECRET_KEY=change-this-in-production-use-openssl-rand-hex-32
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# App
APP_NAME=Sentinel
APP_ENV=development
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000

# RSS Polling
DEFAULT_POLL_INTERVAL_MINUTES=60
MIN_POLL_INTERVAL_MINUTES=5
MAX_POLL_INTERVAL_MINUTES=1440
MAX_CONSECUTIVE_ERRORS=5
```

#### 1.7 Backend Dockerfile

```dockerfile
# src/backend/Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps for asyncpg, PostGIS bindings, and lxml
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libgeos-dev \
    libproj-dev \
    gdal-bin \
    libgdal-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e ".[dev]"

# Copy application code
COPY . .

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 2. Data Ingestion Pipeline — Step by Step

The ingestion pipeline has five components, each responsible for one stage of the data flow. This follows the architecture from findings-06 and the anti-patterns guidance from findings-08.

```
[Feed Registry] → [Feed Poller] → [Feed Parser] → [Deduplication] → [Event Storage]
     CRUD            Celery task     feedparser       SHA256+MinHash     PostgreSQL
```

#### Component 1: Feed Registry (`feed_registry.py`)

**Purpose:** CRUD operations for managing RSS feed sources.

**Key functions:**
```python
# app/ingestion/feed_registry.py

async def create_source(db: AsyncSession, source: SourceCreate) -> Source:
    """Register a new RSS feed source. Sets default polling interval."""

async def list_sources(db: AsyncSession, active_only: bool = True) -> list[Source]:
    """List all registered sources, optionally filtering to active ones."""

async def update_source(db: AsyncSession, source_id: UUID, data: SourceUpdate) -> Source:
    """Update source metadata (URL, polling interval, active status)."""

async def disable_source(db: AsyncSession, source_id: UUID, reason: str) -> Source:
    """Disable a source (e.g., after max consecutive errors)."""

async def get_sources_due_for_polling(db: AsyncSession) -> list[Source]:
    """Return sources where next_poll_at <= now() and is_active = True."""

async def update_poll_metadata(
    db: AsyncSession, source_id: UUID,
    etag: str | None, last_modified: str | None,
    next_poll_at: datetime, error_count: int
) -> None:
    """Update ETag, Last-Modified, next poll time, and error count after a fetch."""
```

**Database model fields for Source:**
- `id` (UUID, PK)
- `name` (str)
- `url` (str, unique)
- `source_type` (enum: rss, api, scraper)
- `is_active` (bool, default True)
- `poll_interval_minutes` (int, default 60)
- `next_poll_at` (datetime)
- `last_polled_at` (datetime, nullable)
- `etag` (str, nullable)
- `last_modified_header` (str, nullable)
- `consecutive_errors` (int, default 0)
- `reliability_score` (float, default 1.0)
- `created_at`, `updated_at` (timestamps)

#### Component 2: Feed Poller (`feed_poller.py`)

**Purpose:** Celery task that fetches RSS feeds using conditional HTTP requests.

**Key functions:**
```python
# app/ingestion/feed_poller.py

@celery_app.task(
    bind=True,
    autoretry_for=(httpx.ConnectError, httpx.TimeoutException),
    retry_backoff=True,
    retry_backoff_max=3600,
    retry_jitter=True,
    max_retries=3,
    acks_late=True,
)
def poll_feed(self, source_id: str) -> dict:
    """
    Fetch a single RSS feed.
    1. Load source from DB (sync session for Celery)
    2. Send HTTP GET with If-None-Match / If-Modified-Since headers
    3. If 304 Not Modified: update next_poll_at, return early
    4. If 200 OK: pass response body to parse_and_store_feed task
    5. On error: increment consecutive_errors; disable if >= MAX_CONSECUTIVE_ERRORS
    """

@celery_app.task
def dispatch_polling_round() -> int:
    """
    Called by Celery Beat on a fixed interval (e.g., every 2 minutes).
    Queries sources due for polling and dispatches poll_feed tasks.
    Returns count of tasks dispatched.
    """
```

**HTTP client configuration:**
- Use `httpx` (synchronous mode inside Celery workers, since Celery does not natively support async tasks without workarounds)
- Connect timeout: 10 seconds
- Read timeout: 30 seconds
- User-Agent: `Sentinel/1.0 (+https://github.com/sentinel-project)`
- Follow redirects: yes (max 5)

#### Component 3: Feed Parser (`feed_parser.py`)

**Purpose:** Parse RSS/Atom feed XML into normalized Python dicts.

**Key functions:**
```python
# app/ingestion/feed_parser.py

def parse_feed_content(raw_content: bytes, feed_url: str) -> list[dict]:
    """
    Two-tier parsing strategy (from findings-06):
    1. Try fastfeedparser first (10-50x faster)
    2. If fastfeedparser raises an exception, fall back to feedparser
    3. Handle feedparser bozo exceptions (log non-critical, reject critical)
    4. Return list of normalized entry dicts
    """

def normalize_entry(entry: dict, source_id: str) -> dict:
    """
    Normalize a single feed entry to the internal event schema:
    - Extract: title, link, published (parse to UTC datetime), summary (plain text)
    - Generate fingerprint: SHA-256 of (guid or link+title)
    - Strip HTML from summary using built-in html parser
    - Return dict ready for dedup check and DB insertion
    """

IGNORABLE_BOZO_EXCEPTIONS = (
    feedparser.CharacterEncodingOverride,
    feedparser.NonXMLContentType,
)
```

#### Component 4: Deduplication (`dedup.py`)

**Purpose:** Three-tier deduplication to prevent storing duplicate events.

**Key functions:**
```python
# app/ingestion/dedup.py

class DeduplicationService:
    """Three-tier dedup service backed by Redis."""

    def __init__(self, redis_client: Redis):
        self.redis = redis_client
        self.lsh = MinHashLSH(
            threshold=0.5, num_perm=128,
            storage_config={'type': 'redis', 'redis': {'host': 'redis', 'port': 6379}},
        )

    async def is_duplicate(self, entry: dict) -> bool:
        """
        Tier 1: Check SHA-256 fingerprint in Redis set (O(1))
        Tier 2: Normalize URL, check again (catches tracking param variants)
        Tier 3: Compute MinHash of text content, query LSH index (catches near-dupes)
        Returns True if duplicate at any tier.
        """

    async def register_entry(self, entry: dict) -> None:
        """Add entry's fingerprint and MinHash to the index after storage."""

def compute_fingerprint(guid: str | None, link: str, title: str) -> str:
    """SHA-256 hash of guid (preferred) or link+title."""

def normalize_url(url: str) -> str:
    """Strip tracking params (utm_*, fbclid, gclid), lowercase host, sort params."""

def compute_minhash(text: str, num_perm: int = 128) -> MinHash:
    """3-word shingling + MinHash signature generation."""
```

#### Component 5: Event Storage (`normalizer.py` + `event_service.py`)

**Purpose:** Store normalized, deduplicated events in PostgreSQL and notify via Redis pub/sub.

**Key functions:**
```python
# app/ingestion/normalizer.py

def to_event_create(normalized_entry: dict, source: Source) -> EventCreate:
    """
    Convert a normalized feed entry dict to an EventCreate Pydantic schema.
    Sets default severity=1, event_type='news_report'.
    Enrichment (NLP, geolocation) happens asynchronously later.
    """

# app/services/event_service.py

async def create_event(db: AsyncSession, event: EventCreate) -> Event:
    """
    Insert event into PostgreSQL.
    After insert, publish to Redis channel 'events:new' for SSE broadcasting.
    Uses INSERT ... ON CONFLICT DO NOTHING for race condition safety.
    """

async def publish_new_event(redis: Redis, event: Event) -> None:
    """Publish event to Redis pub/sub channel for real-time SSE streaming."""
```

---

### 3. Backend API — FastAPI Project Structure

#### 3.1 Application Factory (`main.py`)

```python
# app/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialize DB connection pool, Redis pool
    await init_db()
    await init_redis()
    yield
    # Shutdown: close pools
    await close_db()
    await close_redis()

def create_app() -> FastAPI:
    app = FastAPI(
        title="Sentinel API",
        version="0.1.0",
        lifespan=lifespan,
    )
    app.add_middleware(CORSMiddleware, allow_origins=settings.CORS_ORIGINS, ...)
    app.include_router(health_router, prefix="/api/v1")
    app.include_router(events_router, prefix="/api/v1")
    app.include_router(sources_router, prefix="/api/v1")
    app.include_router(search_router, prefix="/api/v1")
    app.include_router(stream_router, prefix="/api/v1")
    return app

app = create_app()
```

#### 3.2 Database Connection (`core/database.py`)

**Decision: SQLAlchemy 2.0 async with asyncpg driver.**

Raw asyncpg is 2-3x faster, but SQLAlchemy provides ORM features, GeoAlchemy2 for PostGIS, and Alembic migrations. The developer productivity tradeoff is justified per findings-02. If specific queries become bottlenecks, they can use raw asyncpg via `session.execute(text(...))`.

```python
# app/core/database.py

from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.APP_ENV == "development",
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
)

async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

#### 3.3 Router Pattern (Thin Routers)

Routers should be thin — validate input, call service, return response. No business logic in routers.

```python
# app/routers/events.py

from fastapi import APIRouter, Depends, Query
from app.schemas.event import EventRead, EventList
from app.schemas.common import PaginatedResponse
from app.services.event_service import EventService
from app.core.dependencies import get_db, get_event_service

router = APIRouter(prefix="/events", tags=["events"])

@router.get("/", response_model=PaginatedResponse[EventRead])
async def list_events(
    page: int = Query(1, ge=1),
    per_page: int = Query(50, ge=1, le=200),
    severity_min: int | None = Query(None, ge=1, le=10),
    event_type: str | None = None,
    service: EventService = Depends(get_event_service),
):
    """List events with pagination and optional filters."""
    return await service.list_events(
        page=page, per_page=per_page,
        severity_min=severity_min, event_type=event_type,
    )

@router.get("/{event_id}", response_model=EventRead)
async def get_event(event_id: UUID, service: EventService = Depends(get_event_service)):
    """Get a single event by ID."""
    event = await service.get_event(event_id)
    if not event:
        raise HTTPException(status_code=404, detail="Event not found")
    return event
```

#### 3.4 Service Layer Pattern

```python
# app/services/event_service.py

class EventService:
    def __init__(self, db: AsyncSession, redis: Redis):
        self.db = db
        self.redis = redis

    async def list_events(self, page: int, per_page: int, **filters) -> PaginatedResponse:
        """Build query with filters, apply pagination, return results with total count."""

    async def get_event(self, event_id: UUID) -> Event | None:
        """Fetch single event, check Redis cache first."""

    async def create_event(self, data: EventCreate) -> Event:
        """Insert event, publish to Redis pub/sub, invalidate relevant caches."""

    async def search_events(self, query: str, **filters) -> list[Event]:
        """Full-text search using PostgreSQL tsvector + ts_rank."""
```

#### 3.5 SSE Streaming (`routers/stream.py`)

```python
# app/routers/stream.py

from fastapi import APIRouter
from sse_starlette.sse import EventSourceResponse

router = APIRouter(prefix="/stream", tags=["streaming"])

@router.get("/events")
async def stream_events(request: Request):
    """SSE endpoint for real-time event updates. Subscribes to Redis pub/sub."""
    async def event_generator():
        pubsub = redis.pubsub()
        await pubsub.subscribe("events:new")
        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    yield {"event": "new_event", "data": message["data"]}
        finally:
            await pubsub.unsubscribe("events:new")
    return EventSourceResponse(event_generator())
```

#### 3.6 Authentication Approach (MVP)

For MVP: simple API key authentication. No JWT complexity yet (per findings-08 anti-pattern #4: do not build custom auth for MVP).

```python
# app/core/security.py

from fastapi import Security, HTTPException
from fastapi.security import APIKeyHeader

API_KEY_HEADER = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(API_KEY_HEADER)):
    if api_key != settings.API_KEY:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key
```

Upgrade to JWT + OAuth2 in Sprint 3 or Phase 4 when multi-user support is needed.

---

### 4. Testing Strategy

#### 4.1 What to Test in MVP (Don't Over-Test, Don't Under-Test)

| Category | What to Test | What NOT to Test |
|----------|-------------|-----------------|
| **Ingestion** | Feed parsing (well-formed + malformed), dedup logic, error handling | Celery task scheduling internals |
| **API Endpoints** | CRUD operations, pagination, error responses, input validation | Every possible filter combination |
| **Services** | Business logic (severity filtering, search ranking) | SQLAlchemy ORM internals |
| **Integration** | End-to-end: feed fetch -> parse -> dedup -> store -> API read | Frontend rendering, deployment scripts |

**Target: ~80% coverage on business logic, ~60% coverage on routes, 0% coverage on framework code.**

#### 4.2 Core Test Fixtures (`conftest.py`)

```python
# tests/conftest.py

import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

TEST_DATABASE_URL = "postgresql+asyncpg://sentinel:sentinel_dev@localhost:5432/sentinel_test"

@pytest_asyncio.fixture(scope="session")
async def test_engine():
    engine = create_async_engine(TEST_DATABASE_URL)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest_asyncio.fixture
async def db_session(test_engine):
    """Per-test session with rollback for isolation."""
    async_session = async_sessionmaker(test_engine, expire_on_commit=False)
    async with async_session() as session:
        async with session.begin():
            yield session
            await session.rollback()

@pytest_asyncio.fixture
async def client(db_session):
    """Async test client with DB dependency override."""
    app.dependency_overrides[get_db] = lambda: db_session
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()

@pytest.fixture
def sample_rss_content():
    """Load sample RSS XML from fixtures directory."""
    with open("tests/fixtures/sample_rss.xml", "rb") as f:
        return f.read()

@pytest.fixture
def malformed_rss_content():
    """Load malformed RSS XML for error handling tests."""
    with open("tests/fixtures/malformed_rss.xml", "rb") as f:
        return f.read()
```

#### 4.3 Mocking RSS Feeds

```python
# tests/test_ingestion.py

from unittest.mock import patch, MagicMock
from app.ingestion.feed_parser import parse_feed_content

def test_parse_well_formed_feed(sample_rss_content):
    """Test parsing a well-formed RSS feed returns normalized entries."""
    entries = parse_feed_content(sample_rss_content, "https://example.com/feed.xml")
    assert len(entries) > 0
    for entry in entries:
        assert "title" in entry
        assert "link" in entry
        assert "fingerprint" in entry

def test_parse_malformed_feed_does_not_crash(malformed_rss_content):
    """Malformed feeds should return empty list or partial results, not raise."""
    entries = parse_feed_content(malformed_rss_content, "https://example.com/bad.xml")
    assert isinstance(entries, list)  # No exception raised

@patch("app.ingestion.feed_poller.httpx.Client")
def test_poll_feed_handles_304(mock_client):
    """304 Not Modified should skip parsing and update metadata."""
    mock_response = MagicMock(status_code=304)
    mock_client.return_value.__enter__.return_value.get.return_value = mock_response
    result = poll_feed("source-uuid-here")
    assert result["status"] == "not_modified"

def test_dedup_rejects_exact_duplicate():
    """Same fingerprint should be detected as duplicate."""
    dedup = DeduplicationService(redis_client=mock_redis)
    entry = {"fingerprint": "abc123", "link": "https://example.com/article", "title": "Test"}
    assert not dedup.is_duplicate(entry)  # First time: not a duplicate
    dedup.register_entry(entry)
    assert dedup.is_duplicate(entry)      # Second time: duplicate
```

---

### 5. Sprint Plan

#### Sprint 1: Foundation (Week 1-2) — "Data Flows In"

**Goal:** RSS feeds are being fetched, parsed, deduplicated, and stored in PostgreSQL. A health endpoint confirms the system is running.

**Files to create:**

| File | Key Functions/Classes | Definition of Done |
|------|----------------------|-------------------|
| `app/main.py` | `create_app()`, lifespan handler | App starts, health endpoint returns 200 |
| `app/core/config.py` | `Settings(BaseSettings)` | All env vars loaded and validated |
| `app/core/database.py` | `get_db()`, engine setup | Can connect to PostgreSQL async |
| `app/core/redis.py` | `get_redis()`, connection pool | Can connect to Redis |
| `app/models/base.py` | `Base`, `TimestampMixin`, `UUIDMixin` | Base classes for all models |
| `app/models/source.py` | `Source` ORM model | Source table created via Alembic |
| `app/models/event.py` | `Event` ORM model | Event table created with PostGIS geometry |
| `app/schemas/source.py` | `SourceCreate`, `SourceRead` | Pydantic validation works |
| `app/schemas/event.py` | `EventCreate`, `EventRead` | Pydantic validation works |
| `app/schemas/common.py` | `PaginatedResponse` | Generic paginated wrapper |
| `app/routers/health.py` | `GET /api/v1/health` | Returns DB and Redis status |
| `app/routers/sources.py` | `GET/POST /api/v1/sources` | CRUD for feed sources |
| `app/ingestion/celery_app.py` | Celery app config | Celery connects to Redis broker |
| `app/ingestion/feed_registry.py` | `create_source()`, `get_sources_due_for_polling()` | Sources queryable by poll time |
| `app/ingestion/feed_poller.py` | `poll_feed()`, `dispatch_polling_round()` | Feeds fetched with ETags |
| `app/ingestion/feed_parser.py` | `parse_feed_content()`, `normalize_entry()` | RSS parsed into normalized dicts |
| `app/ingestion/dedup.py` | `DeduplicationService` | SHA-256 fingerprint dedup works |
| `docker-compose.yml` | All services defined | `docker compose up` starts everything |
| `Dockerfile` | Backend image | Image builds and runs |
| `migrations/` | Initial migration | Tables created in PostgreSQL |
| `tests/conftest.py` | Test fixtures | Test DB, async client working |
| `tests/test_ingestion.py` | Feed parser + dedup tests | 10+ tests passing |

**Sprint 1 Definition of Done:**
- `docker compose up` starts PostgreSQL, Redis, FastAPI, Celery worker, Celery Beat
- `POST /api/v1/sources` creates a feed source
- `GET /api/v1/health` returns healthy status
- Celery Beat dispatches polling tasks every 2 minutes
- At least one real RSS feed (e.g., ReliefWeb) is fetched, parsed, deduplicated, and stored as events in PostgreSQL
- `GET /api/v1/events` returns stored events with pagination
- 10+ tests pass in pytest

---

#### Sprint 2: API & Real-Time (Week 3-4) — "Data Flows Out"

**Goal:** Full REST API for events with filtering, full-text search, and SSE streaming for real-time updates.

**Files to create/extend:**

| File | Key Functions/Classes | Definition of Done |
|------|----------------------|-------------------|
| `app/routers/events.py` | `GET /events`, `GET /events/{id}` | Paginated list, detail view |
| `app/routers/search.py` | `POST /api/v1/search` | Full-text search with filters |
| `app/routers/stream.py` | `GET /api/v1/stream/events` | SSE endpoint streams new events |
| `app/services/event_service.py` | `EventService` class | Business logic separated from routes |
| `app/services/source_service.py` | `SourceService` class | Source health monitoring |
| `app/services/search_service.py` | `SearchService` class | tsvector-based search |
| `app/services/stream_service.py` | `StreamService` class | Redis pub/sub -> SSE |
| `app/ingestion/scheduler.py` | `calculate_adaptive_interval()` | Adaptive polling intervals |
| `app/ingestion/normalizer.py` | `to_event_create()` | Structured event creation |
| `app/models/actor.py` | `Actor` ORM model | Actor table in DB |
| `app/models/location.py` | `Location` ORM model | Location with PostGIS geometry |
| `app/schemas/search.py` | `SearchQuery`, `SearchResult` | Search schema validation |
| `tests/test_events.py` | Event API tests | CRUD + filters tested |
| `tests/test_search.py` | Search endpoint tests | Full-text search tested |

**Sprint 2 Extension of existing files:**

| File | Changes |
|------|---------|
| `app/models/event.py` | Add tsvector column, GIN index |
| `app/ingestion/feed_poller.py` | Publish to Redis pub/sub after storage |
| `app/ingestion/dedup.py` | Add URL normalization (Tier 2) and MinHash LSH (Tier 3) |
| `app/main.py` | Add events, search, stream routers |

**Sprint 2 Definition of Done:**
- `GET /api/v1/events?severity_min=5&page=2` returns filtered, paginated events
- `POST /api/v1/search` returns ranked full-text search results
- `GET /api/v1/stream/events` delivers SSE updates when new events are stored
- Adaptive polling intervals work (high-frequency feeds polled more often)
- 3-tier deduplication fully operational
- 20+ tests pass

---

#### Sprint 3: Hardening & Multiple Sources (Week 5-6) — "Production-Ready Backend"

**Goal:** Multiple real data sources ingesting, error handling hardened, API key auth, monitoring, source health dashboard endpoint.

**Files to create/extend:**

| File | Key Functions/Classes | Definition of Done |
|------|----------------------|-------------------|
| `app/core/security.py` | `verify_api_key()` | API key middleware works |
| `app/routers/actors.py` | `GET /api/v1/actors` | Actor CRUD endpoints |
| `app/models/conflict.py` | `Conflict` ORM model | Conflict table in DB |
| `app/services/actor_service.py` | `ActorService` class | Actor CRUD logic |
| Seed data script | `scripts/seed_sources.py` | Load 20+ RSS feed sources |
| `app/ingestion/feed_poller.py` | Error tracking + auto-disable | Feeds disabled after 5 errors |
| `app/routers/sources.py` | `GET /api/v1/sources/{id}/health` | Source health + error history |

**Sprint 3 Extension of existing files:**

| File | Changes |
|------|---------|
| `app/main.py` | Add auth middleware, actor router |
| `app/routers/events.py` | Add API key requirement |
| `app/routers/sources.py` | Source health endpoint |
| `app/ingestion/scheduler.py` | Production-tuned intervals |
| `docker-compose.yml` | Add resource limits, restart policies |

**Sprint 3 Data Sources to Integrate:**

| Source | Type | Feed URL Pattern |
|--------|------|-----------------|
| ReliefWeb | RSS | `https://reliefweb.int/updates/rss.xml` |
| Crisis Group | RSS | `https://www.crisisgroup.org/latest-updates/feed` |
| Reuters World | RSS | `https://www.reuters.com/arc/outboundfeeds/v3/all/rss.xml` |
| BBC World | RSS | `https://feeds.bbci.co.uk/news/world/rss.xml` |
| AP News | RSS | `https://rsshub.app/apnews/topics/world-news` |
| Al Jazeera | RSS | `https://www.aljazeera.com/xml/rss/all.xml` |
| ACLED | API | `https://api.acleddata.com/acled/read` (requires key) |

**Sprint 3 Definition of Done:**
- 20+ RSS sources actively polling and ingesting
- Auto-disable works (source disabled after 5 consecutive errors)
- API key authentication on all endpoints
- Source health endpoint shows per-source status, last poll time, error count
- All critical error paths have retry logic
- `docker compose up` runs stable for 24+ hours without intervention
- 30+ tests pass
- System handles 1000+ events in database without performance issues

---

## Comparison Tables

### Database Access Pattern Comparison

| Criteria | SQLAlchemy 2.0 Async | Raw asyncpg | SQLModel | Recommendation |
|----------|---------------------|-------------|----------|----------------|
| Query performance | ~0.54ms/query | ~0.19ms/query (2.7x faster) | Similar to SQLAlchemy | SQLAlchemy 2.0 |
| ORM features | Full ORM, relationships, lazy loading | None (raw SQL only) | Simplified ORM | SQLAlchemy 2.0 |
| GeoAlchemy2 (PostGIS) | Full support | Manual ST_* functions | No support | SQLAlchemy 2.0 |
| Alembic migrations | Native integration | Manual schema management | Partial support | SQLAlchemy 2.0 |
| Developer productivity | High | Low (write all SQL manually) | High | SQLAlchemy 2.0 |
| Escape hatch to raw SQL | `session.execute(text(...))` | Already raw | `session.execute(text(...))` | SQLAlchemy 2.0 |

**Decision:** SQLAlchemy 2.0 async with asyncpg driver. The 2-3x raw performance gap is irrelevant at MVP scale (<100K events). Use raw asyncpg via `text()` for any hot-path queries later.

### Celery Task Configuration Comparison

| Setting | Development | Production | Why |
|---------|------------|------------|-----|
| `concurrency` | 2 | 4-8 | More workers for more feeds |
| `acks_late` | True | True | Re-queue tasks if worker crashes |
| `retry_backoff` | True | True | Exponential backoff on errors |
| `max_retries` | 3 | 5 | More retries in prod (transient errors) |
| `task_soft_time_limit` | 60s | 120s | Prevent hung tasks |
| `task_time_limit` | 120s | 300s | Hard kill for truly stuck tasks |
| `worker_prefetch_multiplier` | 1 | 1 | One task at a time per worker (fair scheduling) |

---

## Priority Implementation Order

1. **Docker Compose + PostgreSQL + Redis** — Get the infrastructure running first. Everything depends on this. Should take 1-2 hours.

2. **FastAPI app skeleton with health endpoint** — Validate that the app starts, connects to DB and Redis, and responds to requests. This is the smoke test for everything.

3. **SQLAlchemy models + Alembic migration** — Define Source and Event tables. Run migration. Confirm tables exist in PostgreSQL with PostGIS geometry columns.

4. **Feed parser (feedparser + fastfeedparser)** — Build and test the parsing layer independently. Use sample RSS files as fixtures. This has zero external dependencies.

5. **Feed poller (Celery task + httpx)** — Build the HTTP fetching layer with conditional requests. Test against a real RSS feed URL.

6. **Deduplication service** — Start with SHA-256 fingerprint tier only. Add URL normalization and MinHash in Sprint 2.

7. **Source CRUD API** — Endpoints to register and manage RSS feeds. This enables adding new sources without code changes.

8. **Event CRUD API** — Endpoints to read stored events. This is the first visible output of the whole pipeline.

9. **SSE streaming** — Real-time event delivery. This makes the dashboard feel alive.

10. **Full-text search** — PostgreSQL tsvector-based search. Critical for analyst workflows.

---

## Cost Analysis

### Development Infrastructure (Docker Compose, local)

| Component | Cost |
|-----------|------|
| PostgreSQL 16 + PostGIS (Docker) | Free |
| Redis 7 (Docker) | Free |
| Celery workers (Docker) | Free |
| All Python libraries | Free (open source) |
| Developer machine (existing) | $0 incremental |
| **Total** | **$0** |

### Staging/Production Infrastructure

| Deployment Option | Monthly Cost | Best For |
|-------------------|-------------|----------|
| Single VPS (4 CPU, 8GB RAM) + Docker Compose | $24-48/month (Hetzner/DigitalOcean) | MVP, demo, small team |
| AWS: RDS t3.small + ElastiCache t3.micro + ECS Fargate | $120-200/month | Production, managed |
| Self-hosted K8s on 3 VPS nodes | $72-150/month + ops time | Scale (premature for MVP) |

### Library License Summary

| Library | License | Commercial Use |
|---------|---------|---------------|
| FastAPI | MIT | Yes |
| SQLAlchemy | MIT | Yes |
| asyncpg | Apache-2.0 | Yes |
| Celery | BSD-3-Clause | Yes |
| Redis (server) | BSD-3-Clause (through v7.2) | Yes |
| feedparser | BSD-2-Clause | Yes |
| fastfeedparser | MIT | Yes |
| datasketch | MIT | Yes |
| spaCy | MIT | Yes |
| Next.js | MIT | Yes |
| Mapbox GL JS | BSD-3-Clause (requires Mapbox access token) | Yes (token needed) |

All dependencies are permissively licensed. The only cost-bearing component is Mapbox GL JS, which requires a free access token (50,000 map loads/month free).

---

## Open Questions

1. **Celery vs native asyncio for feed polling** — Celery adds infrastructure (separate worker processes) but is battle-tested at scale (NewsBlur uses it for millions of feeds). An alternative is `asyncio` + `httpx` + `APScheduler` inside the FastAPI process. The Celery approach is chosen here for reliability, but for fewer than 50 feeds, a simpler async loop may suffice. **Next step:** If feed count stays under 50 for the first 3 months, consider replacing Celery with `APScheduler` to reduce complexity.

2. **redis-py version compatibility with Celery** — Celery 5.6.x (via Kombu) constrains redis to `<6.2`. This means redis-py 7.x cannot be used. This is a known issue in the Celery ecosystem. **Next step:** Monitor Celery 5.7 release for updated Kombu dependency. Pin redis to `>=5.2.0,<6.2` until resolved.

3. **PostGIS vs plain latitude/longitude columns for MVP** — PostGIS enables powerful spatial queries (radius search, bounding box, intersections) but adds Docker image size and complexity. For MVP, simple `lat`/`lon` float columns may suffice. **Next step:** Use PostGIS from the start since the Docker image (`postgis/postgis:16-3.5`) includes it, and retrofitting spatial indexes later is harder than starting with them.

4. **TimescaleDB for MVP** — findings-02 recommends deferring TimescaleDB to Phase 2 (when events table exceeds 500K rows). Standard PostgreSQL tables with timestamp indexes should handle MVP scale. **Next step:** Monitor events table size. When approaching 500K rows, add TimescaleDB extension and convert events table to a hypertable.

5. **ACLED API key and rate limits** — ACLED moved to a freemium model. The free tier provides access to historical data but may have rate limits. **Next step:** Register for an ACLED API key and test rate limits before Sprint 3 integration.

6. **Frontend framework: Next.js vs React SPA** — findings-02 recommends React SPA (Vite), but this roadmap includes Next.js in the frontend scaffold. The CLAUDE.md spec says Next.js. **Decision for now:** Use Next.js as specified in CLAUDE.md. The SSR overhead is minimal with App Router, and the developer experience (file-based routing, API routes as fallback) is excellent for dashboards. This overrides the findings-02 recommendation based on project specification.

---

## Sources & References

### FastAPI Architecture
- FastAPI Official Docs — Bigger Applications: https://fastapi.tiangolo.com/tutorial/bigger-applications/
- zhanymkanov/fastapi-best-practices (GitHub): https://github.com/zhanymkanov/fastapi-best-practices
- Building Production-Ready FastAPI with Service Layer Architecture (2025): https://medium.com/@abhinav.dobhal/building-production-ready-fastapi-applications-with-service-layer-architecture-in-2025-f3af8a6ac563
- FastAPI Best Practices for Production 2026: https://fastlaunchapi.dev/blog/fastapi-best-practices-production-2026

### Docker & Deployment
- TestDriven.io — Definitive Guide to Celery and FastAPI (Docker): https://testdriven.io/courses/fastapi-celery/docker/
- Celery + Redis + FastAPI Production Guide 2025: https://medium.com/@dewasheesh.rana/celery-redis-fastapi-the-ultimate-2025-production-guide-broker-vs-backend-explained-5b84ef508fa7
- FastAPI Boilerplate Docker Setup (benavlabs): https://benavlabs.github.io/FastAPI-boilerplate/user-guide/configuration/docker-setup/
- FastAPI-Celery-Redis-Postgres-Docker example: https://github.com/alperencubuk/FastAPI-Celery-Redis-Postgres-Docker-REST-API

### Database & Async
- asyncpg vs SQLAlchemy performance (GitHub discussion): https://github.com/sqlalchemy/sqlalchemy/discussions/7294
- Building High-Performance Async APIs with FastAPI + SQLAlchemy 2.0 + asyncpg: https://leapcell.io/blog/building-high-performance-async-apis-with-fastapi-sqlalchemy-2-0-and-asyncpg
- SQLAlchemy 2.0 Async Documentation: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
- asyncpg Documentation: https://magicstack.github.io/asyncpg/current/faq.html
- Top PostgreSQL Drivers for Python: https://www.tigerdata.com/learn/top-postgresql-drivers-for-python

### Testing
- FastAPI Official Docs — Testing: https://fastapi.tiangolo.com/tutorial/testing/
- Testing FastAPI with async database session: https://dev.to/whchi/testing-fastapi-with-async-database-session-1b5d
- FastAPI Async Testing (CompileNRun): https://www.compilenrun.com/docs/framework/fastapi/fastapi-testing/fastapi-async-testing/
- Complete Guide to FastAPI x pytest (2025): https://blog.greeden.me/en/2025/08/19/how-tests-grow-robust-apis-a-complete-guide-to-automated-testing-with-fastapi-x-pytest-x-testclient/
- pytest-with-eric — FastAPI CRUD Testing: https://pytest-with-eric.com/pytest-advanced/pytest-fastapi-testing/

### Prior Research (Internal)
- findings-02-architecture.md — System architecture and tech stack decisions
- findings-06-rss-ingestion-methods.md — RSS parsing, polling, deduplication
- findings-08-anti-patterns.md — Anti-patterns and pitfalls to avoid

### Package Versions (PyPI, as of February 2026)
- FastAPI 0.128.5: https://pypi.org/project/fastapi/
- SQLAlchemy 2.0.46: https://github.com/sqlalchemy/sqlalchemy/releases
- asyncpg 0.31.0: https://pypi.org/project/asyncpg/
- Celery 5.6.2: https://pypi.org/project/celery/
- redis-py 5.2.x (constrained by Celery/Kombu <6.2): https://pypi.org/project/redis/
- Pydantic 2.10.x: https://pypi.org/project/pydantic/
