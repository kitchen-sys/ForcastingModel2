# Research Brief 12: Implementation Roadmap — Phase 2 Build Plan

## Objective
Create a concrete implementation roadmap for Phase 2 (Data Ingestion Pipeline) and Phase 3 (Backend API). This is the bridge from research to code. A developer reads this and starts coding.

## Context
Read ALL prior findings:
- findings-02-architecture.md (system architecture)
- findings-05-database-ranking.md (database approach)
- findings-06-rss-ingestion-methods.md (ingestion methods)
- findings-07-what-actually-works.md (MVP features)
- findings-08-anti-patterns.md (what to avoid)

## Research Areas

### 1. Project Scaffolding
- Exact directory structure for the codebase
- Package.json / pyproject.toml dependencies with version numbers
- Docker Compose for local dev (PostgreSQL, Redis, app)
- Environment variables needed
- How projects like Miniflux and FreshRSS structure their codebases

### 2. Data Ingestion Pipeline — Step by Step
- Component 1: Feed Registry (CRUD for managing RSS sources)
- Component 2: Feed Poller (async polling loop with adaptive intervals)
- Component 3: Feed Parser (feedparser + normalization)
- Component 4: Deduplication (SimHash or similar)
- Component 5: Event Storage (normalized events into PostgreSQL)
- For each component: what library, what pattern, what the code structure looks like

### 3. Backend API — Step by Step
- FastAPI project structure (routers, models, services, schemas)
- Authentication approach (API keys? JWT? Supabase auth?)
- Core endpoints: events CRUD, sources CRUD, search, WebSocket feed
- Database connection setup (async SQLAlchemy or raw asyncpg?)
- Background task setup (Celery or native FastAPI background tasks for MVP?)

### 4. Testing Strategy
- What to test in MVP (don't over-test, don't under-test)
- pytest fixtures for database testing
- How to mock RSS feeds for testing

### 5. Sprint Plan
- Sprint 1: What gets built (specific files, specific functions)
- Sprint 2: What gets built next
- Sprint 3: What completes the MVP backend
- Definition of done for each sprint

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-12-implementation-roadmap.md
