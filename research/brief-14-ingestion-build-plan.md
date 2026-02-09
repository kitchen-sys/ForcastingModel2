# Build Plan Brief 14: Data Ingestion Pipeline — Agent Blueprint

## Objective
Create a COMPLETE build plan that a coding agent can follow to build the RSS/data ingestion pipeline from scratch. Every component, every function, every config — spelled out.

## Context — Read These First
- /home/user/ForcastingModel2/research/findings-01-data-sources.md
- /home/user/ForcastingModel2/research/findings-06-rss-ingestion-methods.md
- /home/user/ForcastingModel2/research/findings-11-source-priority.md
- /home/user/ForcastingModel2/research/findings-12-implementation-roadmap.md

## What This Document Must Contain

### 1. Directory Structure
- Exact file tree for `/src/ingestion/`
- Every file that needs to be created, with its purpose

### 2. Dependencies
- All Python packages needed (feedparser, httpx, celery, redis, etc.)
- Pinned versions

### 3. Feed Registry Component
- How sources are stored and managed in the database
- CRUD operations for adding/removing/updating feeds
- Source health tracking schema
- Default seed data: the Tier 1 sources from findings-11 with real URLs

### 4. Feed Poller Component
- Async polling loop architecture
- Adaptive polling intervals (how to calculate)
- ETag/Last-Modified header handling
- Rate limiting and backoff logic
- Error handling for dead/broken feeds
- Exact function signatures

### 5. Feed Parser Component
- RSS/Atom/JSON Feed parsing with feedparser
- Normalization: mapping different feed formats to a unified event schema
- Field extraction: title, description, date, source, URL, categories
- Handling encoding issues, malformed XML, missing fields
- Exact function signatures

### 6. Deduplication Component
- Algorithm choice (SimHash, URL-based, or title fingerprint)
- Implementation details
- Where dedup runs in the pipeline
- How to handle near-duplicates vs exact duplicates
- Exact function signatures

### 7. Event Storage Component
- How parsed/deduped events get written to PostgreSQL
- Batch insert patterns
- Conflict resolution (upsert logic)
- Exact function signatures

### 8. Source Ranking/Scoring Component
- How source reliability scores are calculated
- Metrics: uptime, freshness, content quality, relevance
- Score update frequency
- Exact function signatures

### 9. Pipeline Orchestration
- How all components connect: Poller → Parser → Dedup → Store
- Celery task definitions or async pipeline flow
- Error handling at each stage
- Logging and monitoring hooks

### 10. Seed Configuration
- JSON/YAML config file with all Tier 1 RSS feeds
- Pre-populated source entries ready to load

### 11. Agent Instructions
- Step-by-step build order for a coding agent
- Which file to create first, second, third
- Testing checkpoints at each stage
- "After building the poller, test it against these 3 feeds"

## Deliverable
Write to: /home/user/ForcastingModel2/research/findings-14-ingestion-build-plan.md
