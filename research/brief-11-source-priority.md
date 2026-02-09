# Research Brief 11: Data Source Priority — Which Feeds and APIs to Wire Up First

## Objective
Produce a RANKED, PRIORITIZED list of every data source Sentinel should connect to, in the exact order they should be implemented. Include connection details for each.

## Context
Read these files first:
- findings-01-data-sources.md (comprehensive source catalog)
- findings-06-rss-ingestion-methods.md (how to actually ingest them)
- findings-07-what-actually-works.md (which sources have best signal-to-noise)

## Research Areas

### 1. RSS Feeds — Ranked by Value
- Find the ACTUAL RSS feed URLs for top conflict news sources
- Test which feeds are actually alive and updating (search for current status)
- Rank by: update frequency, content quality, conflict relevance, reliability
- Tier 1 (Day 1): The 5-10 feeds we wire up first
- Tier 2 (Week 2): Next 10-15 feeds
- Tier 3 (Later): Everything else

### 2. Conflict Data APIs — Ranked by Ease + Value
- ACLED: exact API endpoint, auth process, what data we get, how fast
- GDELT: how to query it, what's actually useful vs noise
- ReliefWeb: API quality and conflict relevance
- For each: effort to integrate (S/M/L) vs value delivered (High/Med/Low)

### 3. Source Reliability Baseline
- Which sources have the best uptime?
- Which sources have the cleanest data (least parsing headaches)?
- Which sources update most frequently?
- Which sources have the most conflict-relevant content vs general noise?

### 4. Implementation Checklist Per Source
- For each Tier 1 source: URL, feed type (RSS/Atom/API), auth needed, polling interval, expected volume, parsing notes
- This should be copy-pasteable into code as a config

## Deliverable
Write findings to: /home/user/ForcastingModel2/research/findings-11-source-priority.md
