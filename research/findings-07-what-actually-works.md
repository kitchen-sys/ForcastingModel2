# Findings: What Actually Works in Conflict Monitoring Dashboards

## Executive Summary

Real-world conflict monitoring platforms succeed through a hybrid AI-human pipeline (automated crawling + human verification), a map-centric interface with timeline and feed views, and ruthless data source prioritization -- ACLED for structured conflict events, RSS/news feeds for real-time awareness, and GDELT only as a supplementary signal (not ground truth, due to ~55% field accuracy and massive duplication). The MVP that actually works is far simpler than most teams imagine: an interactive map, a filterable event feed, a timeline chart, and reliable data ingestion from 2-3 high-quality sources. Everything else -- NLP, forecasting, entity graphs -- is a Phase 2+ feature that should only be built after the core loop of "ingest, display, filter, alert" is proven and used daily. Alert fatigue is the number one killer of analyst dashboards; solving it requires risk-based scoring, deduplication, and configurable thresholds from day one.

## Research Scope

This research responds to Brief 07: "What Actually Works in Conflict Monitoring Dashboards." It covers:
- Real working OSINT/conflict platforms and their architectures (Liveuamap, ACLED, HDX)
- Features intelligence analysts actually use daily vs. features nobody clicks
- MVP features for a working conflict dashboard
- Common pitfalls and time wasters in dashboard development
- Data quality reality check across free conflict data APIs
- Alert fatigue: the primary usability threat and how to counter it
- Open-source OSINT dashboard projects on GitHub

---

## Detailed Findings

### 1. Real Working Platforms: How They Actually Work

#### Liveuamap -- The Gold Standard for Real-Time Conflict Mapping

Liveuamap (Live Universal Awareness Map) is the most successful independent conflict mapping platform, covering 30+ regions with real-time event plotting. Founded in 2014 by Ukrainian engineers Rodion Rozhkovskiy and Oleksandr Bilchenko, it is the closest public analog to what Sentinel aims to build.

**Architecture (confirmed via BuiltWith, Crunchbase, Wikipedia, and their own documentation):**

| Component | Technology |
|-----------|------------|
| Mapping Library | Leaflet.js (not Mapbox, not Deck.gl) |
| CDN / Security | Cloudflare CDN + Cloudflare optimization |
| Analytics | Google Universal Analytics |
| Push Notifications | OneSignal |
| Mobile | Native iOS and Android apps + Firebase for analytics/crash |
| Infrastructure | EU-based servers behind Cloudflare |
| Data Ingestion | Proprietary AI web crawlers (not open source) |
| Verification | Human editorial team (minimum 2 editors per event) |

**The core pipeline that makes Liveuamap work:**

```
Social Media / News / Open Sources
    --> AI Web Crawlers (proprietary, NLP + geotag extraction)
    --> Algorithmic Correlation & Threshold Detection
    --> Human Editors (>= 2 verify each event before publication)
    --> Interactive Map (Leaflet.js) + Feed + Timeline
    --> API for enterprise customers
    --> Feedback Loop --> Improves crawlers
```

**Key success factors:**
1. **Hybrid AI-human pipeline.** Crawlers handle scale; humans handle accuracy. They do NOT publish unverified events.
2. **Threshold-based event detection.** When correlated messages about an event pass algorithmic thresholds, it gets flagged for human review. This is NOT a firehose -- it is curated.
3. **Social media author profiling.** The system tracks authors' post history, activity level, and network to assess credibility before surfacing their content.
4. **Simple, fast UI.** Leaflet.js -- not some heavyweight 3D globe. Speed and simplicity win.
5. **Monetization via API.** Enterprise customers pay for data access; the public map is ad-supported (Google AdSense).

**Lesson for Sentinel:** The biggest lesson from Liveuamap is that the technology is not the hard part. Leaflet.js + Cloudflare + a good ingestion pipeline is the entire frontend story. The real value is in the data curation pipeline and the editorial process.

#### ACLED Dashboard -- Tableau-Powered Analytics

ACLED's visualization platform is NOT custom-built. It uses:

| Tool | Purpose |
|------|---------|
| Tableau (via Tableau Foundation partnership) | Primary visualization for conflict maps and trend analysis |
| ArcGIS (Esri) | Geospatial mapping for embedded dashboards |
| RESTful API | Programmatic access (JSON, CSV, XML) with Basic HTTP or OAuth auth |
| PowerBI Integration | Enterprise analytics customers |
| R packages (acledR, acled.api) | Academic/researcher access |

**ACLED's dashboard features that analysts actually use:**
- **Trendfinder:** Week-over-week and year-over-year change detection against historical baselines
- **Explorer:** Filter by location, actor, event type with exportable tables and charts
- **CAST (Conflict Alert System):** 6-month forecasts with accuracy metrics
- **Conflict Index:** Four-indicator scoring (deadliness, civilian danger, geographic diffusion, armed group count)
- **Conflict Exposure Calculator:** Integrates ACLED data with WorldPop population estimates

**Lesson for Sentinel:** ACLED proves that you do NOT need to build custom visualization from scratch. Tableau and ArcGIS power the most-used conflict analytics platform in the world. For Sentinel, this suggests using proven libraries (D3, Recharts, Mapbox) rather than building custom rendering engines.

#### HDX (Humanitarian Data Exchange) -- OCHA's Data Portal

HDX is built on CKAN, an open-source Python-based data management system. Its tech stack:

| Component | Technology |
|-----------|------------|
| Core Platform | CKAN (Python) |
| Database | PostgreSQL |
| Search Engine | Apache Solr |
| Caching/Queue | Redis |
| Web Server | Nginx / uWSGI |
| Cloud | AWS |
| APIs | CKAN REST API + HDX HAPI (launched June 2024) |
| Client Libraries | Python (hdx-python-api), R (rhdx) |
| Data Standards | HXL (Humanitarian Exchange Language) |

**Key insight:** By end of 2024, ~80% of HDX's ~20,000 datasets were updated through automated processes. HDX HAPI consolidates multiple partner APIs (conflict events, food security, displacement, humanitarian needs) into a single standardized interface. This is directly relevant for Sentinel's ingestion architecture.

**Lesson for Sentinel:** HDX's success came from standardizing data formats (HXL) and automating ingestion. Sentinel should similarly define a canonical event schema early and build automated ingestion adapters for each source, not manual imports.

---

### 2. Features Analysts Actually Use vs. Features Nobody Clicks

Based on analysis of ACLED, Liveuamap, ICEWS (Lockheed Martin), Crisis Group CrisisWatch, and ViEWS (Uppsala), the feature usage hierarchy is clear:

**Features analysts use daily (non-negotiable):**
1. **Interactive map with event pins** -- This is THE primary view. Analysts orient themselves spatially first.
2. **Filterable event feed / data table** -- Sort by date, type, location, actor. This is where deep analysis happens.
3. **Timeline chart** -- Event counts and fatality trends over time. Week-over-week comparison is critical.
4. **Search** -- Full-text search across events, actors, locations.
5. **Data export (CSV)** -- Analysts always need to pull data into Excel, R, or Python for custom analysis.

**Features analysts use weekly (high value):**
6. **Subnational drill-down** -- Click a country, see provinces. Click a province, see districts.
7. **Historical baseline comparison** -- "Is this week worse than usual for this region?"
8. **Alerts for significant changes** -- Spike detection: "Fatalities in Region X jumped 300% this week."
9. **Country/region profile pages** -- Quick summary dashboard for a specific area of responsibility.

**Features analysts use monthly or rarely (nice-to-have):**
10. **Actor/entity network graphs** -- Link analysis between armed groups. Used in deep investigations, not daily monitoring.
11. **Forecasting/predictive models** -- Interesting but not trusted enough for operational decisions yet.
12. **Sentiment analysis of news coverage** -- Marginal analytical value for conflict monitoring.
13. **Custom report builder** -- Used occasionally for briefings.

**Features that sound cool but nobody uses:**
- 3D globe visualizations (slower than 2D maps, no analytical advantage)
- Real-time streaming counters ("X events in the last hour") -- Creates anxiety, not insight
- AI-generated narrative summaries of conflicts -- Analysts do not trust them for operational use
- Multi-language auto-translation of source text -- Useful in theory, unreliable in practice for conflict terminology

---

### 3. MVP Features for a Working Dashboard

Based on what successful platforms shipped first and what analysts actually need:

#### Tier 1: Must-Have (Week 1-2 MVP)

| Feature | Justification |
|---------|---------------|
| Interactive map with event markers | Primary analyst view. Use Mapbox GL JS or Leaflet.js. Color-code by event type. |
| Event feed / data table | Sortable, filterable list of events with date, location, type, actors, fatalities. |
| Date range filter | Analysts always ask "what happened this week?" |
| Event type filter | Battles, protests, riots, violence against civilians, explosions/remote violence. |
| Location filter (country/region) | Analysts have geographic areas of responsibility. |
| Basic timeline chart | Event count over time. Use Recharts or Tremor. One chart, one metric. |
| CSV export | Non-negotiable. Analysts will leave if they cannot export data. |

#### Tier 2: Should-Have (Week 3-4)

| Feature | Justification |
|---------|---------------|
| Subnational drill-down on map | Click country to see sub-regions. Critical for granular analysis. |
| Historical baseline comparison | "Is this normal?" is the most important analytical question. |
| Alert thresholds (spike detection) | Automated email/webhook when event count or fatalities exceed N% above baseline. |
| Full-text search | Search events by keyword (actor name, weapon type, location). |
| Country profile pages | One-page summary: map + trend chart + recent events + key actors. |

#### Tier 3: Could-Have (Month 2-3)

| Feature | Justification |
|---------|---------------|
| Actor tracking / profiles | Which armed groups are most active? Where? Trend over time. |
| Multiple data source integration | Combine ACLED + RSS feeds + ReliefWeb. Show source attribution. |
| Saved views / bookmarks | Let analysts save their filtered views for daily checking. |
| Role-based dashboards | Different default views for different analyst teams. |

#### Tier 4: Won't-Have in V1

| Feature | Why Defer |
|---------|-----------|
| NLP / entity extraction pipeline | Complex to build, requires training data, marginal value until data flow is proven. |
| Forecasting / predictive models | ACLED CAST already does this. Do not replicate. Integrate their output instead. |
| Entity relationship graphs (Neo4j) | Deep investigation feature. Not daily monitoring. Build after user demand. |
| Real-time social media ingestion | Requires massive infrastructure. Use curated feeds (RSS, ACLED) first. |
| Multi-language support | English first. Expand only if user base demands it. |
| AI-generated briefings | Trust gap. Analysts will not use AI summaries for operational decisions in V1. |

---

### 4. Common Pitfalls and Time Wasters

#### Pitfall 1: Overengineering the Data Pipeline Before Having Users

The Iran-Israel War OSINT project (GitHub: danielrosehill/Iran-Israel-War-OSINT) demonstrated that simple keyword monitoring with timestamps was more effective for early warning than complex NLP pipelines. The author found that "both the October 2024 attacks and 2025 attacks were preceded by news coverage with clear indications of imminent hostilities" -- detected through basic keyword monitoring, not sophisticated AI.

**Rule:** Get data flowing to a screen first. Optimize the pipeline after you have users telling you what is too slow or too noisy.

#### Pitfall 2: Building NLP Features Nobody Asked For

Liveuamap's success comes from human editors, not NLP. ACLED's success comes from human coders, not machine learning. The platforms that work use AI for *crawling and filtering*, not for *analysis and presentation*. Analysts want raw events with metadata, not AI-generated interpretations.

**Rule:** NLP should reduce the work of data ingestion (deduplication, geolocation, categorization). It should NOT generate analyst-facing content.

#### Pitfall 3: Spending Months on a Custom Map

Liveuamap uses Leaflet.js. That is a free, open-source library that can be set up in hours. ACLED uses Tableau and ArcGIS -- both off-the-shelf products. No successful conflict dashboard built a custom rendering engine.

**Rule:** Use Mapbox GL JS or Leaflet.js. Get a map on screen in day one. Style it later.

#### Pitfall 4: Trying to Monitor Too Many Sources at Once

ACLED monitors sources in 75+ languages with a global team of trained researchers. Sentinel does not have that team. Starting with 50 RSS feeds and 5 APIs simultaneously will produce a firehose of noisy, uncurated data that overwhelms the interface and the team.

**Rule:** Start with 2-3 high-quality sources (ACLED API + ReliefWeb API + 5-10 curated RSS feeds). Add sources incrementally as you build confidence in your deduplication and normalization pipeline.

#### Pitfall 5: Building Alerting Before Having Reliable Data Flow

You cannot build meaningful alerts on unreliable data. If your ingestion pipeline drops events, duplicates events, or has inconsistent update timing, every alert will be either a false positive or a missed detection.

**Rule:** Ingestion reliability must be proven (>99% uptime, consistent update cadence, zero duplicates) before alerts are built on top of it. This means logging, monitoring, and testing the pipeline itself before layering on user-facing alerting.

#### Pitfall 6: The 3D Globe Trap

Multiple open-source projects (G-APT Monitor, various threat map projects) build impressive-looking 3D globes. None of them are used for daily analysis. 3D globes are demo-ware. Analysts need 2D maps they can pan, zoom, and click quickly.

**Rule:** 2D maps with vector tiles. No globes. No WebGL particle effects. Speed and clarity over visual impressiveness.

---

### 5. Data Quality Reality Check

#### ACLED: The Best Available, But Not Perfect

**Reliability:** High. Human-coded by trained researchers with multi-language expertise. Multi-review process for inter-coder reliability. Updated weekly.

**Known limitations (per ACLED's own documentation):**
- Urban bias: Events in rural areas are underreported due to media coverage patterns.
- December reporting gap: Less conflict reported when English-language journalists are on holiday.
- Nascent conflicts poorly covered: Small, peripheral, or early-stage conflicts get less source coverage.
- Historical coverage varies: Africa since 1997, Asia since 2010, full global coverage more recent.
- Geocoding precision varies: Some events coded to city level, others only to province.

**API access:** Free tier (myACLED) provides access to dashboards and aggregated data. Disaggregated event-level data may require higher-tier access. RESTful API with JSON/CSV/XML output. OAuth or cookie-based auth.

**Verdict for Sentinel:** PRIMARY data source. Use ACLED as the backbone for structured conflict event data. Accept its weekly update cadence as a known limitation and supplement with real-time RSS feeds for breaking events.

#### GDELT: Massive Scale, Terrible Accuracy

**Reliability:** Low for event-level analysis. High for media attention tracking.

**Critical limitations (documented by UK ONS, multiple academic papers):**
- ~55% accuracy in keyword fields. Nearly half of coded events have incorrect metadata.
- Up to 20% data redundancy. The same event published in multiple outlets is recorded as multiple discrete events.
- Western-centric and English-language bias baked into the dataset.
- "649 kidnappings" in GDELT means "649 news stories about kidnappings," NOT 649 kidnapping events.
- Automated coding misses one or both conflict actors in large proportions of events.
- UK Office for National Statistics concluded: "This data will most likely not be considered a source of robust data."
- Hammond & Weidmann warned against using GDELT for "geospatial analyses at the subnational level."

**Verdict for Sentinel:** SUPPLEMENTARY source only. Use GDELT for media attention signals and trend detection ("is media coverage of conflict in Region X increasing?"). Do NOT use GDELT as ground truth for conflict event counts or geolocation. Never display GDELT event counts as if they represent actual events.

#### ReliefWeb API: High Quality, Different Purpose

**Reliability:** Very high for humanitarian information. Curated by OCHA editorial team monitoring 4,000+ sources.

**Limitations:**
- NOT an event-level conflict database. ReliefWeb curates reports, situation updates, and analysis -- not discrete events.
- Best for humanitarian context: crisis overviews, disaster reports, funding appeals, policy updates.
- Free API with no fees. Good documentation.
- Job and training data only available after 2011.

**Verdict for Sentinel:** COMPLEMENTARY source for context. Use ReliefWeb to enrich conflict events with humanitarian context (situation reports, crisis severity, response status). Do not attempt to extract conflict events from ReliefWeb -- that is not its purpose.

#### HDX HAPI: The New Unified API (Launched June 2024)

HDX HAPI consolidates data from multiple partner APIs into a single standardized interface covering conflict events, food security, humanitarian needs, and internal displacement. By end of 2024, ~80% of HDX's ~20,000 datasets updated automatically.

**Verdict for Sentinel:** HIGH PRIORITY integration target. HAPI provides a single endpoint for multiple humanitarian data streams that would otherwise require building separate integrations. Evaluate as a potential single-source replacement for individual API integrations.

#### RSS Feeds: The Unsung Workhorse

The most reliable real-time data sources for conflict monitoring are curated RSS feeds from:
- Reuters, AP, BBC (major wire services -- high reliability, fast updates)
- International Crisis Group (expert analysis, weekly updates)
- OCHA ReliefWeb (situation reports, daily updates)
- Al Jazeera, France 24 (non-Western perspective coverage)

**Verdict for Sentinel:** Start with 5-10 high-quality RSS feeds for real-time awareness. RSS is simple to ingest, requires no API keys, and provides immediate value. The signal-to-noise ratio depends entirely on source selection -- curate carefully.

---

### 6. Alert Fatigue: The #1 Usability Killer

SOC teams field an average of 4,484 alerts per day (Vectra 2023 report). Of these, 67% are ignored. Most security analysts spend one-third of their workday investigating false alarms. Malicious actors have learned to weaponize alert fatigue by launching high volumes of low-priority events to mask real threats ("alert storming").

**Solutions that work in production:**

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| Risk-based scoring | Score each event by severity, proximity to user's area of interest, deviation from baseline | HIGH -- reduces noise by 60-80% when well-tuned |
| Dynamic thresholds | Automatically adjust alert thresholds based on historical patterns (LogicMonitor approach) | HIGH -- prevents seasonal false positives |
| Deduplication | Merge related events before alerting (Splunk ITSI correlation) | CRITICAL -- without this, one real-world event generates dozens of alerts |
| Parent-child dependencies | Suppress downstream alerts when an upstream cause is known (Icinga model) | MEDIUM -- prevents alert storms from cascading failures |
| Role-based filtering | Show analysts only alerts for their area of responsibility (Cyware CFIR) | HIGH -- reduces irrelevant alerts by 70%+ |
| AI-powered triage | ML models classify alerts by severity and auto-resolve low-confidence ones (IBM, Swimlane) | MEDIUM-HIGH -- requires training data and tuning period |
| User-configurable notification levels | Let analysts set their own thresholds for email/push/in-app notifications | CRITICAL -- different analysts have different noise tolerance |

**For Sentinel's MVP:**
1. Implement risk-based scoring from day one: severity (fatalities, event type) + recency + deviation from baseline.
2. Implement aggressive deduplication before any alert logic.
3. Let users configure their own alert thresholds per region and event type.
4. Default to LOW alert volume. It is far better to send 5 meaningful alerts per day than 50 noisy ones.
5. Provide a "digest" mode: daily summary email with top changes, no individual event alerts.

---

### 7. Open-Source OSINT Dashboard Projects Worth Studying

| Project | GitHub | Tech Stack | Status | Relevance |
|---------|--------|-----------|--------|-----------|
| Global Threat Map | unicodeveloper/globalthreatmap | Next.js, TypeScript, Mapbox, OpenAI | Active (2024-2025) | Most directly relevant. Full-featured, self-hostable. Uses AI for analysis. |
| Ukraine Conflict Monitor | gmesite/Ukraine-Conflict-Monitor | R Shiny + ACLED data | Academic project | Shows minimum viable ACLED dashboard in R. |
| ACLED Dashboard (R Shiny) | elhajjar/acled | R Shiny + ACLED data | Proof of concept | Map + graphic view with filters. Simple but functional. |
| ACLED Data Pipeline | projectmesadata/armedconflict | Python + ACLED API | Data pipeline only | Good reference for ACLED API integration patterns. |
| G-APT Monitor | carlitosspro/G-APT-Monitor | 3D visualization | Niche (cyber APTs) | Shows what NOT to do for daily use (3D globe). |
| Iran-Israel War OSINT | danielrosehill/Iran-Israel-War-OSINT | Monitoring + keyword analysis | Active (2025) | Valuable for early warning pattern research. |
| Awesome OSINT | jivoi/awesome-osint | Curated link list | Maintained | Tool discovery resource, not a dashboard. |
| OSINT Stuff Tool Collection | cipher387/osint_stuff_tool_collection | Curated link list | Maintained | Hundreds of categorized OSINT tools. |

**Key finding:** The most directly relevant project is **Global Threat Map** (unicodeveloper/globalthreatmap), which is a full Next.js/TypeScript/Mapbox application with AI-powered conflict analysis. It demonstrates that a working conflict dashboard can be built with standard web technologies. However, its reports take 5-10 minutes to generate, highlighting the latency challenge of AI-powered analysis.

---

### 8. OSINT Tools Analysts Actually Use Daily

The U.S. DNI's 2024-2026 strategy named OSINT "The INT of First Resort." The House Permanent Select Committee on Intelligence created a dedicated OSINT subcommittee in 2024.

**Tools in daily professional use (ranked by prevalence):**

1. **Maltego** -- Relationship mapping and link analysis. Input an entity, get all connected entities.
2. **Palantir Gotham/Foundry** -- Big data analytics platform. Government and military standard.
3. **i2 Analyst's Notebook** -- Visualization and analysis. Multiple data views for reporting.
4. **SpiderFoot** -- Automated OSINT reconnaissance from 100+ sources.
5. **Shodan** -- Search engine for internet-connected devices. Infrastructure reconnaissance.
6. **ShadowDragon** -- Real-time monitoring and link analysis from 200+ sources.
7. **OSINT Industries** -- Real-time account lookups across platforms.
8. **Cobwebs** -- Threat intelligence with AI-driven location intelligence and maps.
9. **ExifTool** -- Metadata extraction from images and documents.
10. **Videris** -- Enterprise OSINT automation with human-in-the-loop decisions.

**Newer tools gaining traction (2024-2025):**
- **1 TRACE** -- Combines social media, geospatial, cyber, and financial OSINT.
- **VenariX** -- Automated dark web and social media threat intelligence.
- **SkopeNow** -- Social media intelligence and fake account detection.

**Key workflow insight:** Most analysts use AI daily, primarily for collection, analysis, and writing. They report clear productivity gains but flag concerns about cost, data access limitations, information overload, fast-changing tools, and growing legal/ethical pressures. Python and Jupyter Notebooks are standard for custom analysis workflows.

**Implication for Sentinel:** Sentinel does not need to replace Maltego or Palantir. It needs to occupy the "daily awareness" slot -- the first screen an analyst checks each morning to understand what changed overnight. Think "weather dashboard for conflict" not "investigation platform."

---

## Comparison Tables

### Platform Architecture Comparison

| Criteria | Liveuamap | ACLED Dashboard | HDX | Sentinel (Recommended) |
|----------|-----------|-----------------|-----|----------------------|
| Map Library | Leaflet.js | Tableau/ArcGIS | N/A (data portal) | Mapbox GL JS |
| Data Ingestion | Proprietary AI crawlers | Human coders, 75+ languages | CKAN + automated APIs | FastAPI + Celery workers |
| Verification | Human editors (2+ per event) | Multi-review coding process | Partner-curated data | Source attribution + confidence scoring |
| Update Frequency | Near real-time | Weekly | ~80% automated, varies | Hourly (RSS) + weekly (ACLED) |
| Frontend | Custom (Leaflet + Cloudflare) | Tableau embedded | CKAN templates | Next.js + TypeScript |
| Backend | Unknown (proprietary) | Unknown (behind Tableau) | CKAN (Python) + PostgreSQL + Solr | FastAPI + PostgreSQL + Redis |
| Open Source | No | No (data is, platform is not) | Yes (CKAN-based) | Yes (planned) |

### Data Source Quality Comparison

| Criteria | ACLED | GDELT | ReliefWeb | RSS Feeds | HDX HAPI |
|----------|-------|-------|-----------|-----------|----------|
| Event-level accuracy | HIGH (human-coded) | LOW (~55% field accuracy) | N/A (not event-level) | Varies by source | Varies by partner |
| Update frequency | Weekly | Every 15 minutes | Daily | Minutes to hours | Varies |
| Deduplication | Built-in (human review) | POOR (up to 20% redundancy) | Built-in (editorial) | None (must build) | Varies |
| Geographic coverage | Global (200+ countries) | Global | Global (humanitarian focus) | Depends on feed selection | Global (humanitarian) |
| Cost | Free tier available; higher tiers may cost | Free | Free | Free | Free |
| API quality | Good (REST, OAuth) | Complex (BigQuery, raw files) | Good (REST) | Simple (XML/RSS) | Good (REST, new in 2024) |
| Best used for | Primary conflict events | Media attention signals | Humanitarian context | Breaking news awareness | Consolidated humanitarian data |
| Recommendation | PRIMARY source | SUPPLEMENTARY only | COMPLEMENTARY context | REAL-TIME feed | EVALUATE for consolidation |

---

## Priority Implementation Order

1. **Data ingestion from ACLED API + 5-10 curated RSS feeds** -- This is the foundation. Without reliable data flow, nothing else matters. ACLED provides structured conflict events; RSS provides real-time breaking news. Build adapters for these first. Target: working data pipeline in week 1.

2. **Interactive map with event markers (Mapbox GL JS)** -- The primary analyst view. Plot ACLED events on a map with color-coding by event type. Add RSS-sourced events with a different visual treatment (lower confidence). Target: map on screen by end of week 1.

3. **Filterable event feed / data table** -- Sortable by date, type, location, actor, fatalities. This is where analysts do 60% of their work after initial spatial orientation on the map. Include full-text search. Target: week 2.

4. **Timeline chart (event counts over time)** -- Simple bar or line chart showing event volume by day/week. Use Recharts or Tremor. Include fatality trend overlay. Target: week 2.

5. **CSV export** -- Non-negotiable. Analysts will leave if they cannot export. Wire up early. Target: week 2.

6. **Date range, event type, and location filters** -- Cross-cutting filters that apply to map, feed, and timeline simultaneously. This is the core interaction model. Target: week 2-3.

7. **Historical baseline comparison** -- "Is this week normal?" Show current week vs. rolling 52-week average for any selected region. This is the most requested analytical feature. Target: week 3-4.

8. **Alert thresholds with spike detection** -- Configurable per-region, per-event-type alerts when activity exceeds N% above baseline. Default to conservative thresholds to avoid alert fatigue. Target: week 4.

9. **ReliefWeb API integration for humanitarian context** -- Enrich conflict events with situation reports, crisis status, and humanitarian response data. Target: month 2.

10. **HDX HAPI evaluation and integration** -- Assess whether HAPI can replace individual API integrations for humanitarian data streams. Target: month 2.

---

## Cost Analysis

### Free Tier Stack (Viable for MVP)

| Component | Tool | Cost |
|-----------|------|------|
| Mapping | Mapbox GL JS | Free up to 50,000 map loads/month |
| Conflict data | ACLED API (myACLED free tier) | Free (aggregated data; event-level may require application) |
| Humanitarian data | ReliefWeb API | Free, no fees |
| Humanitarian data | HDX HAPI | Free |
| News feeds | RSS (Reuters, BBC, ICG, etc.) | Free |
| Frontend hosting | Vercel (Next.js) | Free tier available |
| Backend hosting | Railway / Render / Fly.io | Free tier ~$5-7/month for small apps |
| Database | PostgreSQL (Supabase free tier or self-hosted) | Free up to 500MB |
| Cache | Redis (Upstash free tier) | Free up to 10,000 commands/day |
| Search | Elasticsearch (self-hosted) or Meilisearch | Free (self-hosted) |

**Total MVP cost: $0 - $15/month** using free tiers.

### Production Scale Estimates

| Component | Tool | Estimated Monthly Cost |
|-----------|------|----------------------|
| Mapbox GL JS | 50K-200K loads | $0 - $250/month |
| Cloud hosting (backend) | AWS / GCP / Azure | $50 - $200/month |
| PostgreSQL + PostGIS + TimescaleDB | Managed (RDS or equivalent) | $50 - $150/month |
| Redis | Managed | $15 - $50/month |
| Elasticsearch | Managed (Elastic Cloud or self-hosted) | $50 - $200/month |
| CDN | Cloudflare | Free - $20/month |
| ACLED API | Higher tier if needed | $0 - unknown (contact for pricing) |
| Monitoring | Prometheus + Grafana (self-hosted) | Free |

**Total production cost: $165 - $870/month** depending on scale and tier choices.

---

## Open Questions

1. **ACLED API rate limits and event-level data access for free tier** -- The free myACLED tier provides dashboard access and aggregated data. It is unclear whether disaggregated event-level API access is available on the free tier or requires a paid/research agreement. Next step: Register for myACLED account and test API access levels directly.

2. **Liveuamap API pricing and data licensing** -- Liveuamap offers an enterprise API but pricing is not public. Is their data available for integration, or is it proprietary? Next step: Contact liveuamap.com/promo/api for pricing and terms.

3. **GDELT deduplication feasibility** -- With ~20% redundancy and ~55% field accuracy, can GDELT data be cleaned to a usable state, or is the effort not worth the outcome? Next step: Run a pilot: pull one week of GDELT data for a specific country, attempt deduplication, measure resulting accuracy against ACLED ground truth.

4. **HDX HAPI data latency and completeness** -- HAPI launched in June 2024. How mature is it? What is the typical data latency? Are all advertised data streams actually available and reliable? Next step: Build a test integration and monitor for one week.

5. **Optimal RSS feed selection** -- Which specific RSS feeds provide the best signal-to-noise ratio for conflict monitoring? This requires empirical testing. Next step: Monitor 20 candidate feeds for 2 weeks, measure update frequency, relevance rate, and geographic coverage.

6. **Alert threshold calibration** -- What baseline deviation percentage should trigger alerts? Too low = alert fatigue. Too high = missed events. This requires user testing with real analysts. Next step: Start with 200% above rolling weekly baseline as default, let users adjust, collect data on which thresholds are kept vs. modified.

7. **Mobile access requirements** -- Do conflict analysts actually use dashboards on mobile devices, or is this desktop-only? This affects responsive design investment. Next step: Survey target users or review usage analytics from similar platforms.

8. **Leaflet.js vs. Mapbox GL JS for this use case** -- Liveuamap uses Leaflet. Mapbox GL JS offers vector tiles and smoother performance with large marker counts. At what event volume does the choice matter? Next step: Benchmark both with 10K, 50K, and 100K markers.

---

## Sources & References

### Platforms Analyzed
- Liveuamap: https://liveuamap.com/about
- Liveuamap Wikipedia: https://en.wikipedia.org/wiki/Liveuamap
- Liveuamap Tech Stack (BuiltWith): https://builtwith.com/liveuamap.com
- Liveuamap API: https://liveuamap.com/promo/api
- ACLED Platform: https://acleddata.com/conflict-data/data-platforms
- ACLED Trendfinder: https://acleddata.com/platform/trendfinder
- ACLED Explorer: https://acleddata.com/platform/explorer
- ACLED Conflict Index: https://acleddata.com/series/acled-conflict-index
- ACLED API Docs: https://acleddata.com/acled-api-documentation
- ACLED on Tableau: https://www.tableau.com/foundation/featured-projects/acled
- ACLED Data Quality: https://acleddata.com/faq/how-quality-acled-data-ensured
- ACLED Known Limitations: https://acleddata.com/methodology/overview-acleds-data-enhancement-projects-and-known-data-limitations
- HDX Platform: https://data.humdata.org/
- HDX Developer Resources: https://data.humdata.org/faqs/devs
- HDX CKAN API: https://centre.humdata.org/ufaq-category/hdx-resources-devs-accessing-hdx-by-api/
- HDX GitHub (OCHA-DAP): https://github.com/OCHA-DAP
- HDX HAPI: https://centre.humdata.org/what-we-do/
- ReliefWeb API: https://reliefweb.int/help/api
- State of Open Humanitarian Data 2024: https://centre.humdata.org/the-state-of-open-humanitarian-data-2024/

### Data Quality Research
- GDELT Data Quality (UK ONS): https://www.ons.gov.uk/peoplepopulationandcommunity/birthsdeathsandmarriages/deaths/methodologies/globaldatabaseofeventslanguageandtonegdeltdataqualitynote
- GDELT Decontextualized Data: https://source.opennews.org/articles/gdelt-decontextualized-data/
- GDELT Academic Use: https://www.globe-project.eu/the-empirical-use-of-gdelt-big-data-in-academic-research_13809.pdf
- ACLED Comparison Working Paper: https://acleddata.com/report/working-paper-comparing-conflict-data
- Dataset Scope Conditions (Nature): https://www.nature.com/articles/s41599-023-01559-4

### Open-Source Projects
- Global Threat Map: https://github.com/unicodeveloper/globalthreatmap
- Ukraine Conflict Monitor: https://github.com/gmesite/Ukraine-Conflict-Monitor
- ACLED R Shiny Dashboard: https://github.com/elhajjar/acled
- ACLED Data Pipeline: https://github.com/projectmesadata/armedconflict
- Iran-Israel War OSINT: https://github.com/danielrosehill/Iran-Israel-War-OSINT
- G-APT Monitor: https://github.com/carlitosspro/G-APT-Monitor
- Awesome OSINT: https://github.com/jivoi/awesome-osint
- OSINT Stuff Tool Collection: https://github.com/cipher387/osint_stuff_tool_collection
- Bellingcat Toolkit: https://bellingcat.gitbook.io/toolkit/more/all-tools/liveuamap

### Alert Fatigue Research
- IBM Alert Fatigue: https://www.ibm.com/think/topics/alert-fatigue
- IBM AI Alert Reduction: https://www.ibm.com/think/insights/alert-fatigue-reduction-with-ai-agents
- Splunk Alert Noise Reduction: https://www.splunk.com/en_us/solutions/alert-noise-reduction.html
- Icinga Alert Fatigue: https://icinga.com/blog/alert-fatigue-monitoring/
- Cyware Combating Analyst Fatigue: https://www.cyware.com/blog/stop-the-noise-combating-analyst-fatigue-with-contextual-threat-information-93af
- SOC Alert Fatigue (CyberDefenders): https://cyberdefenders.org/blog/soc-alert-fatigue/

### OSINT Tools and Trends
- OSINT Roadmap 2025: https://osintguide.com/2024/11/14/osint-roadmap/
- Top OSINT Tools 2025: https://hackread.com/2025-top-osint-tools-take-on-open-source-intel/
- OSINT Industry Review 2025: https://www.osintnewsletter.osint-jobs.com/p/2025-year-in-review-what-we-have
- ShadowDragon OSINT Tools: https://shadowdragon.io/blog/best-osint-tools/
- ViEWS Forecasting: https://viewsforecasting.org/
- ICEWS (Lockheed Martin): https://www.lockheedmartin.com/en-us/capabilities/research-labs/advanced-technology-labs/icews.html
