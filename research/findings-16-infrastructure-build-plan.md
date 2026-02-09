# Build Plan 16: Infrastructure, Testing & Integration — Coding Agent Instructions

## Executive Summary

Step-by-step build plan for Sentinel's Docker setup, database initialization, testing strategy, CI/CD, and cross-component integration. This is the glue that makes backend + ingestion + frontend work together. Based on findings-08 (anti-patterns), findings-09 (database decision), findings-12 (implementation roadmap).

## Research Scope

Based on: findings-08-anti-patterns.md, findings-09-database-decision.md, findings-12-implementation-roadmap.md

---

## 1. Docker Compose — Complete Local Development Stack

```yaml
# docker-compose.yml (project root)
version: "3.9"

services:
  # ============================================================
  # PostgreSQL + TimescaleDB + PostGIS + pgvector
  # ============================================================
  db:
    image: timescale/timescaledb-ha:pg17-ts2.18.0
    container_name: sentinel-db
    environment:
      POSTGRES_DB: sentinel
      POSTGRES_USER: sentinel
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-sentinel_dev}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - pgdata:/home/postgres/pgdata/data
      - ./docker/init-db:/docker-entrypoint-initdb.d
    shm_size: "256mb"
    command:
      - "postgres"
      - "-c"
      - "shared_preload_libraries=timescaledb"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=512MB"
      - "-c"
      - "work_mem=32MB"
      - "-c"
      - "wal_level=logical"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sentinel -d sentinel"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================================
  # Redis: Cache, Celery broker, pub/sub for real-time
  # ============================================================
  redis:
    image: redis:7.4-alpine
    container_name: sentinel-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redisdata:/data
    command: >
      redis-server
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================================
  # FastAPI Backend
  # ============================================================
  backend:
    build:
      context: ./src/backend
      dockerfile: Dockerfile
    container_name: sentinel-backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - "8000:8000"
    volumes:
      - ./src/backend:/app
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # ============================================================
  # Celery Worker (feed polling + data ingestion)
  # ============================================================
  celery-worker:
    build:
      context: ./src/backend
      dockerfile: Dockerfile
    container_name: sentinel-celery-worker
    command: celery -A app.ingestion.celery_app worker --loglevel=info --concurrency=4
    volumes:
      - ./src/backend:/app
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # ============================================================
  # Celery Beat Scheduler (periodic tasks)
  # ============================================================
  celery-beat:
    build:
      context: ./src/backend
      dockerfile: Dockerfile
    container_name: sentinel-celery-beat
    command: celery -A app.ingestion.celery_app beat --loglevel=info
    volumes:
      - ./src/backend:/app
    env_file:
      - .env
    depends_on:
      redis:
        condition: service_healthy
    restart: unless-stopped

  # ============================================================
  # Next.js Frontend
  # ============================================================
  frontend:
    build:
      context: ./src/frontend
      dockerfile: Dockerfile
    container_name: sentinel-frontend
    command: npm run dev
    ports:
      - "3000:3000"
    volumes:
      - ./src/frontend:/app
      - /app/node_modules
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
      - NEXT_PUBLIC_WS_URL=ws://localhost:8000/api/v1/ws
      - NEXT_PUBLIC_MAPBOX_TOKEN=${MAPBOX_TOKEN:-}
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

---

## 2. Dockerfiles

### Backend Dockerfile

```dockerfile
# src/backend/Dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libgeos-dev \
    libproj-dev \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir -e ".[dev]"

COPY . .

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend Dockerfile

```dockerfile
# src/frontend/Dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm install

COPY . .

EXPOSE 3000
CMD ["npm", "run", "dev"]
```

---

## 3. Database Initialization Scripts

Place in `docker/init-db/` — executed alphabetically on first `docker compose up`.

### 001-extensions.sql

```sql
-- Enable required PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
CREATE EXTENSION IF NOT EXISTS postgis CASCADE;
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Verify
SELECT extname, extversion FROM pg_extension ORDER BY extname;
```

### 002-schema.sql

```sql
-- Sources table
CREATE TABLE IF NOT EXISTS sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL UNIQUE,
    source_type VARCHAR(20) NOT NULL DEFAULT 'rss',
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    poll_interval_minutes INTEGER NOT NULL DEFAULT 60,
    last_poll_at TIMESTAMPTZ,
    last_poll_status VARCHAR(20),
    last_poll_error TEXT,
    etag VARCHAR(255),
    last_modified VARCHAR(255),
    total_items_ingested INTEGER NOT NULL DEFAULT 0,
    error_count INTEGER NOT NULL DEFAULT 0,
    consecutive_errors INTEGER NOT NULL DEFAULT 0,
    next_poll_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reliability_score FLOAT NOT NULL DEFAULT 0.5,
    config JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_sources_next_poll ON sources (next_poll_at) WHERE enabled = TRUE;

-- Events table
CREATE TABLE IF NOT EXISTS events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    description TEXT,
    event_type VARCHAR(100),
    event_date DATE NOT NULL,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    geo GEOGRAPHY(POINT, 4326),
    country VARCHAR(100),
    region VARCHAR(255),
    actors TEXT[],
    fatalities INTEGER DEFAULT 0,
    source_name VARCHAR(255),
    source_url TEXT,
    source_id UUID REFERENCES sources(id) ON DELETE SET NULL,
    source_type VARCHAR(20),
    confidence_score FLOAT DEFAULT 0.5,
    content_hash VARCHAR(64),
    raw_data JSONB,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Convert to TimescaleDB hypertable
SELECT create_hypertable('events', 'ingested_at',
    migrate_data => true,
    if_not_exists => true
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_events_date ON events (event_date DESC);
CREATE INDEX IF NOT EXISTS idx_events_type ON events (event_type);
CREATE INDEX IF NOT EXISTS idx_events_country ON events (country);
CREATE INDEX IF NOT EXISTS idx_events_geo ON events USING GIST (geo);
CREATE INDEX IF NOT EXISTS idx_events_content_hash ON events (content_hash);
CREATE INDEX IF NOT EXISTS idx_events_search ON events USING GIN (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, ''))
);

-- Source rankings (time-series)
CREATE TABLE IF NOT EXISTS source_rankings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    uptime_score FLOAT NOT NULL DEFAULT 0.0,
    freshness_score FLOAT NOT NULL DEFAULT 0.0,
    relevance_score FLOAT NOT NULL DEFAULT 0.0,
    quality_score FLOAT NOT NULL DEFAULT 0.0,
    composite_score FLOAT NOT NULL DEFAULT 0.0,
    items_last_24h INTEGER NOT NULL DEFAULT 0,
    errors_last_24h INTEGER NOT NULL DEFAULT 0,
    UNIQUE(source_id, calculated_at)
);

SELECT create_hypertable('source_rankings', 'calculated_at',
    migrate_data => true,
    if_not_exists => true
);
```

### 003-seed-sources.sql

```sql
-- Tier 1 RSS sources (Day 1)
INSERT INTO sources (name, url, source_type, poll_interval_minutes) VALUES
    ('BBC World News', 'https://feeds.bbci.co.uk/news/world/rss.xml', 'rss', 30),
    ('BBC Middle East', 'https://feeds.bbci.co.uk/news/world/middle_east/rss.xml', 'rss', 30),
    ('BBC Africa', 'https://feeds.bbci.co.uk/news/world/africa/rss.xml', 'rss', 30),
    ('BBC Asia', 'https://feeds.bbci.co.uk/news/world/asia/rss.xml', 'rss', 30),
    ('Al Jazeera', 'https://www.aljazeera.com/xml/rss/all.xml', 'rss', 30),
    ('France 24', 'https://www.france24.com/en/rss', 'rss', 60),
    ('AP World News', 'https://apnews.com/world-news.rss', 'rss', 30),
    ('Crisis Group', 'https://www.crisisgroup.org/rss-0', 'rss', 120)
ON CONFLICT (url) DO NOTHING;
```

---

## 4. Environment Variables (.env.example)

```bash
# Database
POSTGRES_PASSWORD=sentinel_dev
DATABASE_URL=postgresql+asyncpg://sentinel:sentinel_dev@db:5432/sentinel
DATABASE_URL_SYNC=postgresql://sentinel:sentinel_dev@db:5432/sentinel

# Redis
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

# Security
SECRET_KEY=dev-secret-change-in-production
ALGORITHM=HS256

# App
APP_NAME=Sentinel
APP_ENV=development
LOG_LEVEL=INFO
CORS_ORIGINS=http://localhost:3000

# Frontend
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:8000/api/v1/ws
MAPBOX_TOKEN=your-mapbox-token-here

# Polling config
DEFAULT_POLL_INTERVAL_MINUTES=60
MAX_CONSECUTIVE_ERRORS=5

# External APIs (register for free)
ACLED_API_KEY=
ACLED_EMAIL=
```

---

## 5. Testing Strategy

### What to Test (MVP)

| Layer | What to Test | Tool |
|-------|-------------|------|
| API endpoints | All REST routes return correct status + shape | pytest + httpx |
| Event filtering | Date, country, type filters produce correct results | pytest + test DB |
| Feed parsing | feedparser output normalization | pytest + fixture XMLs |
| Deduplication | Same content → same hash, different content → different hash | pytest unit tests |
| Source CRUD | Create, update, disable sources | pytest + test DB |

### What NOT to Test (MVP)

- Frontend components (no Jest/RTL setup needed yet)
- Celery task execution (test the logic, mock the Celery parts)
- WebSocket connections (test manually)
- Full E2E flows (Playwright can wait for Tier 2)

### pytest Configuration

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.core.database import get_db

TEST_DB_URL = "postgresql+asyncpg://sentinel:sentinel_dev@localhost:5432/sentinel_test"

@pytest_asyncio.fixture
async def db_session():
    engine = create_async_engine(TEST_DB_URL)
    session_factory = async_sessionmaker(engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
    await engine.dispose()

@pytest_asyncio.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

### Test Fixtures

```xml
<!-- tests/fixtures/sample_rss.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>Test Conflict News</title>
    <item>
      <title>Clashes erupt in eastern region</title>
      <description>Armed groups clashed near the border...</description>
      <link>https://example.com/article/123</link>
      <pubDate>Mon, 03 Feb 2025 14:30:00 GMT</pubDate>
      <category>Conflict</category>
    </item>
    <item>
      <title>Protests in capital city</title>
      <description>Thousands gathered in the main square...</description>
      <link>https://example.com/article/124</link>
      <pubDate>Mon, 03 Feb 2025 10:15:00 GMT</pubDate>
      <category>Protests</category>
    </item>
  </channel>
</rss>
```

### Sample Test

```python
# tests/test_events.py
import pytest

@pytest.mark.asyncio
async def test_list_events_empty(client):
    response = await client.get("/api/v1/events")
    assert response.status_code == 200
    data = response.json()
    assert data["data"] == []
    assert data["meta"]["total"] == 0

@pytest.mark.asyncio
async def test_health_check(client):
    response = await client.get("/api/v1/health")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"
```

---

## 6. GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, claude/*]
  pull_request:
    branches: [main]

jobs:
  backend-lint-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb-ha:pg17-ts2.18.0
        env:
          POSTGRES_DB: sentinel_test
          POSTGRES_USER: sentinel
          POSTGRES_PASSWORD: sentinel_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U sentinel"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7.4-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        working-directory: src/backend
        run: pip install -e ".[dev]"
      - name: Lint
        working-directory: src/backend
        run: ruff check .
      - name: Type check
        working-directory: src/backend
        run: mypy app/
      - name: Test
        working-directory: src/backend
        env:
          DATABASE_URL: postgresql+asyncpg://sentinel:sentinel_test@localhost:5432/sentinel_test
          REDIS_URL: redis://localhost:6379/0
        run: pytest --cov=app --cov-report=term-missing

  frontend-lint-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
      - name: Install dependencies
        working-directory: src/frontend
        run: npm ci
      - name: Lint
        working-directory: src/frontend
        run: npm run lint
      - name: Type check
        working-directory: src/frontend
        run: npm run type-check
      - name: Build
        working-directory: src/frontend
        run: npm run build
```

---

## 7. .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
.eggs/
*.egg
.mypy_cache/
.ruff_cache/
.pytest_cache/
htmlcov/
.coverage

# Node
node_modules/
.next/
out/

# Environment
.env
.env.local
.env.production

# Docker volumes
pgdata/
redisdata/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Misc
*.log
```

---

## 8. Integration Architecture — End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER'S BROWSER                             │
│                                                                     │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────┐             │
│   │  Mapbox Map  │  │  Event Feed  │  │  Timeline    │             │
│   └──────┬──────┘  └──────┬───────┘  └──────┬───────┘             │
│          │                │                  │                      │
│          └────────────────┼──────────────────┘                      │
│                           │ HTTP REST + WebSocket                    │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                     FASTAPI BACKEND (:8000)                       │
│                                                                   │
│   GET /events  ──▶ event_service ──▶ PostgreSQL (query)          │
│   GET /clusters ──▶ PostGIS ST_ClusterDBSCAN ──▶ GeoJSON        │
│   GET /timeline ──▶ TimescaleDB time_bucket() ──▶ JSON           │
│   WS /ws/events ◀── Redis pub/sub ◀── new event notification     │
│                                                                   │
└────────────────────────────┬──────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
┌──────────────────┐ ┌──────────┐ ┌─────────────────┐
│   PostgreSQL     │ │  Redis   │ │  Celery Worker   │
│   + TimescaleDB  │ │          │ │                   │
│   + PostGIS      │ │  Cache   │ │  poll_feed()      │
│                  │ │  Broker  │ │  fetch_acled()     │
│  events table    │ │  Pub/Sub │ │  fetch_gdelt()     │
│  sources table   │ │          │ │                   │
└──────────────────┘ └──────────┘ └───────┬───────────┘
                                          │
                                          ▼
                               ┌─────────────────────┐
                               │  External Sources    │
                               │  - BBC RSS           │
                               │  - Al Jazeera RSS    │
                               │  - ACLED API         │
                               │  - GDELT API         │
                               │  - Crisis Group RSS  │
                               └─────────────────────┘
```

### Sequence: New Event Arrives → User Sees It

```
1. Celery Beat triggers poll_all_due_feeds() every 60 seconds
2. poll_all_due_feeds() queries sources WHERE next_poll_at <= NOW()
3. For each due source, dispatches poll_feed(source_id) task
4. poll_feed() makes HTTP GET to RSS feed URL with ETag header
5. If 200: pass response body to feed_parser.parse_feed()
6. feed_parser returns list of normalized entry dicts
7. normalizer maps entries to canonical Event schema
8. dedup checks content_hash against events table
9. event_storage inserts new events via bulk INSERT
10. event_storage publishes new event IDs to Redis channel "sentinel:new_events"
11. FastAPI WebSocket handler (stream.py) is subscribed to Redis channel
12. WebSocket pushes {"type": "new_event", "data": {...}} to connected browsers
13. Frontend event-store receives WebSocket message
14. Event feed prepends new event with highlight animation
15. Map adds new marker with pop-in animation
16. Timeline chart updates bar for current day
```

---

## 9. Master Build Order — Full Project

This is the order a coding agent should follow across ALL components:

```
Phase 1: Infrastructure (this document)
  1. Create project root files (.gitignore, .env.example, docker-compose.yml)
  2. Create docker/init-db/ SQL scripts (extensions, schema, seed)
  3. Run docker compose up db redis — verify DB is healthy
  4. Connect to DB and verify: extensions loaded, tables created, seed data present

Phase 2: Backend (findings-13)
  5. Create src/backend/ scaffold (pyproject.toml, Dockerfile, directory tree)
  6. Build core/ (config, database, redis, dependencies)
  7. Build models/ (base, source, event)
  8. Set up Alembic (or use init SQL — skip Alembic for MVP)
  9. Build schemas/ (all Pydantic models)
  10. Build services/ (event_service, source_service)
  11. Build routers/ (health first, then sources, then events)
  12. Build main.py — wire everything
  13. Verify: docker compose up backend, curl /api/v1/health → "ok"

Phase 3: Ingestion (findings-14)
  14. Build celery_app.py
  15. Build feed_poller.py + feed_parser.py
  16. Build normalizer.py + dedup.py + event_storage.py
  17. Build acled_client.py + gdelt_client.py
  18. Create seed_sources.json
  19. Verify: docker compose up celery-worker celery-beat
  20. Check: events appearing in DB after first poll cycle

Phase 4: Frontend (findings-15)
  21. Scaffold Next.js project
  22. Install dependencies, set up Tailwind dark theme
  23. Build lib/ (types, api, constants, stores)
  24. Build map component (Mapbox GL JS)
  25. Build event feed + filter bar
  26. Build timeline chart (Recharts)
  27. Wire up main dashboard page
  28. Build source management page
  29. Verify: docker compose up frontend, visit localhost:3000

Phase 5: Integration
  30. Connect frontend to live backend API
  31. Verify end-to-end: feed polled → event in DB → visible on dashboard
  32. Add WebSocket real-time push
  33. Write tests
  34. Set up CI (GitHub Actions)
```

---

## Comparison Tables

### Docker Image Choices

| Component | Image | Why This One |
|-----------|-------|-------------|
| Database | timescale/timescaledb-ha:pg17-ts2.18.0 | Bundles PostgreSQL 17 + TimescaleDB + PostGIS + pgvector in one image |
| Redis | redis:7.4-alpine | Alpine for small size. v7.4 is latest stable |
| Python | python:3.12-slim | Slim for smaller image. 3.12 for latest features |
| Node | node:22-alpine | LTS version, Alpine for size |

### Testing Framework Comparison

| Criteria | pytest | unittest | nose2 |
|----------|--------|----------|-------|
| Async support | pytest-asyncio | Manual | No |
| Fixtures | Excellent | setUp/tearDown | Basic |
| Plugins | Huge ecosystem | Limited | Limited |
| **Recommendation** | **YES** | No | No |

---

## Priority Implementation Order

1. **Docker Compose + init scripts** — Get database and Redis running
2. **.env.example + .gitignore** — Project hygiene
3. **Backend Dockerfile** — Containerize the app
4. **Frontend Dockerfile** — Containerize the frontend
5. **pytest conftest.py** — Test infrastructure
6. **GitHub Actions CI** — Automated quality gates
7. **Integration verification** — End-to-end smoke test

---

## Open Questions

1. **TimescaleDB image version**: Pin to exact version or allow minor updates?
2. **Elasticsearch**: Deferred per findings-08 — PostgreSQL tsvector for MVP search. Add ES later?
3. **PgBouncer**: Needed for MVP or only at scale? Findings-09 includes it but MVP may not need it.
4. **Production Docker Compose**: Separate docker-compose.prod.yml with production settings?
5. **SSL/TLS**: Self-signed certs for local dev or skip until deployment?
