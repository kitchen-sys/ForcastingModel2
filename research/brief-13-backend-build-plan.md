# Build Plan Brief 13: Backend API — Agent Blueprint

## Objective
Create a COMPLETE build plan that a coding agent can follow to build the FastAPI backend from scratch. Every file, every function, every endpoint — spelled out.

## Context — Read These First
- /home/user/ForcastingModel2/research/findings-02-architecture.md
- /home/user/ForcastingModel2/research/findings-09-database-decision.md
- /home/user/ForcastingModel2/research/findings-10-mvp-scope.md
- /home/user/ForcastingModel2/research/findings-12-implementation-roadmap.md

## What This Document Must Contain

### 1. Directory Structure
- Exact file tree for `/src/backend/`
- Every file that needs to be created, with its purpose

### 2. Dependencies
- Complete `pyproject.toml` or `requirements.txt` with pinned versions
- Why each dependency is needed

### 3. Database Setup
- Complete SQL schema (CREATE TABLE statements) for all tables
- Alembic migration setup
- Connection pooling config
- Database initialization script

### 4. API Endpoints — Full Specification
For EACH endpoint:
- HTTP method + path (e.g., `GET /api/v1/events`)
- Request parameters / body schema (Pydantic model)
- Response schema (Pydantic model)
- Business logic description
- Error cases

### 5. Pydantic Models
- Every request/response model with field types
- Shared base models
- Validation rules

### 6. Service Layer
- Function signatures for each service
- Business logic pseudocode
- Database query patterns

### 7. WebSocket Specification
- Channel names and message formats
- Connection/disconnection handling
- What data streams over WebSocket

### 8. Configuration
- Environment variables needed
- Config file structure
- Secrets management approach

### 9. Agent Instructions
- Step-by-step build order for a coding agent
- "Build file X first, then file Y because it depends on X"
- Testing checkpoints: "After building X, run Y to verify"

## Deliverable
Write to: /home/user/ForcastingModel2/research/findings-13-backend-build-plan.md
