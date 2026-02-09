# Build Plan 14: Data Ingestion Pipeline — Coding Agent Instructions

## Executive Summary

Step-by-step build plan for Sentinel's RSS/API data ingestion pipeline. This pipeline polls RSS feeds and conflict APIs, parses entries, deduplicates, normalizes to a canonical event schema, and stores in PostgreSQL. Based on findings-06 (proven ingestion methods), findings-11 (ranked sources with URLs), and findings-12 (implementation roadmap).

## Research Scope

Based on: findings-06-rss-ingestion-methods.md, findings-11-source-priority.md, findings-12-implementation-roadmap.md

---

## 1. Directory Structure

All ingestion code lives under `src/backend/app/ingestion/`:

```
app/ingestion/
├── __init__.py
├── celery_app.py          # Celery app config + task autodiscovery
├── scheduler.py           # Celery Beat dynamic schedule (adaptive intervals)
├── feed_registry.py       # Source CRUD + polling queue management
├── feed_poller.py         # Celery task: fetch feeds with ETag/Last-Modified
├── feed_parser.py         # Parse RSS/Atom → normalized entries
├── normalizer.py          # Map parsed entries to canonical Event schema
├── dedup.py               # SHA-256 + URL normalization + MinHash dedup
├── event_storage.py       # Batch insert events to PostgreSQL
├── acled_client.py        # ACLED API integration
├── gdelt_client.py        # GDELT DOC 2.0 API integration
└── seed_sources.py        # Load Tier 1 sources into database
```

---

## 2. Pipeline Architecture

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────┐    ┌──────────┐
│ Celery Beat  │───▶│ Feed Poller  │───▶│ Feed Parser  │───▶│  Dedup  │───▶│  Storage │
│ (scheduler)  │    │ (HTTP fetch) │    │ (feedparser) │    │(SHA+URL)│    │(Postgres)│
└─────────────┘    └──────────────┘    └──────────────┘    └─────────┘    └──────────┘
       │                                                                        │
       │            ┌──────────────┐                                            │
       └───────────▶│ ACLED Client │────────────────────────────────────────────┘
                    └──────────────┘
                    ┌──────────────┐
       └───────────▶│ GDELT Client │────────────────────────────────────────────┘
                    └──────────────┘
```

---

## 3. Component Specifications

### 3.1 Celery App Configuration

```python
# app/ingestion/celery_app.py
from celery import Celery
from app.core.config import settings

celery = Celery(
    "sentinel",
    broker=settings.celery_broker_url,
    backend=settings.celery_result_backend,
)

celery.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
    task_routes={
        "app.ingestion.feed_poller.*": {"queue": "polling"},
        "app.ingestion.acled_client.*": {"queue": "api"},
        "app.ingestion.gdelt_client.*": {"queue": "api"},
    },
)

celery.autodiscover_tasks(["app.ingestion"])
```

### 3.2 Feed Poller — Celery Task

```python
# app/ingestion/feed_poller.py

@celery.task(bind=True, max_retries=3, default_retry_delay=60)
def poll_feed(self, source_id: str) -> dict:
    """
    Fetch an RSS/Atom feed with conditional GET (ETag/Last-Modified).

    Steps:
    1. Load source from DB (get URL, ETag, Last-Modified)
    2. Make HTTP GET with If-None-Match / If-Modified-Since headers
    3. If 304 Not Modified → update last_poll_at, return early
    4. If 200 → pass response body to feed_parser
    5. Store new ETag/Last-Modified from response headers
    6. Update source poll metadata (last_poll_at, status, error_count)
    7. On error → increment consecutive_errors, apply exponential backoff

    Returns: {"source_id": str, "new_items": int, "status": str}
    """

@celery.task
def poll_all_due_feeds() -> dict:
    """
    Master scheduler task. Runs every minute via Celery Beat.

    Steps:
    1. Query sources WHERE next_poll_at <= NOW() AND enabled = TRUE
    2. For each source, dispatch poll_feed.delay(source_id)
    3. Return {"dispatched": count}
    """
```

**HTTP fetch implementation details:**
```python
import httpx

async def fetch_feed(url: str, etag: str | None, last_modified: str | None) -> tuple[int, str, dict]:
    headers = {}
    if etag:
        headers["If-None-Match"] = etag
    if last_modified:
        headers["If-Modified-Since"] = last_modified

    async with httpx.AsyncClient(timeout=30.0, follow_redirects=True) as client:
        response = await client.get(url, headers=headers)

    return (
        response.status_code,
        response.text,
        {
            "etag": response.headers.get("etag"),
            "last_modified": response.headers.get("last-modified"),
        }
    )
```

### 3.3 Feed Parser

```python
# app/ingestion/feed_parser.py
import feedparser
from datetime import datetime

def parse_feed(raw_content: str, source_name: str) -> list[dict]:
    """
    Parse RSS/Atom feed content into normalized entry dicts.

    Args:
        raw_content: Raw XML string from HTTP response
        source_name: Name of the source for attribution

    Returns: List of dicts with keys:
        - title: str
        - description: str (HTML stripped)
        - url: str (entry link)
        - published: datetime
        - authors: list[str]
        - categories: list[str]
        - source_name: str

    Handles:
        - RSS 2.0, RSS 1.0, Atom feeds
        - Missing date fields (fallback to NOW)
        - Encoding issues (feedparser handles most)
        - Malformed XML (feedparser is lenient)
    """
    feed = feedparser.parse(raw_content)
    entries = []
    for entry in feed.entries:
        entries.append({
            "title": entry.get("title", "Untitled"),
            "description": strip_html(entry.get("summary", entry.get("description", ""))),
            "url": entry.get("link", ""),
            "published": parse_date(entry.get("published_parsed", entry.get("updated_parsed"))),
            "authors": [a.get("name", "") for a in entry.get("authors", [])],
            "categories": [t.get("term", "") for t in entry.get("tags", [])],
            "source_name": source_name,
        })
    return entries

def strip_html(html: str) -> str:
    """Remove HTML tags from description. Use simple regex, not lxml."""
    import re
    return re.sub(r'<[^>]+>', '', html).strip()

def parse_date(time_struct) -> datetime:
    """Convert feedparser time struct to datetime. Fallback to now()."""
    if time_struct:
        from time import mktime
        return datetime.fromtimestamp(mktime(time_struct))
    return datetime.utcnow()
```

### 3.4 Normalizer — Map to Canonical Event Schema

```python
# app/ingestion/normalizer.py

def normalize_rss_entry(entry: dict, source_id: str, source_type: str = "rss") -> dict:
    """
    Map a parsed RSS entry to the canonical Event schema.

    Input: parsed entry dict from feed_parser
    Output: dict matching EventCreate schema, ready for DB insert

    Fields mapped:
        title        ← entry["title"]
        description  ← entry["description"]
        event_date   ← entry["published"].date()
        source_name  ← entry["source_name"]
        source_url   ← entry["url"]
        source_type  ← "rss"
        raw_data     ← original entry as JSONB
        content_hash ← SHA-256 of (title + url)

    Fields NOT set (require NLP enrichment in Phase 5):
        event_type, latitude, longitude, country, region, actors, fatalities
        These remain NULL for RSS events until NLP pipeline processes them.
    """

def normalize_acled_event(acled_row: dict) -> dict:
    """
    Map an ACLED API response row to the canonical Event schema.

    ACLED provides structured data — all fields map directly:
        title        ← acled_row["notes"][:200]
        description  ← acled_row["notes"]
        event_type   ← acled_row["event_type"]
        event_date   ← acled_row["event_date"]
        latitude     ← float(acled_row["latitude"])
        longitude    ← float(acled_row["longitude"])
        country      ← acled_row["country"]
        region       ← acled_row["admin1"]
        actors       ← [acled_row["actor1"], acled_row["actor2"]]
        fatalities   ← int(acled_row["fatalities"])
        source_name  ← "ACLED"
        source_type  ← "acled"
        confidence_score ← 0.95 (ACLED is human-coded)
    """

def normalize_gdelt_article(article: dict) -> dict:
    """
    Map a GDELT DOC 2.0 API article to the canonical Event schema.

    GDELT provides less structure — partial mapping:
        title        ← article["title"]
        description  ← article["seendate"] context
        source_url   ← article["url"]
        source_name  ← article["domain"]
        source_type  ← "gdelt"
        confidence_score ← 0.4 (GDELT is automated, lower confidence)

    Missing: event_type, lat/lon, country, actors, fatalities
    These require NLP enrichment.
    """
```

### 3.5 Deduplication

```python
# app/ingestion/dedup.py
import hashlib
from urllib.parse import urlparse, urljoin

def compute_content_hash(title: str, url: str) -> str:
    """
    SHA-256 hash of normalized title + URL.
    Used as primary dedup key in the events table.
    """
    normalized = f"{title.lower().strip()}|{normalize_url(url)}"
    return hashlib.sha256(normalized.encode()).hexdigest()

def normalize_url(url: str) -> str:
    """
    Normalize URL for dedup comparison.
    - Lowercase scheme and host
    - Remove trailing slash
    - Remove tracking params (utm_*, ref, fbclid)
    - Sort remaining query params
    """
    parsed = urlparse(url.lower().strip())
    # Remove tracking params
    from urllib.parse import parse_qs, urlencode
    params = {k: v for k, v in parse_qs(parsed.query).items()
              if not k.startswith(("utm_", "ref", "fbclid"))}
    clean_query = urlencode(sorted(params.items()), doseq=True)
    return f"{parsed.scheme}://{parsed.netloc}{parsed.path.rstrip('/')}"

async def is_duplicate(db, content_hash: str) -> bool:
    """Check if content_hash already exists in events table."""
    result = await db.execute(
        select(Event.id).where(Event.content_hash == content_hash).limit(1)
    )
    return result.scalar_one_or_none() is not None

async def batch_dedup(db, events: list[dict]) -> list[dict]:
    """
    Filter out duplicates from a batch of events.
    1. Compute content_hash for each
    2. Check against DB in single query (WHERE content_hash IN (...))
    3. Also check within batch (same event from multiple feeds)
    4. Return only non-duplicate events
    """
```

### 3.6 Event Storage

```python
# app/ingestion/event_storage.py

async def store_events(db: AsyncSession, events: list[dict]) -> int:
    """
    Batch insert events into PostgreSQL.

    Steps:
    1. Compute content_hash for each event
    2. Batch dedup against existing events
    3. Compute PostGIS geography point from lat/lon
    4. Bulk insert using INSERT ... ON CONFLICT (content_hash) DO NOTHING
    5. Publish new event IDs to Redis pub/sub for real-time streaming
    6. Return count of actually inserted events
    """

async def publish_new_events(redis, event_ids: list[str]) -> None:
    """Publish new event IDs to Redis channel 'sentinel:new_events'."""
    for eid in event_ids:
        await redis.publish("sentinel:new_events", eid)
```

### 3.7 ACLED Client

```python
# app/ingestion/acled_client.py

ACLED_BASE_URL = "https://api.acleddata.com/acled/read"

@celery.task(bind=True, max_retries=3)
def fetch_acled_events(self, days_back: int = 7) -> dict:
    """
    Fetch recent conflict events from ACLED API.

    Steps:
    1. Build query params: key, email, event_date range, limit
    2. Make GET request to ACLED API
    3. Parse JSON response → list of event dicts
    4. Normalize each event via normalize_acled_event()
    5. Dedup and store

    Runs: Daily via Celery Beat (ACLED updates weekly, but we check daily)
    Auth: Requires ACLED_API_KEY and ACLED_EMAIL env vars (free registration)
    Rate limit: Respectful — 1 request per fetch, daily
    """

# ACLED API query params:
# key=<API_KEY>&email=<EMAIL>&event_date=2025-02-01|2025-02-08
# &event_date_where=BETWEEN&limit=5000
```

### 3.8 GDELT Client

```python
# app/ingestion/gdelt_client.py

GDELT_DOC_URL = "https://api.gdeltproject.org/api/v2/doc/doc"

@celery.task(bind=True, max_retries=3)
def fetch_gdelt_articles(self, query: str = "conflict OR attack OR violence", timespan: str = "1h") -> dict:
    """
    Fetch recent conflict-related articles from GDELT DOC 2.0 API.

    Steps:
    1. Build URL: ?query=<terms>&mode=artlist&format=json&timespan=<span>&maxrecords=250
    2. Make GET request (no auth required)
    3. Parse JSON response → list of article dicts
    4. Normalize each via normalize_gdelt_article()
    5. Dedup and store

    Runs: Every 30 minutes via Celery Beat
    Auth: None
    Rate limit: Be respectful — no more than 1 request per minute
    """
```

---

## 4. Tier 1 Seed Sources Configuration

```json
[
    {
        "name": "BBC World News",
        "url": "https://feeds.bbci.co.uk/news/world/rss.xml",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "BBC Middle East",
        "url": "https://feeds.bbci.co.uk/news/world/middle_east/rss.xml",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "BBC Africa",
        "url": "https://feeds.bbci.co.uk/news/world/africa/rss.xml",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "BBC Asia",
        "url": "https://feeds.bbci.co.uk/news/world/asia/rss.xml",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "Al Jazeera",
        "url": "https://www.aljazeera.com/xml/rss/all.xml",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "France 24 English",
        "url": "https://www.france24.com/en/rss",
        "source_type": "rss",
        "poll_interval_minutes": 60
    },
    {
        "name": "AP World News",
        "url": "https://apnews.com/world-news.rss",
        "source_type": "rss",
        "poll_interval_minutes": 30
    },
    {
        "name": "Crisis Group",
        "url": "https://www.crisisgroup.org/rss-0",
        "source_type": "rss",
        "poll_interval_minutes": 120
    },
    {
        "name": "ReliefWeb",
        "url": "https://api.reliefweb.int/v1/reports?appname=sentinel&format=json&limit=50",
        "source_type": "api",
        "poll_interval_minutes": 60
    },
    {
        "name": "GDELT Conflict Articles",
        "url": "https://api.gdeltproject.org/api/v2/doc/doc?query=conflict+attack+violence&mode=artlist&format=json&timespan=1h&maxrecords=250",
        "source_type": "api",
        "poll_interval_minutes": 30
    }
]
```

### Seed Loader Script

```python
# app/ingestion/seed_sources.py
import json
from pathlib import Path

async def seed_tier1_sources(db: AsyncSession) -> int:
    """Load Tier 1 sources from seed config. Skip existing (by URL)."""
    seed_file = Path(__file__).parent / "seed_sources.json"
    sources = json.loads(seed_file.read_text())
    count = 0
    for s in sources:
        exists = await db.execute(select(Source).where(Source.url == s["url"]))
        if not exists.scalar_one_or_none():
            db.add(Source(**s))
            count += 1
    await db.commit()
    return count
```

---

## 5. Celery Beat Schedule

```python
# In celery_app.py
celery.conf.beat_schedule = {
    "poll-all-due-feeds": {
        "task": "app.ingestion.feed_poller.poll_all_due_feeds",
        "schedule": 60.0,  # Every minute, check for feeds due
    },
    "fetch-acled-daily": {
        "task": "app.ingestion.acled_client.fetch_acled_events",
        "schedule": crontab(hour=6, minute=0),  # Daily at 06:00 UTC
        "kwargs": {"days_back": 7},
    },
    "fetch-gdelt-hourly": {
        "task": "app.ingestion.gdelt_client.fetch_gdelt_articles",
        "schedule": 1800.0,  # Every 30 minutes
    },
}
```

---

## 6. Adaptive Polling Intervals

```python
def calculate_next_poll(source, poll_result: str, new_items: int) -> int:
    """
    Adjust polling interval based on feed behavior.

    Rules:
    - If 304 Not Modified: increase interval by 25% (max 1440 min)
    - If 200 with 0 new items: increase interval by 10%
    - If 200 with new items: decrease interval by 25% (min 5 min)
    - If error: double interval (up to max), increment consecutive_errors
    - If consecutive_errors >= 5: disable source, alert admin
    """
```

---

## 7. Agent Build Instructions — Step by Step

### Step 1: Create celery_app.py
Set up Celery with Redis broker. Include beat schedule.

### Step 2: Create seed_sources.json
Copy the Tier 1 sources config above.

### Step 3: Create seed_sources.py
Loader script to populate sources table.

### Step 4: Create feed_poller.py
HTTP fetch with ETag/Last-Modified. Celery task.

### Step 5: Create feed_parser.py
feedparser wrapper with HTML stripping and date normalization.

### Step 6: Create normalizer.py
Mappers for RSS entries, ACLED events, GDELT articles.

### Step 7: Create dedup.py
SHA-256 content hash + URL normalization.

### Step 8: Create event_storage.py
Batch insert with ON CONFLICT DO NOTHING.

### Step 9: Create acled_client.py
ACLED API integration Celery task.

### Step 10: Create gdelt_client.py
GDELT DOC 2.0 API integration Celery task.

### Step 11: Create scheduler.py
Adaptive interval calculation.

### Verification Checkpoint
```bash
# Seed sources
python -c "from app.ingestion.seed_sources import seed_tier1_sources; ..."

# Test feed polling manually
python -c "
import feedparser
feed = feedparser.parse('https://feeds.bbci.co.uk/news/world/rss.xml')
print(f'Entries: {len(feed.entries)}')
print(f'Title: {feed.entries[0].title}')
"

# Start Celery worker and verify tasks run
celery -A app.ingestion.celery_app worker --loglevel=info
```

---

## Comparison Tables

### RSS Parser Libraries

| Criteria | feedparser | atoma | fastfeedparser |
|----------|-----------|-------|----------------|
| Format support | RSS 1.0/2.0, Atom, CDF | RSS 2.0, Atom | RSS 2.0, Atom |
| Malformed XML | Very lenient | Strict | Strict |
| Maintenance | Active | Low | Active |
| Speed | Moderate | Fast | Very fast |
| **Recommendation** | **Primary** | Skip | Fallback |

### Message Queue / Task Runner

| Criteria | Celery + Redis | APScheduler | asyncio loop |
|----------|---------------|-------------|-------------|
| Reliability | High (ack, retry) | Medium | Low |
| Monitoring | Flower UI | Manual | Manual |
| Scaling | Multi-worker | Single process | Single process |
| Complexity | Medium | Low | Low |
| **Recommendation** | **YES** | No | Dev only |

---

## Priority Implementation Order

1. **Celery app + beat schedule** — Orchestration layer
2. **Seed sources config** — Data the poller needs
3. **Feed poller** — Core fetch logic
4. **Feed parser** — Turn XML into dicts
5. **Dedup** — Prevent duplicates
6. **Event storage** — Write to PostgreSQL
7. **ACLED client** — Primary structured data source
8. **GDELT client** — Secondary high-volume source
9. **Adaptive scheduling** — Optimization, after basics work

---

## Open Questions

1. **Celery vs native async**: Should MVP use Celery or simpler FastAPI background tasks?
2. **ACLED API key**: Need to register at acleddata.com — free but requires approval
3. **GDELT rate limits**: Undocumented — need to test empirically
4. **Feed health alerting**: Email/Slack notification when feed goes down? Defer to Tier 2?
