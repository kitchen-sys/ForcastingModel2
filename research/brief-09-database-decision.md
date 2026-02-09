# Research Brief 09: Supabase vs Self-Hosted PostgreSQL — Final Decision

## Objective
Make the FINAL database decision for Sentinel. Not more comparisons — a concrete decision with implementation plan.

## Context
Read findings-05-database-ranking.md first. It has the tradeoff analysis. Now we need a DECISION and a path forward.

## Research Areas

### 1. Supabase for Conflict Monitoring — Can It Actually Handle It?
- Real-world Supabase projects handling 100K+ events per day
- Supabase real-time performance with high-frequency inserts (conflict events streaming in)
- PostGIS support in Supabase — how mature? Any gaps?
- Supabase edge functions for feed polling — practical or gimmick?
- Supabase free tier: exactly how far can we get before paying?

### 2. Self-Hosted PostgreSQL — What Does It Actually Take?
- Docker Compose setup with PostgreSQL + PostGIS + TimescaleDB
- Ops burden: backups, monitoring, upgrades for a small team
- Connection pooling (PgBouncer) necessity
- How projects like Miniflux and FreshRSS run their PostgreSQL

### 3. Hybrid Approach — Best of Both?
- Supabase for auth + API + real-time subscriptions
- Self-hosted PostgreSQL for heavy data (TimescaleDB, bulk inserts)
- Is a hybrid practical or just added complexity?

### 4. Final Recommendation
- Pick ONE path. Justify it.
- Include a complete Docker Compose snippet for the chosen approach
- Include the exact schema (CREATE TABLE statements) for: sources, events, rankings
- Include a migration plan if we start with one and need to switch later

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-09-database-decision.md
