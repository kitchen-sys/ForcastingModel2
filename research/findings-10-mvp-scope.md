# Findings: MVP Scope Definition -- Exactly What to Build First

## Executive Summary

This document defines the exact MVP scope for Sentinel. It specifies every feature (in three priority tiers with complexity estimates), every screen (with layout descriptions), every API endpoint (with paths, methods, and request/response shapes), the complete end-to-end data flow, and the precise build order with dependencies. The MVP is ruthlessly scoped: an interactive conflict map, a filterable event feed, a timeline chart, RSS + ACLED data ingestion, and CSV export. That is the entire V1. Everything else -- NLP, forecasting, entity graphs, multi-source fusion, social media ingestion -- is deferred until this core loop is proven and used daily. Liveuamap launched in 2014 with nothing more than a map, a web crawler, and human editors covering a single conflict (Ukraine). Sentinel's MVP follows the same principle: solve one workflow completely before expanding scope.

## Research Scope

This document responds to Research Brief 10: "MVP Scope Definition -- Exactly What to Build First." It builds on findings-07 (what actually works in conflict dashboards) and findings-08 (anti-patterns and failures to avoid). It covers:

- MVP feature list with ruthless three-tier prioritization and complexity estimates
- Exact screens and layouts the MVP needs
- Every API endpoint with full specification
- End-to-end data flow from source to screen
- Build order with explicit dependencies between components

---

## Detailed Findings

### 1. MVP Feature List -- Ruthless Prioritization

The feature tiers are informed by what Liveuamap, ACLED Dashboard, and analyst workflows demonstrate as essential. Liveuamap launched in February 2014 with just a map and web crawler covering Ukraine. ACLED's dashboard started as Tableau embeds over their dataset. Both proved that a map + feed + filters is sufficient to deliver real value.

Complexity estimates use T-shirt sizing: S (1-2 days), M (3-5 days), L (1-2 weeks), XL (2+ weeks).

#### Tier 1: Must Have (Launch Blockers)

These features are required for Sentinel to be useful to a single analyst on day one.

| # | Feature | What It Does | Why It Matters | Complexity |
|---|---------|-------------|----------------|------------|
| 1 | Interactive conflict map | Mapbox GL JS map with event markers, color-coded by event type (battles=red, protests=blue, violence against civilians=orange, explosions=yellow, riots=purple). Markers clustered via Supercluster at low zoom levels. | Primary analyst view. Analysts orient spatially first. Liveuamap proves this is THE feature. | L |
| 2 | Event feed / data table | Scrollable, sortable table showing: date, location, event type, actors, fatalities, source. Sortable by any column. Click to select event and highlight on map. | Where 60% of deep analysis happens after spatial orientation. Every conflict dashboard has this. | M |
| 3 | Date range filter | Calendar picker for start/end date. Applies to map, feed, and timeline simultaneously. Presets: Today, Last 7 Days, Last 30 Days, Last 90 Days, Custom. | "What happened this week?" is the single most common analyst question. | S |
| 4 | Event type filter | Multi-select checkboxes for ACLED event types: Battles, Explosions/Remote Violence, Violence Against Civilians, Protests, Riots, Strategic Developments. | Analysts filter by event type constantly. Military analysts want battles; humanitarian analysts want violence against civilians. | S |
| 5 | Country/region filter | Dropdown with search for country selection. Multi-select supported. Applies to map (zooms to country), feed (filters), and timeline. | Analysts have geographic areas of responsibility. | S |
| 6 | Basic timeline chart | Bar chart showing event count per day/week over the selected date range. Uses Recharts. Single metric: event count. Fatality overlay as optional toggle. | Trend awareness is the second most critical analytical view after the map. | M |
| 7 | CSV export | "Export" button on the event feed. Exports currently filtered events as CSV with all columns. | Non-negotiable. Analysts always need to pull data into Excel, R, or Python. They will leave without this. | S |
| 8 | ACLED data ingestion | Celery background task that fetches ACLED conflict events via their REST API. Runs daily. Normalizes into canonical event schema. Deduplicates by ACLED event_id. | ACLED is the backbone data source. Human-coded, high accuracy, global coverage. | L |
| 9 | RSS feed ingestion | Celery background tasks polling 5-10 curated RSS feeds (Reuters, BBC, Al Jazeera, ICG, ReliefWeb). Uses feedparser library. Conditional requests (ETag/Last-Modified). Exponential backoff on errors. Deduplication via content hash. | Real-time awareness between ACLED's weekly updates. RSS is simple, free, no API keys. | L |
| 10 | Canonical event schema | PostgreSQL table with PostGIS geography column. Fields: id, title, description, event_type, event_date, latitude, longitude, geography(point), country, region, actors[], fatalities, source_name, source_url, source_type (ACLED/RSS), confidence_score, ingested_at, raw_data(jsonb). | Every data source normalizes into this schema. This is the foundation all views query against. | M |

**Total Tier 1 Complexity: ~4-6 weeks for a solo developer, ~2-3 weeks for a team of 2.**

#### Tier 2: Should Have (Week 3-5, After Core Is Proven)

| # | Feature | What It Does | Why It Matters | Complexity |
|---|---------|-------------|----------------|------------|
| 11 | Full-text search | Search box that queries event titles, descriptions, and actor names. Uses PostgreSQL tsvector initially (not Elasticsearch). Results appear in the event feed. | Fourth most-used feature by analysts. Needed for "find all events mentioning Boko Haram." | M |
| 12 | Map-feed synchronization | Clicking a marker in the map highlights the corresponding row in the feed. Clicking a row in the feed pans the map to that event. Bidirectional. | Core interaction model. Map and feed must feel like one unified view. | M |
| 13 | Event detail panel | Side panel or modal that opens when an event is selected. Shows full description, all actors, source link, location details, and a mini-map. | Analysts need to drill into individual events without leaving the dashboard. | M |
| 14 | Subnational drill-down | When a country is selected, the map zooms in and shows admin-1 (province) boundaries. Click a province to filter events to that area. | Analysts covering a country need to see which provinces are hottest. | L |
| 15 | Historical baseline comparison | For any selected country/region, show a dashed line on the timeline chart representing the rolling 52-week average. Current week highlighted in color against the baseline. | "Is this normal?" is the most important analytical question after "what happened?" | M |
| 16 | WebSocket live event push | When new events are ingested, push them to connected dashboard clients via WebSocket. New events appear at the top of the feed with a highlight animation. Badge counter shows "X new events." | Transforms dashboard from periodic-refresh to live awareness. Critical for breaking events. | L |
| 17 | Source management page | Admin page listing all configured RSS feeds and API sources. Shows status (active/error/paused), last poll time, items ingested, error count. Enable/disable toggle per source. | Operators need to monitor ingestion health and manage sources. | M |

**Total Tier 2 Complexity: ~3-4 weeks additional.**

#### Tier 3: Nice to Have (Month 2-3)

| # | Feature | What It Does | Why It Matters | Complexity |
|---|---------|-------------|----------------|------------|
| 18 | Alert thresholds | Configurable per-country alerts. When weekly event count exceeds N% above the 52-week rolling average, send notification. In-app notification bell + optional email digest. | Proactive awareness. Analysts should not have to check every country manually. | L |
| 19 | Country profile pages | Dedicated page per country: map zoomed to country, timeline chart, top actors list, recent events table, conflict index score. | One-stop briefing page for area specialists. | L |
| 20 | Actor tracking | Table of actors (armed groups, state forces) with event counts, fatality counts, active regions. Click to filter events by actor. | Understanding who is fighting whom. Important for deep analysis. | L |
| 21 | ReliefWeb integration | Ingest ReliefWeb situation reports via their API. Display as context cards alongside conflict events for the same country. | Humanitarian context enriches conflict data. | M |
| 22 | Saved filters / bookmarks | Let analysts save their current filter configuration (date range + countries + event types) as a named bookmark. Load bookmarks to restore the view. | Analysts check the same filtered view every morning. | M |
| 23 | Dark/light theme toggle | Toggle between dark theme (default, intelligence-grade) and light theme. Store preference in localStorage. | Some analysts prefer light mode. Accessibility consideration. | S |
| 24 | User authentication | Login/registration using Auth.js (NextAuth). JWT-based session. Role-based access (admin vs analyst). | Required before multi-user deployment. Use existing auth library, never build custom. | L |

**Total Tier 3 Complexity: ~4-6 weeks additional.**

#### Explicitly Deferred to V2+

| Feature | Why Deferred |
|---------|-------------|
| NLP / entity extraction pipeline | Complex, requires training data, marginal value until data flow is proven. Findings-07 confirms this. |
| Forecasting / predictive models | ACLED CAST already does this. Integrate their output instead of building from scratch. |
| Entity relationship graphs (Neo4j) | Deep investigation feature, not daily monitoring. Build only after user demand. |
| Real-time social media ingestion (Twitter/Telegram) | Requires massive infrastructure, API costs, content moderation. Use curated feeds first. |
| AI-generated briefings | Trust gap. Analysts do not trust AI summaries for operational decisions (findings-07). |
| 3D globe visualization | Demo-ware. Slower than 2D maps, no analytical advantage (findings-07 and findings-08). |
| GDELT integration | ~55% field accuracy, ~20% duplication. Not worth the effort for V1 (findings-07). |
| Elasticsearch | PostgreSQL tsvector is sufficient for MVP search. Add ES as secondary index only when measured search performance demands it (findings-08). |
| Kafka / complex message queuing | Celery + Redis handles Sentinel's event volume. Kafka is overkill (findings-08). |
| Microservices architecture | Modular monolith is correct for a small team. Extract services only when proven necessary (findings-08). |

---

### 2. MVP Screens & Layout

The MVP has exactly 3 screens. Everything else is V2.

#### Screen 1: Main Dashboard (Route: `/dashboard`)

This is the primary screen. It follows the "map + feed + timeline" pattern proven by Liveuamap, ACLED Dashboard, and every successful conflict monitoring tool.

```
+------------------------------------------------------------------+
|  SENTINEL                          [Search Box]    [Export CSV]   |
+------------------------------------------------------------------+
|  Filters Bar                                                     |
|  [Date Range: ▼] [Event Types: ▼] [Countries: ▼] [Clear All]    |
+-------------------------------+----------------------------------+
|                               |                                  |
|                               |  EVENT FEED                      |
|                               |  +----------------------------+  |
|    INTERACTIVE MAP            |  | Date | Location | Type |...|  |
|    (Mapbox GL JS)             |  |------|----------|------|---|  |
|                               |  | 2025-02-08 | Kyiv | B  |   |  |
|    - Color-coded markers      |  | 2025-02-08 | Gaza | E  |   |  |
|    - Supercluster clustering  |  | 2025-02-07 | Khar | V  |   |  |
|    - Click marker = highlight |  | 2025-02-07 | Orom | R  |   |  |
|      event in feed            |  | ...                       |  |
|                               |  +----------------------------+  |
|    Width: 60%                 |  Width: 40%                      |
|                               |  Scrollable, sortable            |
+-------------------------------+----------------------------------+
|  TIMELINE CHART (Recharts bar chart)                             |
|  Event count by day/week over selected date range                |
|  ████ ██ ████████ ███ ██████ ████ ███████ ████ ██               |
|  Height: ~200px, full width                                      |
+------------------------------------------------------------------+
```

**Layout details:**
- **Top bar:** App name (left), global search box (center), export button (right).
- **Filter bar:** Horizontal row of filter dropdowns below the top bar. Sticky on scroll.
- **Main content area:** Split 60/40 between map (left) and event feed (right). Resizable divider optional in V2.
- **Map:** Full height of main content area. Mapbox GL JS with dark style (mapbox://styles/mapbox/dark-v11). Markers colored by event type. Supercluster for zoom-dependent clustering.
- **Event feed:** Scrollable data table. Columns: Date, Location, Event Type, Actors, Fatalities, Source. Click row to pan map. Sort by clicking column headers.
- **Timeline:** Fixed at bottom. Full width. Recharts BarChart. Height ~200px. X-axis: dates. Y-axis: event count. Optional fatality line overlay.
- **Event detail panel:** Appears as a right slide-out panel when an event is selected (Tier 2 feature). Overlays the event feed.

#### Screen 2: Source Management (Route: `/dashboard/sources`)

Admin page for monitoring and managing data sources.

```
+------------------------------------------------------------------+
|  SENTINEL  /  Source Management                                  |
+------------------------------------------------------------------+
|                                                                  |
|  DATA SOURCES                                        [Add Feed]  |
|  +------------------------------------------------------------+ |
|  | Source Name  | Type | Status | Last Poll  | Items | Errors | |
|  |-------------|------|--------|------------|-------|--------| |
|  | ACLED API   | API  | Active | 2m ago     | 43221 | 0      | |
|  | Reuters RSS | RSS  | Active | 5m ago     | 8432  | 2      | |
|  | BBC World   | RSS  | Active | 8m ago     | 6218  | 0      | |
|  | ICG RSS     | RSS  | Error  | 2h ago     | 1842  | 15     | |
|  | Al Jazeera  | RSS  | Paused | 1d ago     | 3102  | 0      | |
|  +------------------------------------------------------------+ |
|                                                                  |
|  INGESTION LOG (last 24h)                                        |
|  [Time]  [Source]  [Action]  [Count]  [Status]                   |
|  14:32   Reuters   Poll      12 new   Success                    |
|  14:30   ACLED     Sync      0 new    Success (no new data)      |
|  14:28   BBC       Poll      3 new    Success                    |
|  14:15   ICG       Poll      --       Error: 503 Service Unavail |
|                                                                  |
+------------------------------------------------------------------+
```

**Layout details:**
- Table of all configured data sources with status indicators (green=active, red=error, gray=paused).
- Action buttons per row: Enable/Disable toggle, Force Poll, View Errors.
- "Add Feed" button opens a form to add a new RSS feed URL.
- Below: scrollable ingestion log showing recent poll activity.

#### Screen 3: Event Detail Page (Route: `/dashboard/events/[id]`)

Standalone page for a single event with full details. Linked from the event feed.

```
+------------------------------------------------------------------+
|  SENTINEL  /  Event Detail                          [Back to Map] |
+------------------------------------------------------------------+
|                                                                  |
|  EVENT: Shelling hits residential area in eastern Kharkiv         |
|                                                                  |
|  +----------------------------+  Event Type: Explosions/Remote   |
|  |                            |  Date: 2025-02-07               |
|  |   MINI MAP                 |  Location: Kharkiv, Ukraine     |
|  |   (centered on event)      |  Coordinates: 49.9935, 36.2304  |
|  |   ★ marker                 |  Fatalities: 3                  |
|  |                            |  Source: ACLED                   |
|  +----------------------------+  Confidence: High                |
|                                                                  |
|  Actors:                                                         |
|  - Military Forces of Russia (2022-)                             |
|  - Civilians (Ukraine)                                           |
|                                                                  |
|  Description:                                                    |
|  On 7 February 2025, Russian forces shelled a residential area   |
|  in the Kyivskyi district of Kharkiv. Three civilians were       |
|  killed and twelve were injured in the attack...                 |
|                                                                  |
|  Source: https://acleddata.com/... [Open Source Link]             |
|                                                                  |
|  Raw Data (collapsible):                                         |
|  { "event_id_cnty": "UKR12345", ... }                           |
|                                                                  |
+------------------------------------------------------------------+
```

**Layout details:**
- Mini-map (300x250px) centered on the event coordinates with a single marker.
- All event metadata displayed clearly.
- Link back to dashboard with the event pre-selected.
- Raw data section (collapsible) showing the original JSON from the data source.

#### Screens Deferred to V2

| Screen | Why Deferred |
|--------|-------------|
| Country Profile Page | Requires historical baseline calculation, actor aggregation -- Tier 3 feature. |
| Actor Profile Page | Requires actor tracking infrastructure -- Tier 3 feature. |
| Alert Configuration Page | Requires baseline comparison engine -- Tier 3 feature. |
| User Settings / Profile | Requires authentication -- Tier 3 feature. |
| Admin Panel | Not needed until multi-user deployment. |

---

### 3. MVP API Endpoints

Every endpoint the MVP needs, specified with path, method, and request/response shapes.

#### REST Endpoints

**Events**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/events` | List events with filters | Query params (below) | `{ data: Event[], meta: { total, limit, offset } }` |
| GET | `/api/v1/events/{id}` | Get single event | Path param: id (UUID) | `{ data: Event }` |
| GET | `/api/v1/events/export` | Export filtered events as CSV | Same query params as list | CSV file download |

**Event list query parameters:**

```
GET /api/v1/events?
  date_from=2025-01-01          # ISO date, inclusive
  &date_to=2025-02-08           # ISO date, inclusive
  &event_type=battles,riots     # Comma-separated ACLED types
  &country=Ukraine,Syria        # Comma-separated country names
  &bbox=-74.46,40.78,35.2,50.1  # Bounding box: minLng,minLat,maxLng,maxLat
  &search=Kharkiv               # Full-text search (title, description, actors)
  &sort=-event_date             # Sort field, prefix - for descending
  &limit=50                     # Items per page (default 50, max 500)
  &offset=0                     # Pagination offset
```

**Event response shape (JSON):**

```json
{
  "data": [
    {
      "id": "uuid-v4",
      "title": "Shelling hits residential area in eastern Kharkiv",
      "description": "On 7 February 2025, Russian forces shelled...",
      "event_type": "Explosions/Remote violence",
      "event_date": "2025-02-07",
      "latitude": 49.9935,
      "longitude": 36.2304,
      "country": "Ukraine",
      "region": "Kharkiv",
      "actors": ["Military Forces of Russia (2022-)", "Civilians (Ukraine)"],
      "fatalities": 3,
      "source_name": "ACLED",
      "source_url": "https://acleddata.com/...",
      "source_type": "api",
      "confidence_score": 0.95,
      "ingested_at": "2025-02-08T14:32:00Z"
    }
  ],
  "meta": {
    "total": 12453,
    "limit": 50,
    "offset": 0
  }
}
```

**Map Clusters**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/events/clusters` | Get clustered event markers for map | Query: bbox, zoom, date_from, date_to, event_type, country | `{ data: Cluster[] }` |

**Cluster response shape:**

```json
{
  "data": [
    {
      "cluster_id": 1,
      "latitude": 49.99,
      "longitude": 36.23,
      "count": 47,
      "dominant_type": "Explosions/Remote violence",
      "expansion_zoom": 8
    },
    {
      "latitude": 33.51,
      "longitude": 36.29,
      "count": 1,
      "event_id": "uuid-v4",
      "event_type": "Battles",
      "title": "Clashes in rural Damascus"
    }
  ]
}
```

Note: When `count` is 1, the cluster is an individual event and includes event details. When `count` > 1, it is an aggregate cluster. The `expansion_zoom` tells the frontend what zoom level will break this cluster apart.

**Timeline / Aggregation**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/events/timeline` | Aggregated event counts for timeline chart | Query: date_from, date_to, interval (day/week/month), event_type, country | `{ data: TimelineBucket[] }` |

**Timeline response shape:**

```json
{
  "data": [
    { "date": "2025-02-01", "count": 142, "fatalities": 38 },
    { "date": "2025-02-02", "count": 167, "fatalities": 45 },
    { "date": "2025-02-03", "count": 131, "fatalities": 22 }
  ],
  "meta": {
    "interval": "day",
    "total_events": 12453,
    "total_fatalities": 3102
  }
}
```

**Sources (Admin)**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/sources` | List all configured data sources | None | `{ data: Source[] }` |
| POST | `/api/v1/sources` | Add a new RSS feed source | `{ name, url, type, poll_interval_minutes }` | `{ data: Source }` |
| PATCH | `/api/v1/sources/{id}` | Update source (enable/disable, change interval) | `{ enabled?, poll_interval_minutes? }` | `{ data: Source }` |
| DELETE | `/api/v1/sources/{id}` | Remove a data source | Path param: id | `{ success: true }` |
| POST | `/api/v1/sources/{id}/poll` | Force immediate poll of a source | Path param: id | `{ data: { status: "polling_started" } }` |

**Source response shape:**

```json
{
  "data": {
    "id": "uuid-v4",
    "name": "Reuters World News",
    "url": "https://feeds.reuters.com/reuters/worldNews",
    "type": "rss",
    "enabled": true,
    "poll_interval_minutes": 30,
    "last_poll_at": "2025-02-08T14:32:00Z",
    "last_poll_status": "success",
    "total_items_ingested": 8432,
    "error_count_24h": 2,
    "created_at": "2025-01-15T10:00:00Z"
  }
}
```

**Ingestion Log**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/sources/logs` | Recent ingestion activity | Query: source_id, limit (default 100) | `{ data: LogEntry[] }` |

**Health / System**

| Method | Path | Description | Request | Response |
|--------|------|-------------|---------|----------|
| GET | `/api/v1/health` | System health check | None | `{ status: "ok", db: "ok", redis: "ok", sources_active: 6 }` |
| GET | `/api/v1/stats` | Dashboard summary statistics | None | `{ total_events, events_today, active_sources, countries_covered }` |

#### WebSocket Endpoint (Tier 2)

| Path | Description |
|------|-------------|
| `ws://host/api/v1/ws/events` | Real-time event stream |

**WebSocket protocol:**

Client connects. Server pushes new events as they are ingested.

```json
// Server -> Client: New event
{
  "type": "new_event",
  "data": {
    "id": "uuid-v4",
    "title": "...",
    "event_type": "Battles",
    "latitude": 49.99,
    "longitude": 36.23,
    "event_date": "2025-02-08",
    "fatalities": 0,
    "source_name": "Reuters RSS"
  }
}

// Server -> Client: Heartbeat (every 30 seconds)
{
  "type": "heartbeat",
  "timestamp": "2025-02-08T14:32:00Z"
}

// Client -> Server: Subscribe to filters (optional)
{
  "type": "subscribe",
  "filters": {
    "countries": ["Ukraine", "Syria"],
    "event_types": ["Battles", "Explosions/Remote violence"]
  }
}
```

**Implementation pattern:** Use FastAPI's WebSocket support with a ConnectionManager class. Broadcast new events to all connected clients. Use Redis Pub/Sub as the backend to coordinate across multiple server processes. Implement heartbeat with Web Worker on the client side (per findings-08 anti-pattern guidance).

#### Endpoints Deferred to V2

| Endpoint | Why Deferred |
|----------|-------------|
| `GET /api/v1/countries/{code}` (Country profile) | Requires aggregation pipeline -- Tier 3. |
| `GET /api/v1/actors` (Actor listing/search) | Requires actor extraction -- Tier 3. |
| `GET /api/v1/alerts` (Alert configuration) | Requires baseline engine -- Tier 3. |
| `POST /api/v1/auth/*` (Authentication) | Requires auth system -- Tier 3. |
| `GET /api/v1/events/baseline` (Historical baseline) | Tier 2, built after core. |

---

### 4. End-to-End Data Flow

#### Text-Based Data Flow Diagram

```
DATA SOURCES                    INGESTION LAYER                    STORAGE LAYER
============                    ===============                    =============

+-------------+                 +---------------------+
| ACLED API   |----(daily)----->| ACLED Adapter       |
| REST/JSON   |                 | - Fetch new events  |
+-------------+                 | - Map to canonical  |
                                | - Deduplicate by    |
+-------------+                 |   ACLED event_id    |----+
| Reuters RSS |----(30min)----->| RSS Adapter         |    |
+-------------+                 | - feedparser lib    |    |
                                | - Conditional GET   |    |      +------------------+
+-------------+                 |   (ETag/Last-Mod)   |    |      |                  |
| BBC RSS     |----(30min)----->| - Extract fields    |    +----->| PostgreSQL       |
+-------------+                 | - Content hash      |    |      | + PostGIS        |
                                |   dedup             |    |      | + TimescaleDB    |
+-------------+                 | - Geocode if needed |    |      |                  |
| Al Jazeera  |----(60min)----->|                     |----+      | Tables:          |
+-------------+                 +---------------------+    |      | - events         |
                                        |                  |      | - sources        |
+-------------+                         |                  |      | - ingestion_logs |
| ICG RSS     |----(60min)------------>-+                  |      | - event_actors   |
+-------------+                         |                  |      |                  |
                                        v                  |      +--------+---------+
+-------------+                 +---------------------+    |               |
| ReliefWeb   |----(V2)------->| Celery Task Queue   |----+               |
+-------------+                 | - Redis broker      |                    |
                                | - Retry logic       |                    v
                                | - Exponential       |
                                |   backoff           |          +------------------+
                                | - Error logging     |          | Redis Cache      |
                                +---------------------+          | - Query cache    |
                                                                 | - Rate limiting  |
                                                                 | - Pub/Sub for    |
                                                                 |   WebSocket      |
                                                                 +--------+---------+
                                                                          |
                                                                          |
API LAYER                                FRONTEND
=========                                ========

+---------------------------+            +---------------------------+
| FastAPI Application       |            | Next.js App (TypeScript)  |
| (Modular Monolith)        |            |                           |
|                           |            | /dashboard                |
| GET /api/v1/events        |<---------->| - MapView (Mapbox GL JS)  |
| GET /api/v1/events/{id}   |   HTTP     | - EventFeed (data table)  |
| GET /api/v1/events/cluster|   REST     | - TimelineChart (Recharts)|
| GET /api/v1/events/timeli |            | - FilterBar               |
| GET /api/v1/events/export |            |                           |
| GET /api/v1/sources       |            | /dashboard/sources        |
| GET /api/v1/health        |            | - SourceTable             |
|                           |            | - IngestionLog            |
| WS /api/v1/ws/events      |<---------->|                           |
| (Tier 2)                  |  WebSocket | /dashboard/events/[id]    |
|                           |            | - EventDetail             |
+---------------------------+            +---------------------------+
        |                                         |
        v                                         v
+---------------------------+            +---------------------------+
| Pydantic Models           |            | Zustand (state mgmt)      |
| - EventSchema             |            | React Query (data fetch)  |
| - SourceSchema            |            | Supercluster (map cluster)|
| - TimelineSchema          |            | Tailwind CSS (styling)    |
| - ClusterSchema           |            | Dark theme default        |
+---------------------------+            +---------------------------+
```

#### Step-by-Step Data Flow Narrative

**Step 1: Ingestion (Background)**
1. Celery Beat scheduler triggers poll tasks at configured intervals (ACLED: daily, RSS feeds: 30-60 minutes).
2. Each source adapter fetches data using source-specific logic:
   - **ACLED Adapter:** Calls ACLED REST API with `?event_date_where=>{last_sync_date}`. Receives JSON array of events.
   - **RSS Adapter:** Calls feed URL with `If-None-Match: {saved_etag}`. If 304, skip. If 200, parse with feedparser.
3. Each adapter normalizes raw data into the canonical event schema (Pydantic model).
4. Deduplication check: ACLED events by `event_id_cnty`; RSS items by `sha256(feed_url + item_guid)`.
5. New events are inserted into PostgreSQL via async SQLAlchemy. PostGIS geography point is computed from lat/lng.
6. Ingestion log entry is created for each poll (source_id, timestamp, items_found, items_new, status).
7. If WebSocket is active (Tier 2): new events are published to Redis Pub/Sub channel `events:new`.

**Step 2: API Serving (On Request)**
1. Frontend sends HTTP request to FastAPI endpoint (e.g., `GET /api/v1/events?country=Ukraine&date_from=2025-02-01`).
2. FastAPI handler validates query params via Pydantic.
3. SQLAlchemy query builds with PostGIS functions for bbox filtering, tsvector for search, standard WHERE for other filters.
4. Results are serialized via Pydantic response model.
5. Redis caches the query result with a TTL of 60 seconds (for identical repeated queries).
6. JSON response returned to frontend.

**Step 3: Frontend Rendering (On Receive)**
1. React Query receives the API response and caches it.
2. Zustand store updates with the event data.
3. MapView component receives events, passes to Supercluster for clustering, renders markers on Mapbox GL JS.
4. EventFeed component renders the sortable data table.
5. TimelineChart component renders the Recharts bar chart.
6. All components share the same Zustand filter state, so filter changes trigger re-fetch and re-render of all three views simultaneously.

**Step 4: Real-Time Updates (Tier 2)**
1. Client opens WebSocket connection to `/api/v1/ws/events`.
2. FastAPI WebSocket handler subscribes to Redis Pub/Sub channel `events:new`.
3. When a new event is published, the handler pushes it to all connected clients.
4. Client-side Web Worker manages the WebSocket connection (immune to browser timer throttling).
5. New events are prepended to the event feed with highlight animation.
6. Map adds the new event marker.

---

### 5. Build Order with Dependencies

Each phase depends on the previous phase being complete.

```
PHASE 1: Foundation (Week 1)
├── 1a. Project scaffolding
│   ├── FastAPI project structure (modular monolith)
│   ├── Next.js project structure (App Router)
│   ├── Docker Compose (PostgreSQL + PostGIS + Redis)
│   ├── Alembic migration setup
│   └── CI/CD pipeline (lint + type check + test)
│
├── 1b. Canonical event schema + database
│   ├── PostgreSQL table: events (with PostGIS geography column)
│   ├── PostgreSQL table: sources
│   ├── PostgreSQL table: ingestion_logs
│   ├── Indexes: event_date, country, event_type, geography (GIST)
│   └── Pydantic models: EventCreate, EventRead, EventList
│
└── 1c. ACLED data ingestion
    ├── ACLED API client (async httpx)
    ├── ACLED-to-canonical mapper
    ├── Celery task: poll_acled (daily schedule)
    ├── Deduplication by event_id_cnty
    └── Seed database with 3-6 months of ACLED data

    DEPENDS ON: 1a, 1b complete

PHASE 2: Core API + Map (Week 2)
├── 2a. REST API endpoints
│   ├── GET /api/v1/events (with all query params)
│   ├── GET /api/v1/events/{id}
│   ├── GET /api/v1/events/clusters (server-side clustering)
│   ├── GET /api/v1/events/timeline (aggregation)
│   ├── GET /api/v1/health
│   └── Redis query caching (60s TTL)
│
├── 2b. Interactive map
│   ├── Mapbox GL JS setup (dark style)
│   ├── Supercluster integration
│   ├── Color-coded markers by event type
│   ├── Click marker to select event
│   └── Viewport-based loading (bbox queries)
│
└── 2c. Filter bar
    ├── Date range picker (with presets)
    ├── Event type multi-select
    ├── Country dropdown with search
    ├── Zustand store for filter state
    └── React Query hooks for API calls

    DEPENDS ON: Phase 1 complete

PHASE 3: Feed + Timeline + Export (Week 3)
├── 3a. Event feed / data table
│   ├── Sortable columns (date, location, type, fatalities)
│   ├── Click row to highlight on map
│   ├── Infinite scroll or pagination
│   └── Loading skeleton state
│
├── 3b. Timeline chart
│   ├── Recharts BarChart component
│   ├── Day/week/month interval toggle
│   ├── Fatality line overlay toggle
│   └── Brush selection to zoom date range
│
├── 3c. CSV export
│   ├── Backend: GET /api/v1/events/export (streaming CSV)
│   ├── Frontend: Export button triggers download
│   └── Applies current filters to export
│
└── 3d. RSS feed ingestion
    ├── RSS adapter using feedparser
    ├── Conditional requests (ETag / Last-Modified)
    ├── Exponential backoff with jitter
    ├── Content hash deduplication
    ├── Celery tasks: poll_rss_feed (per-feed schedule)
    └── 5-10 curated feeds configured as seeds

    DEPENDS ON: Phase 2 complete (need API to verify data is correct)

PHASE 4: Polish + Source Management (Week 4)
├── 4a. Event detail page
│   ├── /dashboard/events/[id] route
│   ├── Mini-map centered on event
│   ├── Full metadata display
│   └── Link to source URL
│
├── 4b. Source management page
│   ├── /dashboard/sources route
│   ├── Source table with status indicators
│   ├── Enable/disable toggle
│   ├── POST /api/v1/sources (add new feed)
│   ├── PATCH /api/v1/sources/{id}
│   └── Ingestion log display
│
├── 4c. Map-feed bidirectional sync
│   ├── Click marker -> scroll to and highlight feed row
│   ├── Click feed row -> pan map to event
│   └── Shared selection state in Zustand
│
└── 4d. Full-text search
    ├── PostgreSQL tsvector index on title + description + actors
    ├── Search box component
    └── Results appear in event feed

    DEPENDS ON: Phase 3 complete

PHASE 5: Real-Time + Baselines (Week 5-6, Tier 2)
├── 5a. WebSocket live events
│   ├── FastAPI WebSocket endpoint
│   ├── ConnectionManager class
│   ├── Redis Pub/Sub integration
│   ├── Client Web Worker for connection management
│   ├── Reconnection with exponential backoff
│   └── "X new events" badge + highlight animation
│
├── 5b. Historical baseline comparison
│   ├── Materialized view: rolling 52-week averages per country
│   ├── Dashed line overlay on timeline chart
│   └── "Above/below baseline" indicator per country
│
└── 5c. Subnational drill-down
    ├── Admin-1 boundary GeoJSON (Natural Earth data)
    ├── Click country to zoom and show province boundaries
    └── Filter events to selected province

    DEPENDS ON: Phase 4 complete
```

**Dependency graph (simplified):**

```
1a (scaffold) --> 1b (schema) --> 1c (ACLED ingest) --> 2a (API)
                                                          |
                                    2b (map) <--- 2c (filters) ---> 3a (feed)
                                       |                               |
                                       v                               v
                                    4c (map-feed sync)              3b (timeline)
                                                                       |
                                                                       v
                                                                    3c (export)
                                                                       |
                                                                       v
                                    3d (RSS ingest) -------------> 4b (source mgmt)
                                                                       |
                                                                       v
                                                                    4d (search)
                                                                       |
                                                                       v
                                    5a (WebSocket) --> 5b (baselines) --> 5c (drill-down)
```

---

## Comparison Tables

### MVP Feature Comparison: Sentinel vs Existing Platforms at Launch

| Feature | Liveuamap V1 (2014) | ACLED Dashboard V1 | Sentinel MVP |
|---------|--------------------|--------------------|-------------|
| Interactive map | Yes (Leaflet.js) | Yes (Tableau/ArcGIS) | Yes (Mapbox GL JS) |
| Event feed / table | Minimal (list below map) | Yes (Tableau table) | Yes (sortable, filterable) |
| Timeline chart | No (added later) | Yes (Tableau chart) | Yes (Recharts) |
| Filters (date, type, location) | Location only | Yes (via Tableau) | Yes (all three) |
| CSV export | No | Yes (Tableau export) | Yes |
| Real-time updates | Yes (web crawlers) | No (weekly batch) | Tier 2 (WebSocket) |
| Full-text search | No | Limited | Tier 2 (tsvector) |
| Data sources | Web crawlers (proprietary) | ACLED data only | ACLED API + 5-10 RSS feeds |
| Human verification | Yes (2+ editors per event) | Yes (trained coders) | No (source attribution only) |
| NLP pipeline | Basic (crawlers) | None (human-coded) | None (deferred to V2) |
| Mobile app | No (added 2015) | No | No (deferred) |

### Complexity vs Value Matrix

| Feature | Complexity | Analyst Value | Build? |
|---------|-----------|---------------|--------|
| Map with markers | L | Critical | Tier 1 - YES |
| Event feed table | M | Critical | Tier 1 - YES |
| Date/type/country filters | S each | Critical | Tier 1 - YES |
| Timeline chart | M | High | Tier 1 - YES |
| CSV export | S | High | Tier 1 - YES |
| ACLED ingestion | L | Critical | Tier 1 - YES |
| RSS ingestion | L | High | Tier 1 - YES |
| Full-text search | M | High | Tier 2 |
| WebSocket live updates | L | Medium-High | Tier 2 |
| Historical baselines | M | High | Tier 2 |
| Alert thresholds | L | Medium | Tier 3 |
| Country profiles | L | Medium | Tier 3 |
| NLP entity extraction | XL | Low (V1) | DEFERRED |
| Forecasting models | XL | Low (V1) | DEFERRED |
| Entity relationship graphs | XL | Low (V1) | DEFERRED |

---

## Priority Implementation Order

1. **Project scaffolding + Docker Compose + database schema** -- Must be done first. Everything depends on the database schema and project structure. Use the modular monolith pattern from day one (findings-08). Define the canonical event schema in Pydantic + SQLAlchemy. Get PostgreSQL + PostGIS running in Docker. Target: day 1-2.

2. **ACLED data ingestion pipeline** -- Seed the database with real conflict data. Without data, you cannot test or develop any frontend feature. Write the ACLED adapter, Celery task, and deduplication logic. Import 3-6 months of historical data. Target: day 2-4.

3. **Core REST API endpoints** -- Build GET /api/v1/events with all filter params, GET /api/v1/events/clusters, and GET /api/v1/events/timeline. This is the contract between backend and frontend. Target: day 3-5.

4. **Interactive map with markers** -- The single most important frontend feature. Get Mapbox GL JS rendering with Supercluster clustering and color-coded markers. Connect to the events API. Target: day 5-7.

5. **Filter bar + event feed + timeline chart** -- Complete the core dashboard triptych. All three views share Zustand filter state and React Query data hooks. Target: week 2.

6. **CSV export** -- Quick win, high value. Stream filtered events as CSV from a dedicated endpoint. Target: week 2.

7. **RSS feed ingestion** -- Add real-time data flow from 5-10 curated RSS feeds. feedparser + conditional requests + exponential backoff + dedup. Target: week 2-3.

8. **Full-text search + map-feed sync + event detail page** -- Polish the core experience. Search makes the feed usable for targeted queries. Bidirectional map-feed interaction makes the dashboard feel cohesive. Event detail page gives depth. Target: week 3-4.

9. **Source management page** -- Operational health visibility. Needed once RSS feeds are running to monitor and troubleshoot ingestion. Target: week 4.

10. **WebSocket live updates** -- Transform from periodic-refresh to live dashboard. Requires Redis Pub/Sub + ConnectionManager + client Web Worker. Target: week 5.

---

## Cost Analysis

### MVP Development Stack -- All Free

| Component | Tool | Monthly Cost |
|-----------|------|-------------|
| Map tiles | Mapbox GL JS | $0 (free up to 50K map loads/month) |
| Conflict data | ACLED API | $0 (free tier) |
| News feeds | RSS (Reuters, BBC, ICG, etc.) | $0 |
| Database | PostgreSQL + PostGIS (Docker, local) | $0 |
| Cache/Queue | Redis (Docker, local) | $0 |
| Task queue | Celery (open source) | $0 |
| Frontend framework | Next.js | $0 |
| Backend framework | FastAPI | $0 |
| Charting | Recharts | $0 |
| State management | Zustand + React Query | $0 |
| Map clustering | Supercluster | $0 |

**Total MVP development cost: $0/month** (runs entirely on Docker locally).

### First Deployment (Post-MVP)

| Component | Managed Option | Monthly Cost |
|-----------|---------------|-------------|
| PostgreSQL + PostGIS | Supabase Pro or Railway | $25-50 |
| Redis | Upstash free tier | $0 |
| Backend hosting | Railway or Fly.io | $5-20 |
| Frontend hosting | Vercel free tier | $0 |
| Mapbox | Free tier | $0 |
| Domain name | Namecheap/Cloudflare | $1 |

**Total first deployment: $31-71/month.**

### Production Scale (If Sentinel Gains Users)

| Component | Service | Monthly Cost |
|-----------|---------|-------------|
| PostgreSQL + PostGIS + TimescaleDB | AWS RDS or self-hosted | $50-200 |
| Redis | AWS ElastiCache or Upstash | $15-50 |
| Backend | AWS ECS / Fly.io | $50-200 |
| Frontend | Vercel Pro | $20 |
| Mapbox | 50K-200K loads | $0-250 |
| Elasticsearch (V2) | Elastic Cloud or self-hosted | $95-300 |
| Monitoring | Prometheus + Grafana (self-hosted) | $0 |

**Total production: $230-1,020/month** depending on traffic and data volume.

---

## Open Questions

1. **ACLED free tier API limits for event-level data** -- The myACLED free tier provides dashboard access but it is unclear whether disaggregated event-level API access is available without a paid plan or research agreement. This is a launch blocker for the MVP. **Next step:** Register for myACLED account immediately and test API access with a Python script that requests event-level data.

2. **RSS feed geocoding strategy** -- ACLED events come pre-geocoded (lat/lng). RSS feed items do not. How will Sentinel geocode RSS items for map display? Options: (a) regex location extraction from title/description + geocoding API, (b) country-level geocoding only (center of mentioned country), (c) defer RSS items to feed-only display (no map markers). **Next step:** Test option (b) first -- extract country names from feed items, plot at country centroid. Upgrade to option (a) in V2.

3. **Mapbox vs Leaflet.js decision** -- Liveuamap uses Leaflet (free, no API key). Mapbox GL JS offers vector tiles and better performance at high marker counts but requires an API key and has usage limits. At what event volume does this matter? **Next step:** Benchmark both with 10K and 50K markers. If Leaflet handles it, use Leaflet to avoid the API key dependency.

4. **Server-side vs client-side clustering** -- Supercluster runs in the browser. For 100K+ events, should clustering happen server-side (PostGIS ST_ClusterDBSCAN) to reduce payload size? **Next step:** Test client-side Supercluster performance with the full ACLED dataset (~300K events for recent years). If it is slow (>500ms), implement server-side clustering as the default.

5. **ACLED data freshness for the MVP demo** -- ACLED updates weekly (typically with a 1-2 week lag). For a demo or initial user testing, stale data may underwhelm users expecting "real-time" intelligence. Should the MVP launch be timed around a fresh ACLED data release? **Next step:** Check ACLED's release schedule and plan the first demo accordingly. Use RSS feed data to demonstrate real-time capability.

6. **How to handle RSS items that are not conflict events?** -- A BBC World News RSS feed includes sports, entertainment, and other non-conflict stories. Should the MVP include a keyword filter during ingestion (only ingest items matching conflict-related keywords) or ingest everything and let the user filter? **Next step:** Start with keyword-based ingestion filtering. Maintain a configurable keyword list (e.g., "conflict," "attack," "military," "protest," "killed," "wounded," "shelling"). Low precision is acceptable for the MVP; false negatives are worse than false positives.

---

## Sources & References

### Architecture & Design
- FastAPI WebSocket docs: https://fastapi.tiangolo.com/advanced/websockets/
- fastapi-websocket-pubsub library: https://github.com/permitio/fastapi_websocket_pubsub
- Better Stack FastAPI WebSocket guide: https://betterstack.com/community/guides/scaling-python/fastapi-websockets/
- Next.js App Router layouts and pages: https://nextjs.org/docs/app/getting-started/layouts-and-pages
- Next.js dashboard tutorial: https://nextjs.org/learn/dashboard-app/creating-layouts-and-pages
- Next.js project structure: https://nextjs.org/docs/app/getting-started/project-structure

### API Design
- REST API Design: Filtering, Sorting, Pagination (Moesif): https://www.moesif.com/blog/technical/api-design/REST-API-Design-Filtering-Sorting-and-Pagination/
- Best Practices for REST API Design (Stack Overflow): https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/
- Pragmatic RESTful API Design (Vinay Sahni): https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
- OpenAQ Geospatial API patterns: https://docs.openaq.org/using-the-api/geospatial
- UCDP API documentation: https://ucdp.uu.se/apidocs/

### Conflict Data Sources
- ACLED API documentation: https://acleddata.com/acled-api-documentation
- ACLED conflict data: https://acleddata.com/conflict-data
- Liveuamap about: https://liveuamap.com/about
- Liveuamap Wikipedia: https://en.wikipedia.org/wiki/Liveuamap

### OSINT & Dashboard Research
- ShadowDragon OSINT tools 2026: https://shadowdragon.io/blog/best-osint-tools/
- Knowlesys OSINT dashboard: https://knowlesys.com/en/osint/osint-dashboard.html
- AI-supported conflict monitoring (Traversals): https://traversals.com/blog/ai-supported-conflict-monitoring/
- OSINT for crisis management: https://osintguide.com/2025/01/15/osint-for-crisis-management/

### Prior Sentinel Research
- findings-07-what-actually-works.md -- Real platform architectures, analyst workflows, MVP features
- findings-08-anti-patterns.md -- Database anti-patterns, RSS pitfalls, architecture mistakes, Supabase gotchas
