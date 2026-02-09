# Findings: Data Source Priority — Ranked Feeds and APIs for Day-1 Implementation

## Executive Summary

This document provides a ranked, prioritized list of every data source Sentinel should connect to, grouped into Tier 1 (Day 1), Tier 2 (Week 2), and Tier 3 (Later) categories. For each Tier 1 source, exact URLs, authentication details, polling intervals, expected volume, and parsing notes are included. The highest-value, lowest-effort sources to wire up on Day 1 are: BBC World RSS feeds (4 regional feeds, zero auth, high conflict relevance), Al Jazeera RSS (single feed, zero auth, excellent Middle East coverage), GDELT DOC 2.0 API (zero auth, 15-minute updates, massive volume), and ReliefWeb API (trivial auth, continuous updates, humanitarian crisis focus). ACLED is critical but requires OAuth registration, so it slots into late Tier 1 / early Tier 2. Reuters has no working RSS feeds (discontinued 2020) and requires a third-party workaround.

## Research Scope

This document responds to Research Brief 11: "Data Source Priority — Which Feeds and APIs to Wire Up First." It covers:

1. RSS feeds ranked by value with actual working URLs
2. Conflict data APIs ranked by ease of integration and value delivered
3. Tier 1 / Tier 2 / Tier 3 groupings with justification
4. Implementation checklist per Tier 1 source (URL, type, auth, polling interval, volume, parsing notes)
5. Copy-pasteable configuration format for all Tier 1 sources

Prior findings consulted: findings-01-data-sources.md, findings-06-rss-ingestion-methods.md.

---

## Detailed Findings

### 1. RSS Feeds — Ranked by Value

#### Verified Working RSS Feed URLs (as of February 2026)

| Rank | Source | Feed URL | Status | Auth | Conflict Relevance |
|------|--------|----------|--------|------|-------------------|
| 1 | BBC World News | `https://feeds.bbci.co.uk/news/world/rss.xml` | Active | None | High |
| 2 | BBC Middle East | `https://feeds.bbci.co.uk/news/world/middle_east/rss.xml` | Active | None | Very High |
| 3 | BBC Africa | `https://feeds.bbci.co.uk/news/world/africa/rss.xml` | Active | None | Very High |
| 4 | BBC Asia | `https://feeds.bbci.co.uk/news/world/asia/rss.xml` | Active | None | High |
| 5 | Al Jazeera (All) | `https://www.aljazeera.com/xml/rss/all.xml` | Active | None | Very High |
| 6 | France 24 English | `https://www.france24.com/en/rss` | Active | None | High |
| 7 | AP World News | `https://apnews.com/world-news.rss` | Likely Active | None | High |
| 8 | Crisis Group | `https://www.crisisgroup.org/rss-0` | Active | None | Very High |
| 9 | RFE/RL (main) | See rferl.org/rssfeeds for specific feeds | Active | None | High (E. Europe/Central Asia) |
| 10 | Defence Blog | `https://defence-blog.com/feed` | Active | None | High (military) |
| 11 | Defense News | See defensenews.com/m/rss/ for topic feeds | Active | None | High (defense policy) |
| 12 | BBC Europe | `https://feeds.bbci.co.uk/news/world/europe/rss.xml` | Active | None | Medium-High |
| 13 | BBC Latin America | `https://feeds.bbci.co.uk/news/world/latin_america/rss.xml` | Active | None | Medium |
| 14 | France 24 Europe | `https://www.france24.com/en/europe/rss` | Active | None | Medium |
| 15 | France 24 Middle East | `https://www.france24.com/en/middle-east/rss` | Active | None | High |
| 16 | SIPRI News | `https://www.sipri.org/rss.xml` | Active | None | Medium (arms/military spending) |
| 17 | Homeland Security Newswire | `https://www.homelandsecuritynewswire.com/rss` | Active | None | Medium |

#### Reuters — Special Case (No Working RSS Feed)

Reuters officially discontinued all RSS feeds in June 2020. There is no official `reuters.com` RSS URL that works. The options are:

1. **Third-party RSS generators**: Services like RSS.app, FiveFilters Full-Text RSS, or Feedspot can generate RSS feeds by scraping `https://www.reuters.com/world/`. These are fragile and may break when Reuters changes their site structure.
2. **RSSHub**: The open-source project RSSHub (https://docs.rsshub.app/) can generate feeds from reuters.com pages.
3. **Self-hosted scraper**: Build a lightweight scraper that polls `reuters.com/world/` every 10 minutes, extracts headline links, and produces an internal RSS/JSON feed.

**Recommendation for Sentinel**: Skip Reuters RSS for Day 1. Add it in Tier 2 using a self-hosted RSSHub instance or a custom scraper. Reuters content will still appear via GDELT (which monitors Reuters as a source).

#### AP News — Partial RSS Support

AP News supports RSS by appending `.rss` to category URLs. The primary world news feed is:
```
https://apnews.com/world-news.rss
```

However, AP's RSS feeds have been reported as intermittently unreliable. The feed is licensed for **noncommercial use only**. For commercial use, contact AP directly.

**Recommendation**: Include in Tier 1 but implement health-checking. If the feed goes down, GDELT and BBC provide overlapping coverage.

---

### 2. Conflict Data APIs — Ranked by Ease + Value

| Rank | Source | Effort to Integrate | Value Delivered | Auth Complexity | Update Frequency |
|------|--------|-------------------|-----------------|-----------------|-----------------|
| 1 | GDELT DOC 2.0 API | Small | Very High | None | Every 15 min |
| 2 | ReliefWeb API | Small | High | Trivial (appname param) | Continuous |
| 3 | ACLED API | Medium | Very High | OAuth (registration required) | Weekly |
| 4 | GDELT GEO 2.0 API | Small | Medium | None | Every 15 min |
| 5 | UCDP API | Small | Medium | None | Annual |
| 6 | NASA FIRMS | Small | Medium | MAP_KEY (free registration) | Every 3 hours |
| 7 | GDELT BigQuery | Large | Very High | Google Cloud auth | Every 15 min |

#### GDELT DOC 2.0 API — Easiest High-Value Source

- **Endpoint**: `https://api.gdeltproject.org/api/v2/doc/doc`
- **Authentication**: None. Completely open.
- **Rate Limits**: Undocumented but expects reasonable use. Do not hammer.
- **Data Window**: Rolling 3-month archive of global news.
- **Update Cadence**: Every 15 minutes.
- **Output Formats**: JSON, HTML, CSV, RSS (via mode parameter).

**Conflict-Specific Query Examples**:
```
# Articles about armed conflict, last 24 hours, JSON output
https://api.gdeltproject.org/api/v2/doc/doc?query=armed+conflict&mode=artlist&format=json&timespan=24h

# Timeline of conflict mentions by volume
https://api.gdeltproject.org/api/v2/doc/doc?query=conflict+violence+attack&mode=timelinevol&format=json&timespan=7d

# Articles about a specific country's conflict
https://api.gdeltproject.org/api/v2/doc/doc?query=conflict+sourcecountry:UK&mode=artlist&format=json

# Using GDELT themes for conflict filtering
https://api.gdeltproject.org/api/v2/doc/doc?query=theme:KILL+OR+theme:MILITARY&mode=artlist&format=json
```

**Key Parameters**:
- `query`: Keyword search. Supports AND/OR/NOT, domain, sourcecountry, theme filters.
- `mode`: `artlist` (article list), `timelinevol` (volume timeline), `timelinevolraw` (raw counts), `timelinesourcecountry`, `timelinetone`.
- `format`: `json`, `html`, `csv`, `rss`.
- `timespan`: e.g., `24h`, `7d`, `3m`. Max is 3 months.
- `maxrecords`: Number of results (default 75, max 250).

**Python Client**: `pip install gdelt-doc-api` (GitHub: alex9smith/gdelt-doc-api)

```python
from gdeltdoc import GdeltDoc, Filters

f = Filters(
    keyword="armed conflict",
    start_date="2026-02-01",
    end_date="2026-02-09",
    country="UK"
)
gd = GdeltDoc()
articles = gd.article_search(f)  # Returns DataFrame
timeline = gd.timeline_search("timelinevol", f)
```

**Caveats**: GDELT is auto-coded from news articles. It is noisy. Deduplication and relevance filtering are essential. The DOC API only covers the last 3 months. For longer historical analysis, use BigQuery.

#### ReliefWeb API — Clean, Well-Documented Humanitarian Data

- **Endpoint**: `https://api.reliefweb.int/v1/reports`
- **Authentication**: Include `appname` query parameter with your application name (e.g., `?appname=sentinel`). This is an identifier, not a secret key. No registration required.
- **Rate Limits**: No hard published limits. Reasonable use expected.
- **Update Cadence**: Continuous. Curated by 24/7 editorial team.
- **Output Format**: JSON.

**Key Endpoints**:
```
# Latest reports (humanitarian updates)
https://api.reliefweb.int/v1/reports?appname=sentinel&limit=50&sort[]=date:desc

# Reports filtered by country (Syria)
https://api.reliefweb.int/v1/reports?appname=sentinel&filter[field]=country.name&filter[value]=Syria&limit=50

# Reports filtered by disaster type
https://api.reliefweb.int/v1/reports?appname=sentinel&filter[field]=disaster_type.name&filter[value]=Conflict&limit=50

# Active disasters
https://api.reliefweb.int/v1/disasters?appname=sentinel&filter[field]=status&filter[value]=current&limit=50

# Country profiles
https://api.reliefweb.int/v1/countries?appname=sentinel&limit=50
```

**All Endpoints**: `/v1/reports`, `/v1/disasters`, `/v1/countries`, `/v1/jobs`, `/v1/training`, `/v1/sources`, `/v1/blog`, `/v1/book`, `/v1/references`.

**Filtering**: Supports nested field filtering, date ranges, full-text search, faceting, and field selection. The API is one of the best-documented humanitarian data APIs available.

#### ACLED API — Gold Standard Conflict Data (Requires Registration)

- **Endpoint**: `https://acleddata.com/api/acled/read`
- **Authentication**: OAuth 2.0 token-based.
  1. Register at https://acleddata.com (use institutional email for better access tier).
  2. Verify email, accept Terms of Use.
  3. Generate access key in your myACLED dashboard.
  4. For programmatic access, POST to `https://acleddata.com/oauth/token` with:
     - `email=YOUR_EMAIL`
     - `password=YOUR_PASSWORD`
     - `grant_type=password`
     - `client_id=acled`
  5. Receive access token (24h validity) and refresh token (14d validity).
  6. Include `Authorization: Bearer ACCESS_TOKEN` in API requests.
- **Rate Limits**: Pagination at 5,000 rows per call. Bandwidth-limited.
- **Update Cadence**: Weekly (some regions near real-time).
- **Output Formats**: JSON, CSV (via `_format` parameter).

**Key Endpoints**:
```
# Core conflict events (JSON, limited to 500)
https://acleddata.com/api/acled/read?limit=500

# Events in a specific country
https://acleddata.com/api/acled/read?country=Syria&limit=5000&_format=json

# Events by date range
https://acleddata.com/api/acled/read?event_date=2026-01-01|2026-02-09&limit=5000

# CAST forecasting endpoint (conflict predictions)
https://acleddata.com/api/cast/read?limit=500

# Events by type (battles only)
https://acleddata.com/api/acled/read?event_type=Battles&limit=5000
```

**Filters**: country, region (numeric), event_type, event_date, interaction, fatalities, actor1, actor2, admin1, admin2, admin3, source, notes.

**Python Access**:
```python
import requests

# Step 1: Get OAuth token
token_resp = requests.post("https://acleddata.com/oauth/token", data={
    "email": "your@email.com",
    "password": "yourpassword",
    "grant_type": "password",
    "client_id": "acled"
})
access_token = token_resp.json()["access_token"]

# Step 2: Query events
headers = {"Authorization": f"Bearer {access_token}"}
resp = requests.get(
    "https://acleddata.com/api/acled/read",
    params={"country": "Ukraine", "limit": 5000, "_format": "json"},
    headers=headers
)
events = resp.json()
```

**R Package**: `acled.api` on CRAN (maintained, updated July 2025).

#### UCDP API — Academic Baseline (Easy but Infrequent)

- **Endpoint**: `https://ucdpapi.pcr.uu.se/api/gedevents/25.1`
- **Authentication**: None.
- **Rate Limits**: Pagination via `pagesize` parameter (default 20, recommended 100).
- **Update Cadence**: Annual (version 25.1 covers through 2024).
- **Output Format**: JSON.

**Key Endpoints**:
```
# Georeferenced events (GED) - latest version
https://ucdpapi.pcr.uu.se/api/gedevents/25.1?pagesize=100

# Filter by country (using UCDP country IDs)
https://ucdpapi.pcr.uu.se/api/gedevents/25.1?pagesize=100&Country=365

# Armed conflicts
https://ucdpapi.pcr.uu.se/api/ucdpprioconflict/25.1?pagesize=100

# Non-state conflicts
https://ucdpapi.pcr.uu.se/api/nonstate/25.1?pagesize=100

# One-sided violence
https://ucdpapi.pcr.uu.se/api/onesided/25.1?pagesize=100
```

#### NASA FIRMS API — Satellite Fire Proxy for Hostilities

- **Endpoint**: `https://firms.modaps.eosdis.nasa.gov/api/area/`
- **Authentication**: MAP_KEY required. Free registration at https://firms.modaps.eosdis.nasa.gov/api/
- **Rate Limits**: 5,000 transactions per 10 minutes per MAP_KEY.
- **Update Cadence**: NRT data within 3 hours globally.
- **Output Formats**: CSV, JSON, KML, SHP.

**Example Query** (VIIRS data, South America, last 1 day):
```
https://firms.modaps.eosdis.nasa.gov/api/area/csv/YOUR_MAP_KEY/VIIRS_NOAA20_NRT/-85,-57,-32,14/1
```

---

### 3. Source Reliability Baseline

| Source | Uptime | Data Cleanliness | Update Frequency | Conflict Signal-to-Noise |
|--------|--------|------------------|-----------------|--------------------------|
| BBC RSS | Very High (99.9%+) | Clean RSS 2.0 | Every few minutes | Medium (general news) |
| Al Jazeera RSS | High (99%+) | Clean RSS 2.0 | Every few minutes | High (conflict focus) |
| AP RSS | Medium (intermittent) | Standard RSS | Every few minutes | Medium (general news) |
| France 24 RSS | High (99%+) | Clean RSS 2.0 | Every few minutes | Medium-High |
| Crisis Group RSS | High | Clean RSS/Atom | Daily-weekly | Very High (pure conflict analysis) |
| GDELT API | High (99%+) | Noisy (auto-coded) | Every 15 min | Medium (requires filtering) |
| ReliefWeb API | Very High (UN-hosted) | Very Clean (curated) | Continuous | High (humanitarian crises) |
| ACLED API | High | Very Clean (human-coded) | Weekly | Very High (pure conflict events) |
| UCDP API | High | Very Clean (human-coded) | Annual | Very High (but stale) |
| NASA FIRMS | Very High (NASA) | Clean (satellite data) | Every 3 hours | Low (fire proxy, needs context) |
| RFE/RL RSS | Medium (funding issues since 2025) | Clean RSS | Daily | High (E. Europe/C. Asia conflicts) |
| Defence Blog RSS | Medium | Standard RSS | Daily | High (military developments) |

**Best uptime**: BBC, GDELT, ReliefWeb, NASA FIRMS (all backed by major institutions).
**Cleanest data**: ACLED and UCDP (human-coded). ReliefWeb (professionally curated).
**Highest conflict relevance**: ACLED, Crisis Group, Al Jazeera, UCDP.
**Fastest updates**: GDELT (15 min), BBC/Al Jazeera RSS (minutes), ReliefWeb (continuous).

---

### 4. Tier Groupings

#### TIER 1 — Day 1 (Wire Up Immediately)

These sources are free, require zero or trivial authentication, and provide immediate conflict intelligence value.

| # | Source | Type | Why Day 1 |
|---|--------|------|-----------|
| 1 | BBC World RSS | RSS 2.0 | Zero auth, reliable, global conflict coverage |
| 2 | BBC Middle East RSS | RSS 2.0 | Zero auth, highest-conflict region |
| 3 | BBC Africa RSS | RSS 2.0 | Zero auth, major conflict region |
| 4 | BBC Asia RSS | RSS 2.0 | Zero auth, major conflict region |
| 5 | Al Jazeera RSS | RSS 2.0 | Zero auth, excellent conflict coverage |
| 6 | France 24 English RSS | RSS 2.0 | Zero auth, French-perspective conflict coverage |
| 7 | GDELT DOC 2.0 API | REST API | Zero auth, 15-min updates, massive volume |
| 8 | ReliefWeb API | REST API | Trivial auth (appname), curated humanitarian data |
| 9 | Crisis Group RSS | RSS/Atom | Zero auth, pure conflict analysis |
| 10 | AP World News RSS | RSS 2.0 | Zero auth (noncommercial), breaking news |

#### TIER 2 — Week 2 (Requires Registration or Extra Parsing)

| # | Source | Type | Why Week 2 |
|---|--------|------|-----------|
| 11 | ACLED API | REST API (OAuth) | Requires registration + OAuth flow, but essential |
| 12 | RFE/RL RSS feeds | RSS 2.0 | Need to identify specific feed URLs from their catalog |
| 13 | Defence Blog RSS | RSS 2.0 | Secondary source, military focus |
| 14 | Defense News RSS | RSS 2.0 | Need to identify correct feed URL from their menu |
| 15 | BBC Europe RSS | RSS 2.0 | Lower priority region for conflict |
| 16 | France 24 Middle East RSS | RSS 2.0 | Overlaps with Al Jazeera |
| 17 | GDELT GEO 2.0 API | REST API | Geospatial layer on top of GDELT |
| 18 | NASA FIRMS API | REST API | Requires MAP_KEY registration |
| 19 | Reuters (via RSSHub/scraper) | Custom | No native RSS, requires workaround |

#### TIER 3 — Later (Supplementary or High-Effort)

| # | Source | Type | Why Later |
|---|--------|------|----------|
| 20 | UCDP API | REST API | Annual updates only, historical baseline |
| 21 | SIPRI RSS | RSS 2.0 | Annual data, arms/spending context |
| 22 | GDELT BigQuery | BigQuery SQL | Requires Google Cloud setup, complex queries |
| 23 | Telegram channels | Bot API/TDLib | Complex integration, moderation needed |
| 24 | Twitter/X API | REST API | $100+/month, noisy, API restrictions |
| 25 | Liveuamap API | REST API | Paid, terms unclear |
| 26 | Homeland Security Newswire | RSS | Niche, US-focused |

---

### 5. Implementation Checklist — Tier 1 Sources

#### Source 1: BBC World News RSS
- **URL**: `https://feeds.bbci.co.uk/news/world/rss.xml`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 30-60 new items/day
- **Parsing Notes**: Standard RSS 2.0. Fields: `title`, `description`, `link`, `pubDate`, `guid`. Description is a short summary (1-2 sentences). No full article text in feed. `guid` is the article URL. Use `feedparser` or `fastfeedparser`. Supports ETag/Last-Modified for conditional requests.

#### Source 2: BBC Middle East RSS
- **URL**: `https://feeds.bbci.co.uk/news/world/middle_east/rss.xml`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 10-25 new items/day
- **Parsing Notes**: Same format as BBC World. Regional subset.

#### Source 3: BBC Africa RSS
- **URL**: `https://feeds.bbci.co.uk/news/world/africa/rss.xml`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 10-20 new items/day
- **Parsing Notes**: Same format as BBC World. Regional subset.

#### Source 4: BBC Asia RSS
- **URL**: `https://feeds.bbci.co.uk/news/world/asia/rss.xml`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 10-25 new items/day
- **Parsing Notes**: Same format as BBC World. Regional subset.

#### Source 5: Al Jazeera (All News) RSS
- **URL**: `https://www.aljazeera.com/xml/rss/all.xml`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 40-80 new items/day
- **Parsing Notes**: Single feed covering all topics. Higher conflict signal-to-noise than BBC due to editorial focus. Fields: `title`, `description`, `link`, `pubDate`, `guid`, `category`. Description includes a longer summary. May include media elements. Use `feedparser`.

#### Source 6: France 24 English RSS
- **URL**: `https://www.france24.com/en/rss`
- **Type**: RSS 2.0
- **Auth**: None
- **Polling Interval**: Every 10 minutes
- **Expected Volume**: 30-50 new items/day
- **Parsing Notes**: Standard RSS 2.0. Provides French-perspective coverage of global conflicts. Good for European/African conflict coverage. France 24 also offers region-specific feeds at `/en/{region}/rss`.

#### Source 7: GDELT DOC 2.0 API
- **URL**: `https://api.gdeltproject.org/api/v2/doc/doc`
- **Type**: REST API (GET)
- **Auth**: None
- **Polling Interval**: Every 15 minutes (aligned with GDELT update cadence)
- **Expected Volume**: 100-500 articles per query (depends on filters)
- **Parsing Notes**: Returns JSON array of article objects with `url`, `title`, `seendate`, `socialimage`, `domain`, `language`, `sourcecountry`. Use conflict-specific queries: `query=conflict+OR+attack+OR+military+OR+violence&mode=artlist&format=json&timespan=15min&maxrecords=250`. Requires aggressive deduplication — GDELT returns many duplicates from syndicated articles. No full text in API response, only metadata and URLs.

#### Source 8: ReliefWeb API
- **URL**: `https://api.reliefweb.int/v1/reports`
- **Type**: REST API (GET/POST)
- **Auth**: `appname` query parameter (e.g., `?appname=sentinel`)
- **Polling Interval**: Every 30 minutes
- **Expected Volume**: 50-150 new reports/day
- **Parsing Notes**: Returns JSON with rich metadata: `title`, `body` (full HTML text), `date.created`, `country`, `source`, `theme`, `disaster_type`, `format`. Filter for conflict: `filter[field]=disaster_type.name&filter[value]=Conflict`. Supports pagination via `offset` and `limit`. Body field contains full article HTML — run through HTML sanitizer.

#### Source 9: Crisis Group RSS
- **URL**: `https://www.crisisgroup.org/rss-0`
- **Type**: RSS/Atom
- **Auth**: None
- **Polling Interval**: Every 60 minutes (publishes infrequently but high value)
- **Expected Volume**: 2-5 new items/day
- **Parsing Notes**: Pure conflict analysis content. Every item is highly relevant. Fields include detailed analysis briefs, CrisisWatch updates, and policy recommendations. Lower volume but extremely high signal-to-noise ratio.

#### Source 10: AP World News RSS
- **URL**: `https://apnews.com/world-news.rss`
- **Type**: RSS 2.0
- **Auth**: None (noncommercial use only)
- **Polling Interval**: Every 5 minutes
- **Expected Volume**: 30-50 new items/day
- **Parsing Notes**: Standard RSS 2.0 format. May be intermittently unreliable — implement health monitoring and automatic fallback. AP is a primary wire service so many stories will also appear via GDELT. Licensed for noncommercial use; commercial deployment requires AP licensing agreement.

---

### 6. Copy-Pasteable Configuration for Tier 1 Sources

```python
# Sentinel Tier 1 Source Configuration
# Copy-pasteable into source config module

TIER_1_RSS_FEEDS = [
    {
        "id": "bbc-world",
        "name": "BBC World News",
        "url": "https://feeds.bbci.co.uk/news/world/rss.xml",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,  # 5 minutes
        "priority": "high",
        "tier": 1,
        "categories": ["world", "conflict", "politics"],
        "expected_daily_volume": 50,
        "parser": "fastfeedparser",  # well-formed, use fast parser
        "use_conditional_requests": True,  # ETag / Last-Modified
        "max_consecutive_errors": 5,
    },
    {
        "id": "bbc-middle-east",
        "name": "BBC Middle East",
        "url": "https://feeds.bbci.co.uk/news/world/middle_east/rss.xml",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,
        "priority": "high",
        "tier": 1,
        "categories": ["middle-east", "conflict"],
        "expected_daily_volume": 20,
        "parser": "fastfeedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "bbc-africa",
        "name": "BBC Africa",
        "url": "https://feeds.bbci.co.uk/news/world/africa/rss.xml",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,
        "priority": "high",
        "tier": 1,
        "categories": ["africa", "conflict"],
        "expected_daily_volume": 15,
        "parser": "fastfeedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "bbc-asia",
        "name": "BBC Asia",
        "url": "https://feeds.bbci.co.uk/news/world/asia/rss.xml",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,
        "priority": "high",
        "tier": 1,
        "categories": ["asia", "conflict"],
        "expected_daily_volume": 20,
        "parser": "fastfeedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "aljazeera-all",
        "name": "Al Jazeera (All News)",
        "url": "https://www.aljazeera.com/xml/rss/all.xml",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,
        "priority": "high",
        "tier": 1,
        "categories": ["world", "middle-east", "conflict"],
        "expected_daily_volume": 60,
        "parser": "feedparser",  # use robust parser, less certain about feed quality
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "france24-en",
        "name": "France 24 English",
        "url": "https://www.france24.com/en/rss",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 600,  # 10 minutes
        "priority": "medium",
        "tier": 1,
        "categories": ["world", "europe", "africa", "conflict"],
        "expected_daily_volume": 40,
        "parser": "feedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "crisis-group",
        "name": "International Crisis Group",
        "url": "https://www.crisisgroup.org/rss-0",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 3600,  # 1 hour (low volume, high value)
        "priority": "high",
        "tier": 1,
        "categories": ["conflict-analysis", "policy"],
        "expected_daily_volume": 3,
        "parser": "feedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 5,
    },
    {
        "id": "ap-world",
        "name": "AP World News",
        "url": "https://apnews.com/world-news.rss",
        "type": "rss",
        "auth": None,
        "poll_interval_seconds": 300,
        "priority": "high",
        "tier": 1,
        "categories": ["world", "breaking-news"],
        "expected_daily_volume": 40,
        "parser": "feedparser",
        "use_conditional_requests": True,
        "max_consecutive_errors": 3,  # lower threshold — known intermittent issues
    },
]

TIER_1_APIS = [
    {
        "id": "gdelt-doc",
        "name": "GDELT DOC 2.0 API",
        "base_url": "https://api.gdeltproject.org/api/v2/doc/doc",
        "type": "rest_api",
        "auth": None,
        "poll_interval_seconds": 900,  # 15 minutes (aligned with GDELT updates)
        "priority": "high",
        "tier": 1,
        "default_params": {
            "query": "conflict OR attack OR military OR violence OR bombing OR airstrike",
            "mode": "artlist",
            "format": "json",
            "timespan": "15min",
            "maxrecords": 250,
        },
        "expected_daily_volume": 500,
        "requires_dedup": True,  # CRITICAL: GDELT returns many duplicates
        "max_consecutive_errors": 5,
    },
    {
        "id": "reliefweb-reports",
        "name": "ReliefWeb Reports API",
        "base_url": "https://api.reliefweb.int/v1/reports",
        "type": "rest_api",
        "auth": {"type": "query_param", "key": "appname", "value": "sentinel"},
        "poll_interval_seconds": 1800,  # 30 minutes
        "priority": "high",
        "tier": 1,
        "default_params": {
            "appname": "sentinel",
            "limit": 50,
            "sort[]": "date:desc",
            "filter[field]": "disaster_type.name",
            "filter[value]": "Conflict",
        },
        "expected_daily_volume": 100,
        "max_consecutive_errors": 5,
    },
]

# ACLED Configuration (Tier 1.5 — requires OAuth setup first)
ACLED_CONFIG = {
    "id": "acled-events",
    "name": "ACLED Conflict Events",
    "base_url": "https://acleddata.com/api/acled/read",
    "auth_url": "https://acleddata.com/oauth/token",
    "type": "rest_api_oauth",
    "auth": {
        "type": "oauth2",
        "token_endpoint": "https://acleddata.com/oauth/token",
        "grant_type": "password",
        "client_id": "acled",
        # email and password from environment variables:
        # ACLED_EMAIL, ACLED_PASSWORD
        "token_expiry_hours": 24,
        "refresh_token_expiry_days": 14,
    },
    "poll_interval_seconds": 86400,  # Daily check (data updates weekly)
    "priority": "critical",
    "tier": 1,  # Critical source but needs OAuth setup
    "default_params": {
        "limit": 5000,
        "_format": "json",
    },
    "expected_weekly_volume": 5000,
    "pagination": {"type": "offset", "page_size": 5000},
    "max_consecutive_errors": 3,
}
```

---

## Comparison Tables

### RSS Feeds: Value vs Effort Matrix

| Feed | Conflict Relevance (1-5) | Reliability (1-5) | Volume/Day | Auth Effort | Overall Priority |
|------|--------------------------|-------------------|-----------|-------------|-----------------|
| BBC World | 3 | 5 | 50 | Zero | Tier 1 |
| BBC Middle East | 5 | 5 | 20 | Zero | Tier 1 |
| BBC Africa | 5 | 5 | 15 | Zero | Tier 1 |
| Al Jazeera | 5 | 4 | 60 | Zero | Tier 1 |
| AP World | 3 | 3 | 40 | Zero | Tier 1 |
| France 24 | 3 | 4 | 40 | Zero | Tier 1 |
| Crisis Group | 5 | 4 | 3 | Zero | Tier 1 |
| RFE/RL | 4 | 3 | 15 | Zero | Tier 2 |
| Defence Blog | 4 | 3 | 5 | Zero | Tier 2 |
| Defense News | 3 | 4 | 10 | Zero | Tier 2 |
| Reuters | 3 | N/A | N/A | High (scraper) | Tier 2 |
| SIPRI | 2 | 4 | <1 | Zero | Tier 3 |

### APIs: Integration Effort vs Value

| API | Value (1-5) | Integration Effort | Auth Complexity | Update Speed | Day-1 Ready? |
|-----|-------------|-------------------|-----------------|-------------|-------------|
| GDELT DOC 2.0 | 5 | Small (1-2 hrs) | None | 15 min | YES |
| ReliefWeb | 4 | Small (1-2 hrs) | Trivial | Continuous | YES |
| ACLED | 5 | Medium (4-8 hrs) | OAuth registration | Weekly | After registration |
| UCDP | 3 | Small (1-2 hrs) | None | Annual | YES but low priority |
| NASA FIRMS | 3 | Small (2-3 hrs) | MAP_KEY registration | 3 hours | After registration |
| GDELT BigQuery | 5 | Large (1-2 days) | Google Cloud auth | 15 min | NO |

---

## Priority Implementation Order

1. **BBC Regional RSS Feeds (4 feeds)** — Wire these up first. Zero configuration beyond the URL. Immediate conflict coverage across Middle East, Africa, Asia, and global. These feeds are the most reliable RSS feeds on the internet. Takes 30 minutes to implement with feedparser.

2. **Al Jazeera RSS** — Add immediately after BBC. Single feed URL, zero auth, excellent conflict signal. Particularly strong for Middle East, North Africa, and Muslim-world conflicts where BBC coverage may be lighter.

3. **GDELT DOC 2.0 API** — The single highest-volume free conflict data source. Zero auth. Build a collector that queries every 15 minutes with conflict-specific keywords. Requires deduplication pipeline (use SHA-256 fingerprints + URL normalization from findings-06).

4. **ReliefWeb API** — Add `?appname=sentinel` to requests and filter for disaster_type=Conflict. Provides curated humanitarian/crisis reporting that complements news feeds with operational context.

5. **Crisis Group + France 24 + AP RSS** — Three more zero-auth feeds. Crisis Group is low volume but highest signal. France 24 adds European/Francophone perspective. AP adds wire service breadth.

6. **ACLED API** — Register for myACLED account (takes 1-3 business days for verification). Implement OAuth token flow. This is the most important structured conflict dataset and should be live by end of Week 1. Set up daily polling with weekly full refresh.

7. **Tier 2 RSS Feeds (RFE/RL, Defence Blog, Defense News)** — Add secondary RSS feeds. Each takes 15 minutes to add once the RSS pipeline exists.

8. **NASA FIRMS + Reuters workaround** — Register for FIRMS MAP_KEY. Set up RSSHub or custom scraper for Reuters. Both require external registration/setup.

9. **UCDP API + SIPRI** — Add as historical baseline sources. Annual updates mean these are reference data, not real-time feeds.

10. **GDELT BigQuery + Social Media** — Advanced integrations requiring Google Cloud setup (BigQuery) or paid API access (Twitter/X).

---

## Cost Analysis

### Tier 1 Sources (All Free)

| Source | Cost | Notes |
|--------|------|-------|
| BBC RSS (4 feeds) | $0 | Free, no restrictions on polling |
| Al Jazeera RSS | $0 | Free |
| France 24 RSS | $0 | Free |
| AP World RSS | $0 | Noncommercial use only |
| Crisis Group RSS | $0 | Free |
| GDELT DOC 2.0 API | $0 | Free, open access |
| ReliefWeb API | $0 | Free, UN-hosted |
| **Total Tier 1** | **$0** | |

### Tier 2 Sources

| Source | Cost | Notes |
|--------|------|-------|
| ACLED API | $0 (research) | Free for research/academic. Commercial licensing separate. |
| RFE/RL RSS | $0 | Free |
| Defence Blog RSS | $0 | Free |
| NASA FIRMS | $0 | Free with MAP_KEY registration |
| Reuters (via RSSHub) | $0 (self-hosted) | RSSHub is open-source; hosting cost only |
| **Total Tier 2** | **$0** | (assuming research use for ACLED) |

### Tier 3 Sources (Potentially Paid)

| Source | Cost | Notes |
|--------|------|-------|
| GDELT BigQuery | $0-50/month | Google Cloud free tier covers moderate use |
| Twitter/X Basic | $100/month | 10,000 tweets/month read |
| Liveuamap API | TBD | Paid, pricing not public |
| ACLED Commercial | Contact sales | Required for commercial deployment |
| **Total Tier 3** | **$100-200+/month** | |

### Infrastructure for All Tiers

| Component | Self-Hosted | Managed |
|-----------|------------|---------|
| Redis (broker + dedup) | $0 | $25-50/month |
| PostgreSQL + PostGIS | $0 | $30-60/month |
| Single VPS (4GB RAM) | $20/month | N/A |
| **Total infrastructure** | **$20/month** | **$55-110/month** |

---

## Open Questions

1. **AP News RSS reliability** — The `https://apnews.com/world-news.rss` URL is based on documented pattern but may be unreliable. Needs live testing over 7 days to confirm uptime. If it fails, AP content is still captured via GDELT. Suggested next step: curl the feed daily for a week and log HTTP status codes.

2. **ACLED commercial licensing terms** — If Sentinel is deployed as a commercial product, ACLED's free research access will not apply. Commercial licensing terms and pricing are not publicly available. Suggested next step: Contact access@acleddata.com to inquire about commercial API access pricing.

3. **RFE/RL operational status** — In March 2025, USAGM terminated RFE/RL funding grants. RFE/RL sued to block this. As of early 2026, their website and RSS feeds appear active, but long-term operational viability is uncertain. Suggested next step: Monitor rferl.org uptime and feed freshness weekly.

4. **Al Jazeera topic-specific RSS feeds** — Al Jazeera only officially offers the single `all.xml` feed. Topic-specific feeds (e.g., Middle East only, Africa only) would require third-party RSS generation from section pages. Suggested next step: Test if `aljazeera.com/middle-east/rss.xml` or similar patterns work natively.

5. **GDELT DOC API rate limits** — GDELT does not publish formal rate limits. Polling every 15 minutes with a single query should be safe, but adding multiple parallel queries (by region, by keyword) could trigger throttling. Suggested next step: Start with a single query and gradually increase, monitoring for HTTP 429 responses.

6. **Feed format variations across sources** — Not all feeds use the same RSS version or field naming. Some may use Atom, some RSS 2.0, some may include georss:point or media:content extensions. Suggested next step: Fetch each Tier 1 feed once and document the exact XML structure and available fields.

7. **Reuters alternative via GDELT** — Since Reuters has no RSS feed, one approach is to query GDELT DOC API with `domain:reuters.com` to capture Reuters articles. This would provide Reuters content with a 15-minute delay. Suggested next step: Test `https://api.gdeltproject.org/api/v2/doc/doc?query=domain:reuters.com&mode=artlist&format=json&timespan=24h` and evaluate coverage.

---

## Sources & References

### RSS Feed Sources
- BBC RSS Feeds: https://feeds.bbci.co.uk/news/world/rss.xml
- Al Jazeera RSS: https://www.aljazeera.com/xml/rss/all.xml
- AP News RSS pattern: https://apnews.com/world-news.rss
- France 24 RSS Feeds: https://www.france24.com/en/rss-feeds
- Crisis Group RSS: https://www.crisisgroup.org/rss-0
- RFE/RL RSS: https://www.rferl.org/rssfeeds
- Defense News RSS: https://www.defensenews.com/m/rss/
- Defence Blog RSS: https://defence-blog.com/feed
- SIPRI RSS: https://www.sipri.org/rss.xml
- Feedspot World News RSS List: https://rss.feedspot.com/world_news_rss_feeds/
- Reuters RSS discontinuation (FiveFilters): https://www.fivefilters.org/2021/reuters-rss-feeds/

### API Documentation
- ACLED API Docs: https://acleddata.com/acled-api-documentation
- ACLED Getting Started: https://acleddata.com/api-documentation/getting-started
- ACLED Access Guide: https://acleddata.com/methodology/acled-access-guide
- ACLED ACLED Endpoint: https://acleddata.com/api-documentation/acled-endpoint
- ACLED myACLED FAQs: https://acleddata.com/myacled-faqs
- GDELT DOC 2.0 API: https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/
- GDELT Context 2.0 API: https://blog.gdeltproject.org/announcing-the-gdelt-context-2-0-api/
- GDELT Python Client: https://github.com/alex9smith/gdelt-doc-api
- GDELT Complex Queries (BigQuery): https://blog.gdeltproject.org/complex-queries-combining-events-eventmentions-and-gkg/
- ReliefWeb API Docs: https://apidoc.reliefweb.int/
- UCDP API Docs: https://ucdp.uu.se/apidocs/
- NASA FIRMS API: https://firms.modaps.eosdis.nasa.gov/api/

### Tools and Libraries
- feedparser: https://github.com/kurtmckee/feedparser
- fastfeedparser: https://github.com/kagisearch/fastfeedparser
- gdelt-doc-api (Python): https://pypi.org/project/gdelt-doc-api/
- acled.api (R package): https://cran.r-project.org/web/packages/acled.api/
- RSSHub (self-hosted RSS generator): https://docs.rsshub.app/
- RSS.app (third-party RSS generator): https://rss.app/
