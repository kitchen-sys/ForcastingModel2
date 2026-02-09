# CLAUDE.md - Sentinel Project Configuration

## Project Overview

**Sentinel** is a Palantir Gotham-inspired global conflict intelligence dashboard. It monitors RSS feeds, OSINT sources, and conflict databases in real-time to deliver a professional, intelligence-grade operational picture of worldwide conflicts and security events.

The platform ingests data from sources like ACLED, GDELT, UCDP, ReliefWeb, and dozens of RSS feeds, then applies NLP and AI analytics to produce entity-linked, geospatially-enriched, temporal conflict intelligence. The goal is a production-quality tool that rivals commercial platforms like Dataminr, Recorded Future, and Palantir Foundry in analytical capability.

---

## Project Structure

```
/home/user/ForcastingModel2/
├── CLAUDE.md              # This file - project config and rules (read every session)
├── AGENT_RULES.md         # Strict behavioral rules for sub-agents
├── research/              # Research briefs (input) and findings (output)
│   ├── brief-XX-*.md      # Research task definitions
│   └── findings-XX-*.md   # Completed research output
├── src/                   # Source code (to be created)
│   ├── backend/           # Python FastAPI backend
│   ├── frontend/          # TypeScript Next.js frontend
│   ├── ingestion/         # Data ingestion pipeline
│   └── analytics/         # AI/NLP analytics engine
├── docs/                  # Documentation (to be created)
├── docker/                # Docker and deployment configs
└── tests/                 # Test suites
```

---

## Development Phases

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Research & Planning | **CURRENT** |
| **Phase 2** | Data Ingestion Pipeline | Pending |
| **Phase 3** | Backend API & Processing | Pending |
| **Phase 4** | Frontend Dashboard | Pending |
| **Phase 5** | AI/NLP Analytics Engine | Pending |
| **Phase 6** | Integration & Testing | Pending |
| **Phase 7** | Deployment | Pending |

---

## Agent Deployment Rules (CRITICAL)

These rules are non-negotiable. Every agent launch must follow them.

### Launch Protocol

1. **ALWAYS** include the contents of `AGENT_RULES.md` in the agent prompt. No exceptions. Use this template at the START of every agent prompt:

```
MANDATORY: Before doing anything else, read and internalize /home/user/ForcastingModel2/AGENT_RULES.md. These rules are NON-NEGOTIABLE. You have a hard limit of 10 research tool calls before you MUST start writing your output file. Your output file is your ONLY deliverable. An incomplete file beats no file.
```

2. Set `max_turns` to **25** for research agents.
3. **ALWAYS** run agents with `run_in_background: true`.
4. **Never** launch more than **4 agents simultaneously**. Wait for one to finish before starting a fifth.
5. **ALWAYS** tell the agent the exact output file path in the prompt (e.g., `/home/user/ForcastingModel2/research/findings-XX-topic.md`).

### Monitoring Protocol

1. Check on running agents every **2-3 minutes**.
2. If an agent has not written its output file by **15 tool calls**, resume it with an explicit **"WRITE NOW"** nudge directing it to write its output immediately.
3. Verify each agent's output file exists and contains substantive content before considering the task complete.

### Output Enforcement

- Agents **MUST** write their output file. This is non-negotiable.
- An agent that completes without writing its designated output file has **failed**.
- Output files must contain actionable, specific findings -- not vague summaries.

---

## Research Phase Rules

All research artifacts live in `/research/`.

### File Naming Convention

- **Briefs** (input): `brief-XX-<topic>.md` -- defines the research task
- **Findings** (output): `findings-XX-<topic>.md` -- completed research

### Research Quality Standards

- Each research domain gets its own dedicated agent.
- Findings must be **actionable and specific**: real URLs, real API endpoints, real library names, real version numbers, real pricing.
- No filler content. No vague recommendations. Every claim must be backed by a concrete source or example.
- Include code snippets, configuration examples, and schema samples where relevant.

### Completed Research

- `findings-01-data-sources.md` -- Data sources and ingestion pipeline research
- `findings-02-architecture.md` -- System architecture and tech stack research
- `findings-03-ai-analytics.md` -- AI/NLP analytics engine research
- `findings-04-ui-visualization.md` -- UI design and visualization research

---

## Tech Stack

### Frontend
- **Framework**: TypeScript + Next.js (App Router)
- **State Management**: Zustand + React Query (TanStack Query)
- **Mapping**: Mapbox GL JS or Deck.gl for geospatial visualization
- **Charts**: D3.js, Recharts, or Tremor for analytics panels
- **Real-time**: WebSocket connections for live event streaming
- **Design**: Dark theme, intelligence-grade UI. Clean typography, high information density, muted color palette with signal-color highlights for alerts.

### Backend
- **Framework**: Python FastAPI
- **Task Queue**: Celery with Redis broker for background ingestion jobs
- **Streaming**: WebSockets / Server-Sent Events for real-time push
- **API Design**: REST for CRUD, WebSocket for streaming, GraphQL optional for complex queries

### Data Layer
- **Primary DB**: PostgreSQL with PostGIS (geospatial) + TimescaleDB (time-series)
- **Cache**: Redis for dashboard caching and rate limiting
- **Search**: Elasticsearch for full-text search across events and entities
- **Graph** (optional): Neo4j for entity relationship mapping and link analysis
- **Message Queue**: Redis Streams or Apache Kafka for ingestion pipeline

### Infrastructure
- Docker Compose for local development
- Kubernetes for production deployment
- CI/CD via GitHub Actions
- Monitoring via Prometheus + Grafana

---

## Code Standards

### General

- All code must be **production-quality**. No prototypes, no "we'll fix it later" shortcuts.
- Comprehensive error handling. Every external API call must have retry logic and graceful degradation.
- Type safety everywhere. TypeScript strict mode on frontend. Python type hints on backend.
- All public functions and API endpoints must have docstrings/documentation.

### Python (Backend)

- Python 3.11+
- Use `async`/`await` throughout FastAPI handlers
- Pydantic models for all request/response schemas
- Alembic for database migrations
- pytest for testing

### TypeScript (Frontend)

- Strict TypeScript -- no `any` types
- Functional components with hooks
- Server components where appropriate (Next.js App Router)
- Tailwind CSS for styling
- Component-driven architecture with a shared design system

### Testing

- Unit tests for all business logic
- Integration tests for API endpoints
- E2E tests for critical dashboard workflows

---

## Git Workflow

- **Branch**: `claude/clear-repo-new-project-cYd9I`
- Commit early and often.
- Push after every significant milestone.
- Commit messages must clearly describe what was added or changed.
- Keep commits atomic -- one logical change per commit.

---

## Key Data Sources (Reference)

These are the primary data sources identified during research:

| Source | Type | Priority |
|--------|------|----------|
| ACLED | Conflict event database / API | High |
| GDELT | Global event database | High |
| UCDP | Academic conflict dataset | Medium |
| UN OCHA ReliefWeb | Humanitarian crisis API | High |
| NASA FIRMS | Fire/hotspot satellite data | Medium |
| SIPRI | Arms and military expenditure | Medium |
| Reuters/AP/BBC RSS | Real-time news feeds | High |
| Crisis Group RSS | Conflict analysis feeds | High |
| Telegram/OSINT channels | Social media intelligence | Medium |

---

## Session Startup Checklist

At the start of every session:

1. Read this file (`CLAUDE.md`).
2. Check current phase status and what work has been completed.
3. Review any existing findings in `/research/` if relevant to the current task.
4. Confirm the git branch is `claude/clear-repo-new-project-cYd9I`.
5. Ask for clarification if the task is ambiguous -- do not assume.
