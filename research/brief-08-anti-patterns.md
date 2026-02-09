# Research Brief 08: Anti-Patterns, Pitfalls & Failed Approaches to Avoid

## Objective
Research what NOT to do when building a real-time monitoring dashboard. Find real post-mortems, failed projects, abandoned repos, and lessons learned. The goal is to save weeks of wasted effort by learning from others' mistakes.

## Research Areas

### 1. Failed Open-Source OSINT/Dashboard Projects
- Search GitHub for abandoned conflict monitoring / OSINT dashboard projects
- Why did they fail? (Too complex, no users, bad architecture, maintenance burden)
- What tech choices led to abandonment?
- Common overengineering patterns

### 2. Database Anti-Patterns
- Supabase pitfalls: when does it break? Real user complaints
- PostgreSQL mistakes: wrong indexing, missing partitioning, N+1 queries
- Storing raw HTML/XML in the database — why it's a trap
- Not planning for data growth — what happens at 1M, 10M, 100M events
- Elasticsearch as primary store — why this fails

### 3. RSS/Feed Ingestion Anti-Patterns
- Polling too frequently and getting IP-banned
- Not handling feed format variations (RSS 1.0, 2.0, Atom, JSON Feed)
- Assuming feeds are well-formed XML
- Not implementing backoff when feeds are down
- Storing everything vs storing only what's needed
- Building a custom feed parser instead of using feedparser

### 4. Frontend Dashboard Anti-Patterns
- Rendering 10,000 markers on a map without clustering
- Real-time updates that cause memory leaks
- WebSocket connections that never reconnect
- Loading all data on initial page load
- Over-customizable dashboards nobody configures
- D3.js for everything when a chart library would suffice

### 5. Architecture Anti-Patterns
- Microservices for a team of 1-3 developers
- Kafka when Redis Streams would suffice
- GraphQL when REST is simpler and sufficient
- Building a custom auth system instead of using an existing one
- Premature optimization before having working features

### 6. Supabase-Specific Gotchas
- Real-world Supabase scaling issues
- Row-level security performance impact
- Real-time subscription limits
- Edge function cold starts
- When to choose Supabase vs when to go self-hosted

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-08-anti-patterns.md
