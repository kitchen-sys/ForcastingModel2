# Findings: Proven RSS Ingestion Methods for Production

## Executive Summary

Production RSS ingestion requires solving five interrelated problems: reliable parsing of malformed feeds, intelligent polling that respects publisher resources, deduplication of near-identical articles from multiple sources, a robust processing pipeline with retry logic, and scalable scheduling across thousands of feeds. The recommended approach combines `feedparser` (for robustness with malformed feeds) or `fastfeedparser` (for speed with well-formed feeds) with Celery-based distributed task scheduling, adaptive polling intervals using ETags/Last-Modified headers, and MinHash+LSH deduplication via the `datasketch` library backed by Redis. This architecture is validated by studying three battle-tested open-source aggregators: Miniflux (Go), NewsBlur (Python/Django), and FreshRSS (PHP).

## Research Scope

This document responds to Research Brief 06: "Proven RSS Ingestion Methods That Actually Work." It covers:

1. RSS parsing libraries -- real limitations and production gotchas
2. Feed polling strategies that scale -- ETags, adaptive intervals, scheduling
3. Lessons from open-source RSS aggregators (Miniflux, NewsBlur, FreshRSS)
4. Deduplication methods that work in practice (SimHash, MinHash, URL normalization)
5. Real-time processing pipeline architecture (queues, backpressure, error handling)

---

## Detailed Findings

### 1. RSS Parsing Libraries -- What Actually Works

#### feedparser (Python) -- The Robust Standard

`feedparser` (v6.0.12, maintained by Kurt McKee) is the most widely used Python RSS/Atom parser. Its key strength is handling malformed feeds -- it almost never raises exceptions, instead setting a `bozo` flag when problems occur.

**Production gotchas discovered through research:**

- **Silent failures are the biggest risk.** feedparser does not raise exceptions on parse errors by design. Instead, it sets `d.bozo = 1` and populates `d.bozo_exception`. If you do not check the bozo flag, you will silently get incomplete or wrong data. Parsing can stop mid-feed without any error raised, silently dropping entries.
- **Encoding nightmares.** feedparser cannot distinguish between iso-8859-1 and windows-1252. Many feeds are mislabeled. Install the `chardet` library for better auto-detection. Some servers serve feeds as `text/plain` or `application/octet-stream`, which feedparser will attempt to parse but flags as `NonXMLContentType`.
- **Performance is poor at scale.** Benchmarked at approximately 2.5 feeds/sec on an Intel Core i5 750. For a system polling hundreds of feeds concurrently, feedparser becomes a CPU bottleneck. The old `speedparser` alternative reached 65 feeds/sec with HTML cleaning and 200 feeds/sec without, but it is unmaintained.
- **Crash-causing edge cases** (fixed in recent versions but instructive): `TypeError` when URLs contain embedded credentials, `UnicodeDecodeError` on empty type attributes, `UnicodeEncodeError` on Unicode characters in URL paths.

**Essential bozo exception types to handle in production:**

```python
import feedparser

IGNORABLE_BOZO = (
    feedparser.CharacterEncodingOverride,
    feedparser.NonXMLContentType,
)

CRITICAL_BOZO = (
    xml.sax._exceptions.SAXParseException,
    UnicodeEncodeError,
    UnicodeDecodeError,
)

def parse_feed(url: str) -> dict | None:
    d = feedparser.parse(url)
    if d.bozo:
        exc = d.bozo_exception
        if isinstance(exc, CRITICAL_BOZO):
            logger.error(f"Critical parse error for {url}: {exc}")
            return None
        else:
            logger.warning(f"Non-critical bozo for {url}: {exc}")
    if not d.entries:
        logger.warning(f"No entries found for {url}")
        return None
    return d
```

#### FastFeedParser (by Kagi) -- The Modern Fast Alternative

Released November 2024, `fastfeedparser` is a new high-performance alternative created by Kagi (the search engine company). It powers Kagi Small Web's feed processing at scale.

- **10-50x faster than feedparser** depending on feed structure (benchmarks show 5.5x to 50.1x speedups across test feeds)
- Familiar API designed to be a drop-in replacement
- MIT licensed, very actively maintained (19 releases, latest v0.4.5 on January 1, 2026)
- **Install:** `pip install fastfeedparser`
- **GitHub:** https://github.com/kagisearch/fastfeedparser

**Tradeoff:** Less battle-tested than feedparser for malformed feed handling. Best for well-formed feeds from known sources.

#### Atoma -- Lightweight and Memory-Efficient

`atoma` is a minimal Atom/RSS/JSON Feed parser for Python 3.

- **6x faster than feedparser** and uses **55% less memory** (benchmarks: 1.5s/28MB vs 9.0s/61MB parsing 157 feeds)
- **Does NOT handle malformed feeds** -- will raise exceptions on bad XML
- **Does NOT do HTML sanitization** -- you must sanitize entry content yourself
- **Does NOT resolve relative links**
- **GitHub:** https://github.com/NicolasLM/atoma
- Less actively maintained than feedparser or fastfeedparser

#### Recommendation for Sentinel

Use a **two-tier parsing strategy:**

1. **Primary parser: `fastfeedparser`** for known, well-formed feeds (major news agencies like Reuters, AP, BBC whose feeds are reliably well-formed). Speed matters when polling hundreds of feeds.
2. **Fallback parser: `feedparser`** when `fastfeedparser` fails or for unknown/untrusted feed sources. feedparser's robustness with malformed XML is unmatched.
3. **Always validate output** -- never assume fields exist. Check for `title`, `link`, `published`, `summary` on every entry regardless of parser.

---

### 2. Feed Polling Strategies That Scale

#### Conditional HTTP Requests (ETags and Last-Modified)

The single most impactful optimization for RSS polling is respecting HTTP conditional request headers. This avoids re-downloading and re-parsing feeds that have not changed.

**How it works:**

1. On first fetch, store the response's `ETag` and `Last-Modified` headers.
2. On subsequent fetches, send `If-None-Match: <etag>` and `If-Modified-Since: <last-modified>`.
3. If the feed has not changed, the server returns `304 Not Modified` with no body -- saving bandwidth and CPU.

**feedparser supports this natively:**

```python
import feedparser

# First fetch
d = feedparser.parse('https://example.com/feed.xml')
etag = d.get('etag')
modified = d.get('modified')

# Subsequent fetches
d = feedparser.parse(
    'https://example.com/feed.xml',
    etag=etag,
    modified=modified
)
if d.status == 304:
    # Feed has not changed, skip processing
    pass
```

**Critical implementation detail:** Store ETags and Last-Modified timestamps per-feed in your database. Many tutorials skip this and re-download everything every time.

#### Adaptive Polling Intervals

Not all feeds update at the same rate. Reuters may publish dozens of articles per hour; a niche conflict analysis blog may publish once a week. Polling both every 15 minutes wastes resources.

**Miniflux's `entry_frequency` scheduler** is the proven model:

- Analyze the average time between entries over the past week
- Set the polling interval proportional to the update frequency
- Enforce a configurable minimum interval (e.g., 5 minutes) and maximum interval (e.g., 24 hours)
- New/unknown feeds start at a default interval (e.g., 60 minutes) and adapt over time

**Implementation pattern:**

```python
from datetime import datetime, timedelta

MIN_INTERVAL = timedelta(minutes=5)
MAX_INTERVAL = timedelta(hours=24)
DEFAULT_INTERVAL = timedelta(hours=1)

def calculate_adaptive_interval(feed) -> timedelta:
    """Calculate next poll interval based on recent entry frequency."""
    recent_entries = feed.entries_from_last_7_days()
    if len(recent_entries) < 2:
        return DEFAULT_INTERVAL

    # Calculate average gap between entries
    timestamps = sorted([e.published for e in recent_entries])
    gaps = [timestamps[i+1] - timestamps[i] for i in range(len(timestamps)-1)]
    avg_gap = sum(gaps, timedelta()) / len(gaps)

    # Poll at half the average gap (to catch new entries promptly)
    interval = avg_gap / 2
    return max(MIN_INTERVAL, min(MAX_INTERVAL, interval))
```

#### Responsible Polling Practices

FreshRSS enforces a hard minimum of 20 minutes between polls per feed. This is not just politeness -- **aggressive polling gets you IP-banned.**

**Rules from production aggregators:**

- **Never poll faster than every 5 minutes** for any single feed
- **Respect `Cache-Control` and `Expires` headers** -- if a feed says it updates hourly, do not poll every 5 minutes
- **Use a recognizable User-Agent string** that includes a contact URL (e.g., `Sentinel/1.0 (+https://sentinel.example.com/bot)`)
- **Implement exponential backoff on errors** -- if a feed returns 5xx or times out, double the interval each time up to a max
- **Track consecutive errors per feed** -- Miniflux stops polling after 3 consecutive errors and requires manual re-enable. This prevents wasting resources on dead feeds.

#### Scheduling: Celery Beat vs APScheduler vs Custom Async

| Feature | Celery Beat | APScheduler | Custom asyncio loop |
|---------|------------|-------------|-------------------|
| Distributed workers | Yes (native) | Limited | Manual |
| Dynamic schedule changes | Via django-celery-beat or redbeat | Yes (job stores) | Manual |
| Persistence across restarts | Yes (DB-backed) | Yes (DB job stores) | Manual |
| Overhead | Higher (broker required) | Lower | Lowest |
| Production-proven at scale | Instagram, NewsBlur, thousands of projects | Moderate adoption | Depends on implementation |
| Best for | 100+ feeds, multi-worker | 10-100 feeds, single process | Prototype, small scale |

**Recommendation for Sentinel:** Celery Beat with Redis broker. NewsBlur uses this exact stack (Django + Celery + Redis) to handle millions of feeds. Use `django-celery-beat` or `redbeat` for dynamic per-feed scheduling stored in the database.

---

### 3. Lessons from Open-Source RSS Aggregators

#### Miniflux (Go) -- Minimalist and Efficient

- **GitHub:** https://github.com/miniflux/v2
- **Language:** Go, compiled to single binary
- **Database:** PostgreSQL only (no ORM)
- **Architecture:** CLI -> Daemon -> Worker pool
- **Memory usage:** A few MB even with hundreds of feeds

**Key architectural lessons:**

- **Two polling schedulers:** `round_robin` (fixed rotation) and `entry_frequency` (adaptive based on past week's publishing pattern). The `entry_frequency` approach is what Sentinel should adopt.
- **Respects all HTTP caching headers:** ETag, Last-Modified, If-None-Match, If-Modified-Since, Cache-Control, Expires.
- **Error limit per feed:** After N consecutive parse errors (default 3), stops polling that feed. Prevents wasting resources on permanently broken feeds.
- **Batch processing:** Polls feeds in configurable batch sizes per cycle, controlled by `POLLING_FREQUENCY` and `BATCH_SIZE` settings.
- **Worker concurrency:** Configurable number of concurrent feed fetch workers.

**What to steal:** The adaptive entry_frequency scheduler logic and the per-feed error tracking with automatic disable.

#### NewsBlur (Python/Django) -- Full-Scale Production

- **GitHub:** https://github.com/samuelclay/NewsBlur
- **Language:** Python 3.7+ / Django / Backbone.js
- **Stack:** PostgreSQL + MongoDB + Redis + Elasticsearch + Celery
- **Scale:** Handles millions of feeds in production at newsblur.com

**Key architectural lessons:**

- **Polyglot persistence:** PostgreSQL for relational data (feeds, subscriptions, accounts), MongoDB for document-oriented story storage, Redis for caching and real-time features, Elasticsearch for full-text search. This is exactly the stack Sentinel should consider.
- **Celery for distributed feed crawling:** Feed fetching/parsing is distributed across Celery workers. This allows horizontal scaling -- add more workers to handle more feeds.
- **Real-time push via WebSockets:** Stories are pushed to connected clients in real-time as they are fetched, not just on page refresh.
- **Intelligence/training layer:** Users can train the system on what they like/dislike, and it prioritizes stories accordingly. This is analogous to Sentinel's conflict relevance scoring.

**What to steal:** The Celery-based distributed crawl architecture and the polyglot persistence model (PostgreSQL for structure, document store for articles, Redis for real-time, Elasticsearch for search).

#### FreshRSS (PHP) -- Pragmatic and Ecosystem-Responsible

- **GitHub:** https://github.com/FreshRSS/FreshRSS
- **Language:** PHP
- **Database:** MySQL/MariaDB, PostgreSQL, or SQLite
- **Scale:** Tested up to 20k+ feeds, 1000+ users, runs on Raspberry Pi

**Key architectural lessons:**

- **Mutex-based update locking:** The `actualize_script.php` uses a mutex to prevent concurrent update runs. Simple but effective at preventing duplicate work.
- **Hard minimum polling interval of 20 minutes per feed.** This is a deliberate ecosystem-responsibility choice. Sentinel should enforce a similar floor.
- **WebSub (PubSubHubbub) for push-based updates:** For feeds that support WebSub, FreshRSS receives instant push notifications and reduces polling to once per 24 hours. This dramatically reduces load for supported feeds.
- **Cron-triggered, not daemon-based:** Updates are triggered by external cron jobs, not an internal scheduler. This simplifies deployment and makes the system more resilient to crashes.

**What to steal:** The WebSub integration (for feeds that support it) and the responsible minimum polling interval enforcement.

---

### 4. Deduplication That Actually Works

News deduplication operates at three levels: exact duplicate detection, URL-based deduplication, and near-duplicate content detection.

#### Level 1: Exact Duplicate Detection (Fast, Cheap)

Use a hash of the article's unique identifier (typically `guid` from RSS, or a hash of `link + title`).

```python
import hashlib

def article_fingerprint(entry: dict) -> str:
    """Generate a unique fingerprint for exact duplicate detection."""
    # Prefer GUID if available (most reliable)
    if entry.get('id'):
        return hashlib.sha256(entry['id'].encode()).hexdigest()
    # Fallback to link + title hash
    key = f"{entry.get('link', '')}{entry.get('title', '')}"
    return hashlib.sha256(key.encode()).hexdigest()
```

Store fingerprints in a Redis set for O(1) lookup. Before processing any article, check if its fingerprint already exists.

#### Level 2: URL Normalization (Medium Effort, High Value)

The same article often appears with different URL parameters (tracking params, session IDs, etc.). Normalize URLs before fingerprinting.

**Recommended library: `url-normalize`** (PyPI: `url_normalize` v2.2.1)

```python
from url_normalize import url_normalize

# These all become the same normalized URL:
url_normalize("https://example.com/article?utm_source=twitter&id=123")
url_normalize("https://EXAMPLE.COM/article?id=123&utm_source=facebook")
# Both normalize to: https://example.com/article?id=123
```

**Additional URL normalization rules for news deduplication:**

- Strip tracking parameters: `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `fbclid`, `gclid`
- Lowercase the hostname
- Remove trailing slashes
- Remove default ports (`:80`, `:443`)
- Sort query parameters alphabetically
- Remove fragment identifiers (`#section`)

**For advanced canonical detection:** Fetch the article page and look for `<link rel="canonical" href="...">` in the HTML. This is the publisher's declaration of the canonical URL. The `url-normalize` and `urlcanon` libraries can help, but extracting canonical tags requires an HTTP request + HTML parsing, so only do this for articles that pass initial dedup checks.

#### Level 3: Near-Duplicate Content Detection (Higher Effort, Critical for News)

Multiple news outlets often report the same event with slightly different wording. Reuters publishes, then AP, BBC, and dozens of smaller outlets all write their own version of the same story. These are not exact duplicates -- they are near-duplicates.

**MinHash + LSH is the proven production approach.**

- Google used MinHash+LSH for Google News personalization (reported 2007).
- The `datasketch` Python library is the standard production tool.
- It can run with a **Redis backend** for scalable lookups across millions of documents.

**How MinHash+LSH works for news deduplication:**

1. **Shingling:** Convert article text to a set of n-grams (e.g., 3-word shingles)
2. **MinHash:** Generate a compact signature (e.g., 128 hash values) that approximates the Jaccard similarity of the original shingle sets
3. **LSH (Locality-Sensitive Hashing):** Index signatures so that similar documents hash to the same bucket, enabling sub-linear lookup time

**Production implementation with datasketch:**

```python
from datasketch import MinHash, MinHashLSH

# Create LSH index with Jaccard threshold of 0.5
# (articles sharing 50%+ content are considered near-duplicates)
lsh = MinHashLSH(threshold=0.5, num_perm=128)

def compute_minhash(text: str) -> MinHash:
    """Compute MinHash signature for article text."""
    m = MinHash(num_perm=128)
    # Create 3-word shingles
    words = text.lower().split()
    for i in range(len(words) - 2):
        shingle = ' '.join(words[i:i+3])
        m.update(shingle.encode('utf-8'))
    return m

def is_near_duplicate(article_text: str, article_id: str) -> bool:
    """Check if article is a near-duplicate of any existing article."""
    mh = compute_minhash(article_text)
    # Query for similar articles
    result = lsh.query(mh)
    if result:
        return True  # Near-duplicate found
    # No duplicate -- insert into index
    lsh.insert(article_id, mh)
    return False
```

**For production scale, use the Redis-backed MinHashLSH:**

```python
from datasketch import MinHashLSH

lsh = MinHashLSH(
    threshold=0.5,
    num_perm=128,
    storage_config={
        'type': 'redis',
        'redis': {'host': 'localhost', 'port': 6379},
    }
)
```

This allows the LSH index to persist across process restarts and be shared across multiple workers.

**Key tuning parameters:**

| Parameter | Recommended Value | Effect |
|-----------|------------------|--------|
| `threshold` | 0.4 - 0.6 | Lower = more aggressive dedup, higher = more permissive |
| `num_perm` | 128 | Higher = more accurate but slower. 128 is good balance. |
| Shingle size | 3 words | Smaller = more sensitive to minor changes |

**Alternative: SimHash for faster fingerprinting**

SimHash is faster to compute and compare (Hamming distance on fixed-length bit vectors) but less accurate for detecting subtle near-duplicates. It works well when articles are very similar (>80% overlap) but struggles with moderate similarity. For news deduplication where outlets rewrite stories significantly, MinHash+LSH is more reliable.

**Recommended Python libraries for deduplication:**

| Library | Purpose | PyPI | Notes |
|---------|---------|------|-------|
| `datasketch` | MinHash + LSH | `pip install datasketch` | Production-grade, Redis-backed, most recommended |
| `text-dedup` | All-in-one dedup toolkit | `pip install text-dedup` | Supports MinHash, SimHash, Bloom, Suffix Array |
| `url-normalize` | URL normalization | `pip install url-normalize` | Strip tracking params, normalize URLs |
| `simhash` | SimHash fingerprinting | `pip install simhash` | Fast but less accurate for near-duplicates |

---

### 5. Real-Time Processing Pipeline Architecture

#### Recommended Pipeline Flow

```
[RSS Feeds] --> [Scheduler (Celery Beat)] --> [Fetch Workers (Celery)]
                                                      |
                                              [Parse & Validate]
                                                      |
                                              [Dedup Check (Redis)]
                                                      |
                                          [Enrich (NLP, Geo, Entity)]
                                                      |
                                              [Store (PostgreSQL)]
                                                      |
                                          [Notify (WebSocket Push)]
```

#### Component Details

**1. Scheduler (Celery Beat + Redis)**

- Celery Beat dispatches feed-fetch tasks according to per-feed schedules stored in the database
- Use `redbeat` (Redis-backed Celery Beat scheduler) for dynamic schedule management -- no restarts needed when adding/removing feeds or changing intervals
- Each feed has its own schedule entry with an adaptive interval

**2. Fetch Workers (Celery Workers)**

- Pool of Celery workers that execute feed-fetch tasks
- Each worker: sends conditional HTTP request (with ETags/Last-Modified) -> receives response -> hands off to parser
- Use `requests` or `httpx` with configurable timeouts (connect: 10s, read: 30s)
- Set a proper User-Agent header
- Implement retry with exponential backoff: `autoretry_for=(RequestException,), retry_backoff=True, max_retries=3`

**3. Parse and Validate**

- Parse response with `fastfeedparser` (primary) or `feedparser` (fallback)
- Validate each entry has required fields: title, link, published date
- Normalize dates to UTC (feeds use inconsistent date formats -- feedparser handles most of them)
- Sanitize HTML content (strip dangerous tags, fix relative URLs)

**4. Dedup Check (Redis-backed)**

- Level 1: Check article fingerprint (SHA-256 of GUID or link+title) against Redis set. O(1) lookup.
- Level 2: Normalize URL and check again.
- Level 3: For articles that pass Level 1 and 2, compute MinHash and query LSH index for near-duplicates.
- If duplicate found at any level, log it and skip further processing.

**5. Enrich**

- After dedup, enqueue enrichment tasks (NLP entity extraction, geolocation, conflict classification)
- These are separate Celery tasks that can run asynchronously
- Store raw article immediately, enrich asynchronously

**6. Store (PostgreSQL + Elasticsearch)**

- Insert article into PostgreSQL (with PostGIS for geospatial data)
- Index in Elasticsearch for full-text search
- Update feed metadata (last_fetched, next_fetch_at, etag, last_modified, error_count)

**7. Notify (WebSocket Push)**

- After storage, publish to a Redis Pub/Sub channel or Redis Stream
- WebSocket server (FastAPI WebSocket endpoint) subscribes and pushes new articles to connected dashboard clients

#### Error Handling and Retry Patterns

```python
from celery import shared_task
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task(
    bind=True,
    autoretry_for=(ConnectionError, TimeoutError),
    retry_backoff=True,        # Exponential backoff
    retry_backoff_max=3600,    # Max 1 hour between retries
    retry_jitter=True,         # Add randomness to prevent thundering herd
    max_retries=5,
    acks_late=True,            # Re-queue if worker crashes mid-task
    reject_on_worker_lost=True,
)
def fetch_feed(self, feed_id: int):
    """Fetch and process a single RSS feed."""
    feed = Feed.objects.get(id=feed_id)
    try:
        response = fetch_with_conditional_headers(feed)
        if response.status_code == 304:
            feed.update_next_fetch_time()
            return
        entries = parse_feed_response(response)
        new_entries = deduplicate(entries)
        store_entries(new_entries)
        feed.reset_error_count()
        feed.update_next_fetch_time()
    except FeedParseError as e:
        feed.increment_error_count()
        if feed.error_count >= MAX_CONSECUTIVE_ERRORS:
            feed.disable()
            logger.error(f"Feed {feed.url} disabled after {MAX_CONSECUTIVE_ERRORS} errors")
        raise  # Let Celery retry handle it
```

#### Backpressure Handling

When major events occur (e.g., a conflict escalation), many feeds update simultaneously, creating a spike in new articles.

**Strategies:**

- **Celery rate limits:** Use `rate_limit='100/m'` on the fetch task to cap throughput
- **Redis Streams as a buffer:** Instead of processing articles synchronously in the fetch worker, push them to a Redis Stream. A separate consumer group processes articles at a controlled rate.
- **Priority queues:** Use separate Celery queues for high-priority feeds (major news agencies) and low-priority feeds (niche blogs). High-priority queue gets more workers.

```python
# Celery queue configuration
CELERY_TASK_ROUTES = {
    'feeds.tasks.fetch_feed_high_priority': {'queue': 'feeds_high'},
    'feeds.tasks.fetch_feed_low_priority': {'queue': 'feeds_low'},
    'feeds.tasks.enrich_article': {'queue': 'enrichment'},
}
```

---

## Comparison Tables

### RSS Parsing Libraries Comparison

| Criteria | feedparser | FastFeedParser | Atoma | lxml (manual) |
|----------|-----------|---------------|-------|---------------|
| Speed (relative) | 1x (baseline) | 10-50x faster | 6x faster | ~10x faster |
| Malformed feed handling | Excellent | Good | Poor (raises exceptions) | None (manual) |
| Memory usage | High (61MB/157 feeds) | Low-medium | Low (28MB/157 feeds) | Low |
| Active maintenance | Yes (v6.0.12) | Very active (v0.4.5, Jan 2026) | Less active | Built-in |
| Feed format support | RSS, Atom, RDF, JSON | RSS, Atom, RDF, JSON | Atom, RSS, JSON | Manual |
| HTML sanitization | Built-in | Built-in | None | None |
| Production adoption | Extremely high | Growing (Kagi) | Low | Common |
| **Recommendation** | Fallback parser | **Primary parser** | Not recommended | Special cases |

### Open-Source Aggregator Architecture Comparison

| Feature | Miniflux | NewsBlur | FreshRSS |
|---------|----------|----------|----------|
| Language | Go | Python/Django | PHP |
| Database | PostgreSQL | PostgreSQL + MongoDB | MySQL/PG/SQLite |
| Task queue | Built-in worker pool | Celery + Redis | Cron-triggered |
| Adaptive polling | Yes (entry_frequency) | Yes | No (fixed interval) |
| WebSub support | No | No | Yes |
| ETag/Last-Modified | Full support | Full support | Partial |
| Error auto-disable | Yes (configurable limit) | Yes | No |
| Max tested scale | Hundreds of feeds | Millions of feeds | 20k+ feeds |
| Memory footprint | Few MB | Hundreds of MB | Low |
| **Best lesson for Sentinel** | Adaptive scheduler | Celery + polyglot persistence | WebSub + responsible polling |

### Deduplication Methods Comparison

| Method | Speed | Accuracy | Scale | Storage | Best For |
|--------|-------|----------|-------|---------|----------|
| SHA-256 fingerprint | O(1) | Exact only | Unlimited | Tiny (Redis set) | Exact duplicates |
| URL normalization | O(1) | High for same-source | Unlimited | Tiny | Same article, different URL params |
| MinHash + LSH | O(1) query | High (tunable) | Millions of docs | Medium (Redis) | Near-duplicate news articles |
| SimHash | O(1) | Moderate | Millions of docs | Small | High-similarity duplicates (>80%) |
| TF-IDF cosine similarity | O(n) | Very high | Thousands | Large | Small-scale, high accuracy needs |
| Sentence Transformers | O(n) | Highest | Thousands | Very large | Semantic similarity |

---

## Priority Implementation Order

1. **Basic feed fetcher with feedparser/fastfeedparser** -- Get articles flowing. Implement conditional HTTP requests (ETags, Last-Modified) from day one. This is the foundation everything else builds on. Use feedparser initially for maximum robustness, swap in fastfeedparser for known well-formed feeds once the pipeline is stable.

2. **Celery-based task scheduling with Redis** -- Replace any polling loop with Celery Beat + per-feed scheduled tasks. This enables distributed workers and dynamic schedule management. Use `redbeat` for Redis-backed schedules. This unblocks horizontal scaling.

3. **Three-tier deduplication pipeline** -- Implement in order: (a) SHA-256 fingerprint check in Redis, (b) URL normalization with tracking parameter stripping, (c) MinHash+LSH via `datasketch` with Redis backend. Each tier catches progressively subtler duplicates.

4. **Adaptive polling intervals** -- Implement Miniflux-style entry_frequency scheduling. Analyze each feed's publishing cadence over the past 7 days and adjust polling intervals dynamically. Enforce minimum 5-minute and maximum 24-hour bounds.

5. **Error tracking and automatic feed disable** -- Track consecutive errors per feed. After N failures (recommend 5), automatically disable the feed and alert operators. Implement exponential backoff on transient errors.

6. **WebSub integration** -- For feeds that support WebSub/PubSubHubbub (WordPress, Mastodon, Blogger, FeedBurner), register as a subscriber to receive push notifications. Reduce polling of WebSub-enabled feeds to once per 24 hours as a consistency check.

7. **Real-time push to dashboard** -- After articles are stored, publish to Redis Pub/Sub. FastAPI WebSocket endpoints subscribe and push new articles to connected clients.

---

## Cost Analysis

### Infrastructure Costs

| Component | Free/Self-Hosted Option | Managed Service Option | Estimated Monthly Cost |
|-----------|------------------------|----------------------|----------------------|
| Redis (broker + cache + dedup) | Self-hosted Redis | AWS ElastiCache t3.small | $0 / $25-50 |
| PostgreSQL | Self-hosted PostgreSQL | AWS RDS t3.small | $0 / $30-60 |
| Celery workers | Same server | AWS ECS/Fargate | $0 / $20-50 |
| Elasticsearch | Self-hosted | AWS OpenSearch t3.small | $0 / $25-50 |

**Total self-hosted cost: $0** (all open-source software, runs on a single VPS)
**Total managed services: $100-210/month** for moderate scale

### Library Costs

All recommended libraries are free and open-source:

| Library | License | Cost |
|---------|---------|------|
| feedparser | BSD-2-Clause | Free |
| fastfeedparser | MIT | Free |
| datasketch | MIT | Free |
| url-normalize | MIT | Free |
| Celery | BSD-3-Clause | Free |
| Redis | BSD-3-Clause | Free |
| text-dedup | Apache-2.0 | Free |

### Scale Estimates

For Sentinel's expected scale (~200-500 RSS feeds, conflict/security domain):

- A single Celery worker can handle 200+ feeds with 15-minute polling intervals
- Redis memory for dedup: ~50MB for 1M article fingerprints + MinHash index
- PostgreSQL storage: ~1GB/year for article metadata at 1000 articles/day
- The entire system can run on a single 2-core, 4GB RAM VPS for initial deployment

---

## Open Questions

1. **FastFeedParser production reliability at scale** -- FastFeedParser is new (November 2024). While Kagi uses it in production, there are limited third-party production reports. Needs testing with the specific conflict/security RSS feeds Sentinel will ingest. Suggested next step: Run a side-by-side comparison of feedparser vs fastfeedparser on 50 target feeds for 1 week.

2. **MinHash threshold tuning for conflict news** -- The optimal Jaccard similarity threshold for deduplicating conflict news articles is not known. Too aggressive (0.3) will merge legitimately different articles about the same topic; too permissive (0.7) will miss near-duplicates. Suggested next step: Collect 1000 articles, manually label duplicates, test thresholds 0.3-0.7 to find optimal F1 score.

3. **WebSub adoption among conflict/security feeds** -- How many of the target RSS feeds (ACLED, ReliefWeb, Crisis Group, Reuters, etc.) support WebSub? If very few do, the WebSub integration effort may not be justified. Suggested next step: Check each target feed's HTTP headers for `Link: <hub>; rel="hub"`.

4. **Celery vs native Python asyncio for feed fetching** -- For a FastAPI-based system that is already async, running a separate Celery infrastructure adds complexity. An alternative is `asyncio` + `httpx` with `APScheduler` for scheduling. This would be simpler but less battle-tested at scale. Suggested next step: Prototype both approaches with 50 feeds and compare resource usage, error handling, and code complexity.

5. **Rate limiting by news source** -- Some news sources may rate-limit or block automated access. Need to identify which target sources have explicit bot policies and whether API keys are available. Suggested next step: Test each target feed URL with a bot User-Agent and document response codes.

6. **Cross-language deduplication** -- Conflict events are reported in multiple languages. MinHash+LSH operates on text tokens and will not detect that an English Reuters article and a French AFP article describe the same event. Semantic deduplication (e.g., sentence transformers with multilingual models) may be needed but is much more expensive. Suggested next step: Evaluate multilingual sentence-transformer models (e.g., `paraphrase-multilingual-MiniLM-L12-v2`) for cross-language dedup feasibility.

---

## Sources & References

### Libraries and Tools
- feedparser documentation: https://feedparser.readthedocs.io/en/latest/
- feedparser GitHub (Kurt McKee): https://github.com/kurtmckee/feedparser
- FastFeedParser (Kagi): https://github.com/kagisearch/fastfeedparser
- FastFeedParser PyPI: https://pypi.org/project/fastfeedparser/
- Atoma: https://github.com/NicolasLM/atoma
- datasketch (MinHash/LSH): https://github.com/ekzhu/datasketch
- text-dedup: https://github.com/ChenghaoMou/text-dedup
- url-normalize: https://pypi.org/project/url-normalize/
- urlcanon (IIPC): https://github.com/iipc/urlcanon
- speedparser: https://github.com/jmoiron/speedparser
- rss-apifier (DRF+Celery+Redis example): https://github.com/ralphqq/rss-apifier

### Open-Source Aggregators
- Miniflux: https://github.com/miniflux/v2
- Miniflux documentation: https://miniflux.app/docs/configuration.html
- NewsBlur: https://github.com/samuelclay/NewsBlur
- FreshRSS: https://github.com/FreshRSS/FreshRSS
- FreshRSS feed updating docs: https://freshrss.github.io/FreshRSS/en/admins/08_FeedUpdates.html
- FreshRSS WebSub docs: https://freshrss.github.io/FreshRSS/en/users/WebSub.html

### Research and Articles
- Near-duplicate detection with datasketch: https://yorko.github.io/2023/practical-near-dup-detection/
- SimHash guide: https://spotintelligence.com/2023/01/02/simhash/
- MinHash LSH in Milvus: https://milvus.io/blog/minhash-lsh-in-milvus-the-secret-weapon-for-fighting-duplicates-in-llm-training-data.md
- Document deduplication with LSH: https://mattilyra.github.io/2017/05/23/document-deduplication-with-lsh.html
- URL normalization for dedup (Cornell/ACM): https://dl.acm.org/doi/10.1145/1645953.1646283
- Benchmarking NDD for paywalled news (ACM 2025): https://dl.acm.org/doi/10.1145/3701716.3715303
- Building an intelligent RSS fetcher: https://nikolajjsj.com/blog/building-an-intelligent-rss-feed-fetcher/
- feedparser ETag docs: https://pythonhosted.org/feedparser/http-etag.html
- feedparser encoding docs: https://pythonhosted.org/feedparser/character-encoding.html
- Celery best practices: https://denibertovic.com/posts/celery-best-practices/
- FreshRSS scaling PR (20k+ feeds): https://github.com/FreshRSS/FreshRSS/pull/4347
- Mozilla Chronicle URL canonicalization notes: https://github.com/mozilla/chronicle/wiki/%5Bresearch-notes%5D-URL-Canonicalization-and-Normalization
