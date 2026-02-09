# Findings 01: Data Sources & Ingestion Pipeline

**Research Date:** 2026-02-09
**Status:** Complete
**Brief:** research/brief-01-data-sources.md

---

## Table of Contents

1. [Conflict Data APIs & Datasets](#1-conflict-data-apis--datasets)
2. [RSS Feeds & News Sources](#2-rss-feeds--news-sources)
3. [Social Media & OSINT Sources](#3-social-media--osint-sources)
4. [Geospatial & Satellite Data](#4-geospatial--satellite-data)
5. [Supplementary Databases](#5-supplementary-databases)
6. [Recommended Ingestion Architecture](#6-recommended-ingestion-architecture)
7. [Normalized Event Schema](#7-normalized-event-schema)
8. [Cost Analysis Summary](#8-cost-analysis-summary)
9. [Source Ranking & Recommendations](#9-source-ranking--recommendations)

---

## 1. Conflict Data APIs & Datasets

### 1.1 ACLED (Armed Conflict Location & Event Data)

| Field | Details |
|-------|---------|
| **URL** | https://acleddata.com |
| **API Docs** | https://acleddata.com/acled-api-documentation |
| **API Base URL** | `https://acleddata.com/api/acled/read` |
| **Authentication** | myACLED account required; cookie-based or OAuth token-based auth |
| **Data Format** | CSV, JSON (via `_format` parameter) |
| **Update Frequency** | Weekly (near real-time for select regions) |
| **Coverage** | Global political violence, protest, and strategic development events since 1997 |
| **Cost** | Free for academic/research use; commercial licensing available on request |
| **Rate Limits** | Bandwidth-limited; pagination recommended at 5,000 rows per call |

**Key Endpoints:**
- **ACLED Endpoint** (`/api/acled/read`): Core event dataset -- political violence, demonstrations, strategic developments
- **CAST Endpoint** (`/api/cast/read`): Conflict Alert System predictions -- forecasts of political violence events for rolling 4-week periods across every country
- **Deleted Endpoint**: Track removed/corrected records

**Example API Call:**
```
https://acleddata.com/api/acled/read?_format=csv&country=Georgia&limit=5000
```

**Query Filters:** country, region (numeric codes), event_type, event_date, interaction, fatalities, and more. Region names in API use numeric codes (not string names).

**Data Modes:** Dyadic (default, matches ACLED Codebook with one event per row showing actor interaction) and Monadic.

**Pros:**
- Gold-standard conflict event dataset, widely cited in academic research
- Global coverage with consistent coding methodology
- Built-in forecasting via CAST endpoint
- Well-documented API with pagination support
- Active community and R package (`acled.api`)

**Cons:**
- Weekly update cadence (not truly real-time)
- Rate limits require careful pagination for bulk downloads
- Commercial use requires separate licensing
- OAuth token flow adds complexity vs. simple API key

---

### 1.2 GDELT (Global Database of Events, Language, and Tone)

| Field | Details |
|-------|---------|
| **URL** | https://www.gdeltproject.org |
| **API Docs** | https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/ |
| **API Base URL** | `https://api.gdeltproject.org/api/v2/` |
| **Authentication** | None required (open access) |
| **Data Format** | JSON, JSONP, CSV (raw files), BigQuery |
| **Update Frequency** | Every 15 minutes |
| **Coverage** | Global news monitoring across 65 languages, back to 1979 |
| **Cost** | 100% free and open |
| **Rate Limits** | Undocumented; reasonable use expected |

**Key Endpoints:**

1. **DOC 2.0 API** (`/api/v2/doc/doc`): Full-text search across rolling 3-month window of global news coverage. Supports keyword, domain, country, language, and theme filters.

2. **GEO 2.0 API** (`/api/v2/geo/geo`): Geographic mapping of keyword mentions. Returns locations mentioned near keywords across 65 languages over last 7 days. Updated every 15 minutes.

3. **TV 2.0 API** (`/api/v2/tv/tv`): Television news search covering 9+ years of US and international TV news.

4. **Context 2.0 API** (`/api/v2/context/context`): Sentence-level search for granular analysis.

5. **Raw Data Files**: Available on S3/HTTP for bulk download, processable at scale via Google BigQuery.

**Query Parameters (common):** keyword, domain, domain_exact, country (FIPS codes), language (ISO 639), theme (GKG themes), timespan (min/h/d/w format).

**Timeline Modes:** timelinevol (volume %), timelinevolraw (raw counts), timelinesourcecountry (by publisher country), timelinetone (average sentiment).

**Python Libraries:**
- `gdelt-doc-api` (PyPI): Clean client for DOC 2.0 API
- `gdeltPyR` (PyPI): Full Python framework returning Pandas DataFrames

**Pros:**
- Massive scale: monitors virtually all global news in 65 languages
- 15-minute update cadence (near real-time)
- No authentication required
- Free BigQuery integration for unlimited-scale analysis
- Rich NLP features: tone analysis, themes, geographic extraction
- Active and maintained through 2026

**Cons:**
- News-derived data (not curated event data like ACLED) -- noisy, requires filtering
- DOC API only searches rolling 3-month window
- No formal SLA or guaranteed uptime
- Can be overwhelming in volume; deduplication needed
- Event coding is automated (machine-generated), not human-verified

---

### 1.3 UCDP (Uppsala Conflict Data Program)

| Field | Details |
|-------|---------|
| **URL** | https://ucdp.uu.se |
| **API Docs** | https://ucdp.uu.se/apidocs/ |
| **API Base URL** | `https://ucdpapi.pcr.uu.se/api/<resource>/<version>` |
| **Authentication** | None required |
| **Data Format** | JSON (API), CSV, Excel, R, STATA (downloads) |
| **Update Frequency** | Annually (version 25.1 is latest, covering through 2024; GED updated more frequently) |
| **Coverage** | Global organized violence 1946-2024 |
| **Cost** | Free |
| **Rate Limits** | Pagination via `pagesize` parameter |

**Key Datasets (version 25.1):**
- UCDP/PRIO Armed Conflict Dataset
- UCDP Dyadic Dataset
- UCDP Non-State Conflict Dataset
- UCDP One-Sided Violence Dataset
- UCDP Battle-Related Deaths Dataset
- UCDP Georeferenced Event Dataset (GED) -- granular event-level data with coordinates
- UCDP Actor Dataset

**Example API Calls:**
```
https://ucdpapi.pcr.uu.se/api/nonstate/25.1?pagesize=100&Country=90,91,92
https://ucdpapi.pcr.uu.se/api/ucdpprioconflict/25.1?pagesize=100&Country=365,369&year=2014,2015
```

**GitHub Resources:** https://github.com/UppsalaConflictDataProgram (includes `basic_api_recipes` Jupyter Notebooks)

**Pros:**
- Longest-running conflict dataset (since 1946); gold standard for academic research
- GED provides georeferenced event data with precise coordinates
- Clean REST API with versioned endpoints
- Multiple data formats supported
- No authentication required
- Rigorous human-coded data with transparent methodology

**Cons:**
- Annual update cycle means significant lag for current events
- Not suitable for real-time monitoring
- Focused on organized violence (does not cover protests, strategic developments like ACLED)
- Smaller scope than ACLED for sub-annual analysis

---

### 1.4 UN OCHA ReliefWeb API

| Field | Details |
|-------|---------|
| **URL** | https://reliefweb.int |
| **API Docs** | https://apidoc.reliefweb.int/ |
| **API Base URL** | `https://api.reliefweb.int/v1/` |
| **Authentication** | `appname` parameter (identifier, not auth key) |
| **Data Format** | JSON |
| **Update Frequency** | Continuous (curated by 24/7 editorial team) |
| **Coverage** | Global humanitarian crises, disasters, conflicts -- archive back to 1980s |
| **Cost** | Free |
| **Rate Limits** | Reasonable use; no hard published limits |

**Endpoints (9 content types):**
- `/v1/reports` -- Main content: updates and analysis from 4,000+ sources
- `/v1/disasters` -- Disaster metadata grouping reports by event
- `/v1/countries` -- Geographic metadata
- `/v1/jobs` -- Humanitarian job listings
- `/v1/training` -- Training opportunities
- `/v1/sources` -- Source organizations
- `/v1/blog`, `/v1/book`, `/v1/references` -- Additional content types

**Request Types:**
- **List requests** (no ID): Support extensive filtering, field selection, sorting, and faceting
- **Item requests** (with ID): Detailed information on a specific piece of content

**Related OCHA Services:**
- **Humanitarian Data Exchange (HDX)**: https://data.humdata.org -- Real-time humanitarian datasets
- **OCHA Taxonomy as a Service**: https://vocabulary.unocha.org -- Standardized vocabularies
- **OCHA-DAP GitHub**: https://github.com/OCHA-DAP -- Open-source tools

**Pros:**
- Curated by professional editorial team (24/7)
- Deep historical archive (1980s to present)
- Standardized taxonomy across humanitarian vocabulary
- Publishing API available for content partners
- Well-documented REST API
- Humanitarian-focused context complements pure conflict data

**Cons:**
- Focus is humanitarian response rather than conflict events per se
- Reports are document-level (not structured event data)
- Requires NLP/extraction to derive structured conflict events
- Not designed for real-time event alerting

---

## 2. RSS Feeds & News Sources

### 2.1 Major News Wire Services

| Source | RSS Feed URL | Update Frequency | Focus |
|--------|-------------|-----------------|-------|
| **BBC World** | `https://feeds.bbci.co.uk/news/world/rss.xml` | Minutes | Global news, conflicts, politics |
| **BBC Asia** | `https://feeds.bbci.co.uk/news/world/asia/rss.xml` | Minutes | Asian conflicts & politics |
| **BBC Middle East** | `https://feeds.bbci.co.uk/news/world/middle_east/rss.xml` | Minutes | Middle East conflicts |
| **BBC Africa** | `https://feeds.bbci.co.uk/news/world/africa/rss.xml` | Minutes | African conflicts & politics |
| **AP World** | `https://hosted.ap.org/lineups/WORLDHEADS.rss` | Minutes | Global breaking news |
| **Reuters World** | Available via Feedspot RSS builder or direct site | Minutes | Geopolitics, conflicts |
| **Al Jazeera** | `https://www.aljazeera.com/xml/rss/all.xml` | Minutes | Middle East & global conflicts |
| **Defence Blog** | `https://defence-blog.com/feed` | Hours | Military/defense developments |
| **Radio Free Europe** | `https://www.rferl.org/api/` | Minutes | Eastern Europe, Central Asia |

### 2.2 Specialized Security & Conflict RSS Feeds

| Source | URL/Feed | Focus |
|--------|----------|-------|
| **International Crisis Group** | `https://www.crisisgroup.org/latest-updates/rss` | Conflict analysis & prevention |
| **SIPRI News** | `https://www.sipri.org/rss.xml` | Arms, military spending |
| **NATO News** | `https://www.nato.int/cps/en/natohq/news.htm` (RSS available) | Alliance activities |
| **UN Security Council** | `https://www.un.org/securitycouncil/` (RSS available) | Resolutions, meetings |
| **DOD News** | `https://www.defense.gov/News/` (RSS available) | US military operations |
| **Jane's/Janes** | Paid subscription required | Defense intelligence |

### 2.3 RSS Feed Integration Architecture

**Polling Strategy:**
- High-priority feeds (BBC, AP, Reuters): Poll every 2-5 minutes
- Medium-priority feeds (specialized conflict): Poll every 15-30 minutes
- Low-priority feeds (analysis/think tanks): Poll every 1-2 hours

**Key Considerations:**
- Respect `If-Modified-Since` and `ETag` headers to minimize unnecessary traffic
- Implement exponential backoff for failed requests
- Parse both RSS 2.0 and Atom feed formats
- Extract and normalize: title, description, published date, link, categories/tags
- Use feed-specific parsers for custom fields (e.g., georss:point)

**Recommended Libraries:**
- Python: `feedparser`, `aiohttp` (async polling)
- Node.js: `rss-parser`, `node-fetch`

---

## 3. Social Media & OSINT Sources

### 3.1 Liveuamap

| Field | Details |
|-------|---------|
| **URL** | https://liveuamap.com |
| **API** | Paid API at https://liveuamap.com/promo/api |
| **Cost** | $5/month for Pro; API pricing separate |
| **Data Format** | JSON (API), interactive web maps |
| **Coverage** | Active conflict zones: Ukraine, Syria, Yemen, Israel-Gaza, and more |

**Key Features:**
- Aggregates data from news outlets, social media, and direct reports
- Interactive maps with time-based filtering
- Chronological archive of events
- Free browser access; paid API for programmatic use

**Open-Source Scraping:**
- GitHub: `conflict-investigations/liveuamap-analysis` -- Python scraper for territory control data, outputs JSON/CSV
- Community norm: if using scraped data for publications, support Liveuamap with a subscription

**Pros:** Real-time conflict mapping, intuitive visualization, broad conflict coverage
**Cons:** Paid API, coverage may be uneven for less-publicized conflicts, ethical considerations around scraping

### 3.2 Twitter/X API

| Field | Details |
|-------|---------|
| **API** | X API v2 |
| **Cost** | Basic: $100/month (10,000 tweets/month read); Pro: $5,000/month (1M tweets/month) |
| **Relevance** | Primary source of first-hand conflict reports, OSINT community discussions |

**Key Considerations:**
- High cost for meaningful volume
- API access increasingly restricted since 2023
- Content quality highly variable; requires NLP filtering
- Critical for breaking events before news coverage
- Monitor specific accounts: war correspondents, OSINT analysts, official military accounts

**Pros:** Fastest source for breaking events, first-person accounts
**Cons:** Expensive, noisy, misinformation risk, API instability

### 3.3 Telegram Channel Monitoring

| Field | Details |
|-------|---------|
| **API** | Telegram Bot API / TDLib |
| **Cost** | Free (API access) |
| **Relevance** | Critical for Ukraine/Russia, Middle East conflict zones |

**Tools:**
- `Telegago` -- OSINT tool for Telegram channel analysis and sentiment tracking
- `Telethon` (Python) -- Full Telegram client library for channel monitoring
- TDLib -- Official Telegram Database Library

**Pros:** Free API access, critical source for conflict zone communications
**Cons:** Language barriers, misinformation, ethical/legal considerations, channels may be ephemeral

### 3.4 Other OSINT Platforms & Tools

| Tool | Purpose | URL |
|------|---------|-----|
| **Bellingcat Investigation Toolkit** | Comprehensive OSINT methodology | https://bellingcat.gitbook.io/toolkit |
| **OSINT Framework** | Directory of free OSINT tools | https://osintframework.com |
| **Oryx** | Military equipment tracking & losses | https://www.oryxspioenkop.com |
| **Center for Information Resilience** | Verified OSINT evidence | https://www.info-res.org |
| **Maltego** | Link analysis across OSINT sources | https://www.maltego.com |
| **Intelligence X** | Deep/dark web monitoring | https://intelx.io |
| **Hunchly** | Web capture & evidence preservation | https://hunch.ly |

---

## 4. Geospatial & Satellite Data

### 4.1 NASA FIRMS (Fire Information for Resource Management System)

| Field | Details |
|-------|---------|
| **URL** | https://firms.modaps.eosdis.nasa.gov |
| **API Docs** | https://firms.modaps.eosdis.nasa.gov/api/ |
| **API Base URL** | `https://firms.modaps.eosdis.nasa.gov/api/area/` |
| **Authentication** | MAP_KEY required (free registration) |
| **Data Format** | CSV, JSON, KML, SHP |
| **Update Frequency** | NRT data within 3 hours; Ultra Real-Time (URT) within 60 seconds for US/Canada |
| **Coverage** | Global fire detections from MODIS and VIIRS satellites |
| **Cost** | Free |
| **Rate Limits** | 5,000 transactions per 10-minute interval per MAP_KEY |

**Key Endpoints:**
- **Area API** (`/api/area/csv/[MAP_KEY]/[SOURCE]/[AREA]/[DAY_RANGE]`): Fire detections within bounding box or worldwide
- **Data Availability** (`/api/data_availability/`): Check current data status
- **KML Fire Footprints** (`/api/kml_fire_footprints/`): Fire outlines in KML format
- **Archive Download** (`/download/`): Historical fire data

**Data Sources:** MODIS (1km resolution) and VIIRS (375m resolution) from NOAA-20 and Suomi-NPP satellites.

**Conflict Monitoring Use:** Fire detection data serves as a proxy for artillery strikes, bombings, and infrastructure destruction in conflict zones. Anomalous fire clusters in known conflict areas can indicate active hostilities.

**Example API Call:**
```
https://firms.modaps.eosdis.nasa.gov/api/area/csv/YOUR_MAP_KEY/VIIRS_NOAA20_NRT/-85,-57,-32,14/1
```

**Pros:**
- Free, high-quality satellite data
- Near real-time updates (3 hours globally, <60 seconds for US/Canada)
- Global coverage
- Well-documented API with multiple output formats
- Python tutorials and Fire Data Academy available

**Cons:**
- Fire data, not conflict data -- requires contextual interpretation
- Cloud cover can block detections
- False positives from agricultural burning, wildfires
- Resolution limitations (375m-1km) for precise targeting analysis

### 4.2 Additional Geospatial Sources

| Source | Access | Cost | Resolution | Relevance |
|--------|--------|------|------------|-----------|
| **Sentinel Hub** (Copernicus) | API | Free (limited) / Paid | 10m | Change detection, infrastructure damage |
| **Planet Labs** | API | Paid ($$$) | 3-5m daily | High-frequency monitoring |
| **Maxar** | API | Paid ($$$$) | 30cm | Detailed damage assessment |
| **OpenStreetMap** | Overpass API | Free | N/A (vector) | Baseline infrastructure mapping |
| **HOT (Humanitarian OSM Team)** | Tasking Manager API | Free | N/A (vector) | Crisis response mapping |

---

## 5. Supplementary Databases

### 5.1 SIPRI (Stockholm International Peace Research Institute)

| Field | Details |
|-------|---------|
| **URL** | https://www.sipri.org/databases |
| **API** | No official API; unofficial Python wrapper on GitHub |
| **Data Format** | Excel, CSV (via unofficial tools), HTML |
| **Update Frequency** | Annual |
| **Cost** | Free |

**Databases:**
1. **Arms Transfers Database** (https://armstransfers.sipri.org): Major conventional arms transfers 1950-2024. Updated March 2025.
2. **Military Expenditure Database** (https://milex.sipri.org): Military spending 1949-2024 in multiple currencies and formats.
3. **Arms Industry Database**: Top 100 arms-producing and military services companies.

**Programmatic Access:**
- Unofficial Python wrapper: `github.com/benryan58/sipri_arms` -- supports CSV, JSON, HTML, RTF output as Pandas DataFrames
- World Bank API: SIPRI data available via World Bank open data portal (well-documented API)

**Pros:** Authoritative data on arms flows and military spending, long time series
**Cons:** No official API, annual updates, supplementary rather than core conflict data

### 5.2 START Global Terrorism Database (GTD)

| Field | Details |
|-------|---------|
| **URL** | https://www.start.umd.edu/gtd/ |
| **API** | No public API; bulk download |
| **Data Format** | CSV |
| **Update Frequency** | Annual |
| **Coverage** | Terrorist attacks worldwide, 1970-2020 (as of last public release) |
| **Cost** | Free for research |

**Note:** GTD's public data release has been irregular. Verify current availability before depending on this source.

### 5.3 Crisis24 / Risk Intelligence APIs

| Field | Details |
|-------|---------|
| **URL** | https://crisis24.garda.com |
| **API** | Commercial API |
| **Cost** | Enterprise pricing ($$$$) |
| **Coverage** | Global risk alerts, travel security, threat intelligence |

**Pros:** Professional-grade, real-time alerts
**Cons:** Very expensive, designed for corporate security teams

---

## 6. Recommended Ingestion Architecture

### 6.1 High-Level Architecture

```
+-------------------+     +-------------------+     +-------------------+
|   DATA SOURCES    |     |   INGESTION LAYER |     |   PROCESSING      |
|                   |     |                   |     |                   |
| RSS Feeds ------->|---->| Kafka Topic:      |---->| Flink/Kafka       |
| ACLED API ------->|---->|   raw-events      |---->| Streams:          |
| GDELT API ------->|---->|                   |---->|  - Normalize      |
| UCDP API -------->|---->| Kafka Topic:      |---->|  - Deduplicate    |
| ReliefWeb API --->|---->|   raw-documents   |---->|  - Geocode        |
| NASA FIRMS ------>|---->|                   |---->|  - Classify       |
| Twitter/X ------->|---->| Kafka Topic:      |---->|  - Enrich         |
| Telegram -------->|---->|   raw-social      |---->|  - Score          |
| Liveuamap ------->|---->|                   |     |                   |
+-------------------+     +-------------------+     +-------------------+
                                                            |
                                                            v
                          +-------------------+     +-------------------+
                          |   STORAGE LAYER   |     |   SERVING LAYER   |
                          |                   |     |                   |
                          | PostgreSQL/PostGIS |<--->| REST API          |
                          | Elasticsearch     |<--->| GraphQL API       |
                          | S3/MinIO (raw)    |<--->| WebSocket (live)  |
                          | TimescaleDB       |     | Dashboard         |
                          +-------------------+     +-------------------+
```

### 6.2 Component Details

#### Collector Services (Python microservices)

| Collector | Schedule | Source | Output Topic |
|-----------|----------|--------|-------------|
| `rss-collector` | Every 2-5 min | RSS feeds (BBC, AP, Reuters, etc.) | `raw-documents` |
| `acled-collector` | Weekly (cron) | ACLED API | `raw-events` |
| `gdelt-collector` | Every 15 min | GDELT DOC 2.0 API | `raw-documents` |
| `ucdp-collector` | Weekly/Monthly | UCDP API | `raw-events` |
| `reliefweb-collector` | Every 30 min | ReliefWeb API | `raw-documents` |
| `firms-collector` | Every 3 hours | NASA FIRMS API | `raw-geospatial` |
| `social-collector` | Continuous | Twitter/X, Telegram | `raw-social` |
| `liveuamap-collector` | Every 10 min | Liveuamap API | `raw-events` |

#### Message Queue: Apache Kafka

- **Topics:** `raw-events`, `raw-documents`, `raw-social`, `raw-geospatial`, `normalized-events`, `enriched-events`, `alerts`
- **Partitioning:** By geographic region (for ordered processing within regions)
- **Retention:** 30 days for raw topics, 90 days for processed topics
- **Schema Registry:** Apache Avro schemas with Confluent Schema Registry

#### Stream Processing: Apache Flink

**Processing Pipeline Stages:**

1. **Normalization**: Convert source-specific formats into unified `ConflictEvent` schema
2. **Deduplication**: Sliding-window dedup using event fingerprints (location + time + type + actors)
3. **Geocoding & Geo-enrichment**: Standardize locations using GeoNames/OSM, add country/admin boundaries
4. **Classification**: ML-based event type classification for unstructured sources (news, social)
5. **Entity Extraction**: NER for actor names, organizations, weapons systems
6. **Confidence Scoring**: Multi-source corroboration scoring (events confirmed by 3+ sources score higher)
7. **Alert Generation**: Rules-based and ML-based alert triggers for significant events

#### Storage Layer

| Store | Purpose | Data |
|-------|---------|------|
| **PostgreSQL + PostGIS** | Primary structured storage | Normalized events with geospatial indexing |
| **Elasticsearch** | Full-text search & analytics | Documents, news articles, social posts |
| **TimescaleDB** | Time-series analytics | Event counts, trends, conflict intensity metrics |
| **S3/MinIO** | Raw data lake | All raw ingested data (immutable archive) |
| **Redis** | Caching & dedup state | Recent event fingerprints, API response cache |

### 6.3 Deduplication Strategy

Cross-source deduplication is critical since the same real-world event may appear in ACLED, GDELT, news RSS, and social media.

**Approach: Multi-level dedup**

1. **Exact dedup** (within source): Hash of source-specific ID fields
2. **Near-dedup** (cross-source): Composite fingerprint using:
   - Geohash (precision 6, ~1.2km)
   - Time window (same 24-hour period)
   - Event type cluster
   - Actor name similarity (Jaccard > 0.6)
3. **Semantic dedup**: Embedding-based similarity for news articles (cosine similarity > 0.85)
4. **Entity resolution**: Link events referring to the same actors/organizations across sources

### 6.4 Rate Limiting & Respectful Collection

| Source | Rate Limit | Strategy |
|--------|-----------|----------|
| ACLED | 5,000 rows/page | Paginate with cursor; weekly batch |
| GDELT | Reasonable use | 15-min polling aligned with update cadence |
| UCDP | Not published | Conservative polling; cache aggressively |
| ReliefWeb | Reasonable use | Use `appname` param; cache responses |
| NASA FIRMS | 5,000 tx/10 min | Batch by region; respect MAP_KEY limits |
| RSS Feeds | Varies | Respect `ttl`, `If-Modified-Since`, `ETag` |
| Twitter/X | Tier-dependent | Strict rate limit compliance per tier |

---

## 7. Normalized Event Schema

### 7.1 Core `ConflictEvent` Schema

```json
{
  "event_id": "uuid-v4",
  "source_id": "acled-12345",
  "source_name": "ACLED",
  "ingestion_timestamp": "2026-02-09T14:30:00Z",

  "event_date": "2026-02-08",
  "event_timestamp": "2026-02-08T15:30:00Z",
  "precision_date": "day",

  "event_type": "BATTLE",
  "event_subtype": "Armed clash",
  "cameo_code": null,

  "location": {
    "latitude": 48.5734,
    "longitude": 37.9912,
    "geohash": "u8vf5g",
    "precision": "village",
    "location_name": "Bakhmut",
    "admin1": "Donetsk Oblast",
    "admin2": null,
    "country": "Ukraine",
    "country_iso3": "UKR",
    "region": "Eastern Europe"
  },

  "actors": [
    {
      "name": "Military Forces of Ukraine",
      "type": "state_force",
      "country": "UKR"
    },
    {
      "name": "Military Forces of Russia",
      "type": "state_force",
      "country": "RUS"
    }
  ],

  "fatalities": {
    "total": 12,
    "precision": "estimated",
    "civilians": null,
    "combatants": null
  },

  "description": "Armed clash between Ukrainian and Russian forces near Bakhmut...",
  "tags": ["artillery", "frontline", "urban_combat"],

  "sources": [
    {
      "source_name": "ACLED",
      "source_url": "https://acleddata.com/...",
      "source_event_id": "ACL-2026-12345"
    },
    {
      "source_name": "BBC News",
      "source_url": "https://bbc.co.uk/news/...",
      "source_event_id": null
    }
  ],

  "confidence": {
    "score": 0.92,
    "corroboration_count": 3,
    "verified": true
  },

  "metadata": {
    "processing_pipeline_version": "1.0.0",
    "normalized_at": "2026-02-09T14:31:00Z",
    "dedup_cluster_id": "cluster-789"
  }
}
```

### 7.2 Event Type Taxonomy (Unified)

Mapping across ACLED, GDELT, and UCDP coding schemes:

| Unified Type | ACLED Code | GDELT CAMEO | UCDP Type |
|-------------|------------|-------------|-----------|
| `BATTLE` | Battles | 190-195 | State-based |
| `EXPLOSION_REMOTE` | Explosions/Remote violence | 180-186 | State-based |
| `VIOLENCE_AGAINST_CIVILIANS` | Violence against civilians | 175-186 | One-sided |
| `PROTEST` | Protests | 140-145 | N/A |
| `RIOT` | Riots | 145-155 | N/A |
| `STRATEGIC_DEVELOPMENT` | Strategic developments | 010-170 | N/A |
| `NON_STATE_CONFLICT` | N/A | 190-195 | Non-state |

### 7.3 Database Schema (PostgreSQL + PostGIS)

```sql
CREATE TABLE conflict_events (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id           TEXT NOT NULL,
    source_name         TEXT NOT NULL,
    ingestion_ts        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    event_date          DATE NOT NULL,
    event_timestamp     TIMESTAMPTZ,
    precision_date      TEXT CHECK (precision_date IN ('exact', 'day', 'week', 'month')),

    event_type          TEXT NOT NULL,
    event_subtype       TEXT,

    location            GEOGRAPHY(POINT, 4326),
    location_name       TEXT,
    admin1              TEXT,
    country_iso3        CHAR(3),
    region              TEXT,

    fatalities_total    INTEGER,
    fatalities_precision TEXT,

    description         TEXT,
    tags                TEXT[],

    confidence_score    REAL,
    corroboration_count INTEGER DEFAULT 1,
    dedup_cluster_id    TEXT,

    raw_data            JSONB,

    UNIQUE(source_name, source_id)
);

CREATE INDEX idx_events_date ON conflict_events (event_date DESC);
CREATE INDEX idx_events_type ON conflict_events (event_type);
CREATE INDEX idx_events_country ON conflict_events (country_iso3);
CREATE INDEX idx_events_location ON conflict_events USING GIST (location);
CREATE INDEX idx_events_cluster ON conflict_events (dedup_cluster_id);
CREATE INDEX idx_events_tags ON conflict_events USING GIN (tags);

CREATE TABLE event_actors (
    id                  SERIAL PRIMARY KEY,
    event_id            UUID REFERENCES conflict_events(event_id),
    actor_name          TEXT NOT NULL,
    actor_type          TEXT,
    actor_country       CHAR(3)
);

CREATE TABLE event_sources (
    id                  SERIAL PRIMARY KEY,
    event_id            UUID REFERENCES conflict_events(event_id),
    source_name         TEXT NOT NULL,
    source_url          TEXT,
    source_event_id     TEXT,
    retrieved_at        TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 8. Cost Analysis Summary

### Free Sources (Core Pipeline)

| Source | Cost | Value |
|--------|------|-------|
| ACLED | Free (research) | High -- gold-standard event data |
| GDELT | Free | High -- near real-time, massive scale |
| UCDP | Free | Medium -- annual, academic reference |
| ReliefWeb | Free | Medium -- humanitarian context |
| NASA FIRMS | Free | Medium -- satellite fire proxy |
| RSS Feeds | Free | High -- real-time news |
| Telegram | Free | Medium -- conflict zone comms |
| SIPRI | Free | Low -- annual supplementary |

### Paid Sources (Optional Enhancements)

| Source | Cost | Value |
|--------|------|-------|
| Twitter/X Basic | $100/month | Medium -- breaking events |
| Twitter/X Pro | $5,000/month | High -- volume analysis |
| Liveuamap API | TBD (paid) | Medium -- curated conflict maps |
| Liveuamap Pro | $5/month | Low -- ad-free browser access |
| Planet Labs | $$$ (enterprise) | High -- daily satellite imagery |
| Maxar | $$$$ (enterprise) | High -- high-res damage assessment |
| Crisis24 | $$$$ (enterprise) | High -- professional risk intelligence |
| Janes | $$$$ (enterprise) | High -- defense intelligence |

### Recommended Budget Tiers

- **Tier 1 (Free):** ACLED + GDELT + UCDP + ReliefWeb + NASA FIRMS + RSS feeds -- covers 80% of needs
- **Tier 2 ($200/month):** Tier 1 + Twitter/X Basic + Liveuamap Pro
- **Tier 3 ($5,500/month):** Tier 2 + Twitter/X Pro + Liveuamap API
- **Tier 4 (Enterprise):** Tier 3 + satellite imagery + commercial risk intelligence

---

## 9. Source Ranking & Recommendations

### Priority Ranking (for conflict forecasting model)

| Rank | Source | Priority | Rationale |
|------|--------|----------|-----------|
| 1 | **ACLED** | Critical | Gold-standard structured conflict events with forecasting (CAST) |
| 2 | **GDELT** | Critical | Near real-time global news monitoring at scale, free |
| 3 | **RSS Feeds** (BBC, AP, Reuters, Al Jazeera) | Critical | Real-time breaking news with minimal latency |
| 4 | **NASA FIRMS** | High | Satellite-based proxy for active hostilities |
| 5 | **ReliefWeb** | High | Humanitarian context and crisis reports |
| 6 | **UCDP** | High | Academic reference data, historical baselines |
| 7 | **Telegram** | Medium | Conflict-zone primary source intelligence |
| 8 | **Liveuamap** | Medium | Curated conflict mapping (if budget allows API) |
| 9 | **Twitter/X** | Medium | Breaking events (if budget allows) |
| 10 | **SIPRI** | Low | Background context on arms/spending trends |

### Implementation Phases

**Phase 1 (MVP -- Weeks 1-4):**
- Implement ACLED collector (weekly batch)
- Implement GDELT DOC 2.0 collector (15-min polling)
- Implement RSS feed collector for top 10 feeds (5-min polling)
- Set up Kafka with basic topics
- Implement basic normalization to `ConflictEvent` schema
- Store in PostgreSQL + PostGIS

**Phase 2 (Enhanced -- Weeks 5-8):**
- Add NASA FIRMS collector
- Add ReliefWeb collector
- Add UCDP collector
- Implement cross-source deduplication
- Add Elasticsearch for full-text search
- Build confidence scoring based on corroboration

**Phase 3 (Full Pipeline -- Weeks 9-12):**
- Add social media collectors (Telegram, optionally Twitter/X)
- Implement Flink-based stream processing
- Add ML-based event classification for unstructured sources
- Add entity extraction and resolution
- Implement alerting system
- Add Liveuamap integration (if budget allows)

**Phase 4 (Advanced -- Weeks 13+):**
- Satellite imagery integration
- Semantic deduplication with embeddings
- Historical backfill from UCDP and ACLED archives
- Forecasting model integration with ACLED CAST
- Dashboard and API serving layer

---

## References & Key URLs

| Resource | URL |
|----------|-----|
| ACLED API Documentation | https://acleddata.com/acled-api-documentation |
| ACLED CAST Endpoint | https://acleddata.com/api-documentation/cast-endpoint |
| GDELT Project Data | https://www.gdeltproject.org/data.html |
| GDELT DOC 2.0 API | https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/ |
| GDELT GEO 2.0 API | https://blog.gdeltproject.org/gdelt-geo-2-0-api-debuts/ |
| GDELT Context 2.0 API | https://blog.gdeltproject.org/announcing-the-gdelt-context-2-0-api/ |
| UCDP Download Center | https://ucdp.uu.se/downloads/ |
| UCDP API Docs | https://ucdp.uu.se/apidocs/ |
| ReliefWeb API | https://apidoc.reliefweb.int/ |
| ReliefWeb Endpoints | https://apidoc.reliefweb.int/endpoints |
| NASA FIRMS API | https://firms.modaps.eosdis.nasa.gov/api/ |
| NASA FIRMS Area API | https://firms.modaps.eosdis.nasa.gov/api/area/ |
| SIPRI Databases | https://www.sipri.org/databases |
| SIPRI Arms Transfers | https://armstransfers.sipri.org |
| SIPRI Military Expenditure | https://milex.sipri.org |
| Liveuamap | https://liveuamap.com |
| Liveuamap API | https://liveuamap.com/promo/api |
| OCHA Humanitarian Data Exchange | https://data.humdata.org |
| OSINT Framework | https://osintframework.com |
| Bellingcat Toolkit | https://bellingcat.gitbook.io/toolkit |
| Feedspot World News RSS | https://rss.feedspot.com/world_news_rss_feeds/ |
| UCDP GitHub | https://github.com/UppsalaConflictDataProgram |
| OCHA-DAP GitHub | https://github.com/OCHA-DAP |
| SIPRI Unofficial Python Wrapper | https://github.com/benryan58/sipri_arms |
| GDELT Python Client | https://github.com/alex9smith/gdelt-doc-api |
