# Research Brief 10: MVP Scope Definition — Exactly What to Build First

## Objective
Define the EXACT MVP scope for Sentinel. Not vague priorities — specific features, specific screens, specific API endpoints. A developer should read this and know exactly what to build on day 1.

## Context
Read these files first:
- findings-07-what-actually-works.md (real platforms, MVP features, analyst workflows)
- findings-08-anti-patterns.md (what NOT to build)

## Research Areas

### 1. MVP Feature List — Ruthless Prioritization
- Research what features Liveuamap, ACLED dashboard, and Dataminr had in their V1
- What is the absolute minimum to make Sentinel USEFUL to someone?
- Define 3 tiers: Must Have (launch blocker), Should Have (week 2), Nice to Have (later)
- For each feature: what it does, why it matters, estimated complexity (S/M/L)

### 2. MVP Screens & Layout
- Exactly which pages/views does the MVP need?
- Main dashboard layout (map + feed + filters?)
- Event detail view
- Source management view
- What can be a V2 screen?

### 3. MVP API Endpoints
- List every API endpoint the MVP needs
- REST endpoint paths, methods, request/response shapes
- WebSocket channels for real-time updates
- What endpoints can wait for V2?

### 4. MVP Data Flow
- End-to-end flow: RSS feed → parse → store → display
- What processing happens at each step in MVP (minimal NLP vs full pipeline)?
- What enrichment can be deferred to V2?

### 5. Build Order
- What gets built first, second, third?
- Dependencies between components
- Estimated effort per component (not time — relative sizing S/M/L/XL)

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-10-mvp-scope.md
