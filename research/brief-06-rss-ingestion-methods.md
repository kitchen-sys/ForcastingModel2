# Research Brief 06: Proven RSS Ingestion Methods That Actually Work

## Objective
Research PROVEN, production-tested methods for RSS feed ingestion and real-time news monitoring. Focus exclusively on approaches that work in practice, not theoretical architectures. Find real open-source projects, real libraries, and real implementations.

## Research Areas

### 1. RSS Parsing Libraries That Work
- feedparser (Python) — the standard, but what are its real limitations?
- atoma, reader — alternatives worth using?
- How to handle malformed RSS/Atom feeds in practice
- Encoding issues, date parsing nightmares, real gotchas
- What breaks in production that tutorials never mention

### 2. Feed Polling Strategies That Scale
- Intelligent polling: respecting ETags, Last-Modified headers, 304 responses
- Adaptive polling intervals based on feed update frequency
- How real news aggregators poll thousands of feeds
- Celery vs APScheduler vs custom async polling loops
- Avoiding getting rate-limited or IP-banned

### 3. Real Open-Source RSS Aggregators to Study
- Miniflux — Go-based, self-hosted, battle-tested
- FreshRSS — PHP, mature, huge community
- Tiny Tiny RSS — another established option
- NewsBlur — open source, handles millions of feeds
- What can we steal from their architectures?

### 4. Deduplication That Actually Works
- Near-duplicate detection across multiple sources reporting the same event
- SimHash, MinHash for content similarity
- URL normalization and canonical URL detection
- Title/content fingerprinting approaches
- What duplicate detection methods fail in practice?

### 5. Real-Time Processing Pipeline
- Polling → parsing → dedup → enrichment → storage → notification
- What message queue to use (Redis Streams vs Celery vs simple async)
- Backpressure handling when feeds spike
- Error handling and retry patterns that work

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-06-rss-ingestion-methods.md
