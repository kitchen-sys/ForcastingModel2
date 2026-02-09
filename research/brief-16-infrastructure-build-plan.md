# Build Plan Brief 16: Infrastructure, Testing & Integration — Agent Blueprint

## Objective
Create a COMPLETE build plan for Docker setup, CI/CD, testing strategy, and how all components integrate. This is the glue that makes everything work together.

## Context — Read These First
- /home/user/ForcastingModel2/research/findings-02-architecture.md
- /home/user/ForcastingModel2/research/findings-08-anti-patterns.md
- /home/user/ForcastingModel2/research/findings-09-database-decision.md
- /home/user/ForcastingModel2/research/findings-12-implementation-roadmap.md

## What This Document Must Contain

### 1. Docker Setup
- Complete `docker-compose.yml` for local development
- Services: PostgreSQL (with PostGIS + TimescaleDB), Redis, backend (FastAPI), frontend (Next.js), Celery worker
- Volume mounts, port mappings, health checks
- Environment variable files (.env.example)
- Dockerfile for backend
- Dockerfile for frontend
- Docker startup order and dependency management

### 2. Database Initialization
- Init scripts that run on first docker-compose up
- Schema creation (all tables from findings-09)
- Seed data loading (Tier 1 sources from findings-11)
- TimescaleDB hypertable setup
- PostGIS extension enabling

### 3. Testing Strategy
- What to test and what NOT to test in MVP
- pytest configuration for backend
- Test fixtures: database, mock feeds, sample events
- How to mock RSS feeds for testing (local XML fixtures)
- Frontend testing: what's worth testing vs overkill for MVP
- Integration test: end-to-end feed → parse → store → display

### 4. CI/CD Pipeline
- GitHub Actions workflow file
- Steps: lint, type check, test, build, (optional: deploy)
- When to run: on push, on PR
- Keep it simple — no over-engineering

### 5. Environment Configuration
- All environment variables documented
- .env.example file contents
- Config loading pattern (pydantic-settings for backend)
- Frontend env vars (NEXT_PUBLIC_ prefix)

### 6. Monitoring & Logging (MVP Level)
- Structured logging setup (Python logging, not a complex stack)
- Key metrics to track: feed polling success rate, event ingest rate, API latency
- Health check endpoints
- What to skip for MVP (no Prometheus/Grafana yet)

### 7. Project Root Files
- README.md structure (setup instructions, architecture overview)
- Makefile or just docker-compose commands
- .gitignore contents
- Pre-commit hooks (optional for MVP)

### 8. Integration Architecture
- How backend talks to database
- How ingestion pipeline feeds events to backend
- How frontend connects to backend (REST + WebSocket)
- How real-time updates flow end-to-end
- Sequence diagram (text-based) for: new event arrives → user sees it

### 9. Agent Instructions — Build Order
- What a coding agent should build FIRST across the entire project
- Cross-component dependencies
- "Build Docker setup first → database schema → backend skeleton → ingestion pipeline → frontend scaffold"
- Verification checkpoints at each stage

## Deliverable
Write to: /home/user/ForcastingModel2/research/findings-16-infrastructure-build-plan.md
