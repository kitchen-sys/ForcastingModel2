# Research Brief 05: Database Architecture for RSS Source Management & Ranking

## Objective
Research the best database solution for storing, managing, and ranking RSS/data sources for a real-time conflict monitoring dashboard. Compare Supabase vs self-hosted PostgreSQL vs other options.

## Research Areas

### 1. Supabase vs Custom PostgreSQL
- Supabase features: real-time subscriptions, auth, edge functions, storage
- Supabase pricing tiers and limits (free tier, pro, enterprise)
- Supabase row limits, API rate limits, connection limits
- Self-hosted PostgreSQL: full control, no limits, more ops work
- Supabase PostgREST API vs custom FastAPI
- Can Supabase handle real-time conflict event streaming at scale?
- Supabase real-time: channels, broadcast, presence — practical limits

### 2. RSS Source Ranking System
- How to score RSS feed reliability (uptime, latency, content quality)
- Feed freshness scoring (how often does it actually update?)
- Content relevance scoring for conflict data
- Source credibility/authority ranking (Reuters > random blog)
- Automated quality monitoring — detecting dead feeds, duplicate content
- Schema design for source metadata, scores, and health tracking

### 3. Database Schema for Source Management
- Sources table: URL, name, type, category, reliability score, last checked, status
- Feed health tracking: uptime percentage, avg latency, error count
- Content quality metrics: relevance score, duplication rate, entity richness
- Ranking algorithm: weighted composite score from multiple factors
- How to store and query ranked sources efficiently

### 4. Data Storage for Ingested Events
- Schema for normalized conflict events from multiple sources
- Time-series considerations for event data
- Full-text search across events
- Geospatial indexing for location-based queries
- Supabase pgvector for semantic search — is it practical?

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-05-database-ranking.md
