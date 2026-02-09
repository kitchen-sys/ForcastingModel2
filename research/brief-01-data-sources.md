# Research Brief 01: Data Sources & Ingestion Pipeline

## Objective
Research all viable data sources for monitoring global conflicts in real-time, and design an ingestion pipeline architecture.

## Research Areas

### 1. RSS Feeds & News Sources
- Identify the best RSS feeds for global conflict monitoring (Reuters, AP, Al Jazeera, BBC World, etc.)
- Find specialized conflict/security RSS feeds (ACLED, GDELT, Crisis Group, SIPRI, etc.)
- Research government and military RSS feeds (DOD, NATO, UN Security Council)
- Evaluate feed reliability, update frequency, and data quality

### 2. Conflict Data APIs & Datasets
- ACLED (Armed Conflict Location & Event Data) - API access, data schema, update frequency
- GDELT Project - Global Database of Events, Language, and Tone
- Uppsala Conflict Data Program (UCDP)
- START Global Terrorism Database
- UN OCHA ReliefWeb API
- Crisis24 / risk intelligence APIs
- SIPRI databases (arms transfers, military expenditure)

### 3. Social Media & OSINT Sources
- Twitter/X API for conflict zone monitoring
- Telegram channel monitoring for conflict zones
- Reddit threat intelligence subreddits
- Liveuamap and similar OSINT platforms
- Bellingcat-style open source intelligence methods

### 4. Geospatial & Satellite Data
- Satellite imagery APIs (Sentinel, Planet Labs, Maxar)
- NASA FIRMS (Fire Information for Resource Management)
- OpenStreetMap for conflict zone mapping
- Humanitarian OpenStreetMap Team (HOT) data

### 5. Ingestion Pipeline Design
- How to build a scalable RSS polling system
- Rate limiting and respectful scraping practices
- Data normalization across different source formats
- Deduplication strategies for cross-source events
- Real-time vs batch processing tradeoffs
- Message queue architecture (Kafka, RabbitMQ, Redis Streams)

## Deliverable
Produce a comprehensive research document with:
- A ranked list of data sources with pros/cons
- API endpoints, authentication requirements, rate limits
- Recommended ingestion architecture diagram (described textually)
- Data schema recommendations for normalized conflict events
- Cost analysis (free vs paid sources)
