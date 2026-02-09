# Research Findings 03: AI/NLP & Conflict Analytics Engine

## Executive Summary

This document presents comprehensive research findings on AI and NLP techniques for automatically analyzing, classifying, and extracting intelligence from conflict-related data streams. It covers NLP pipeline architecture, specific model recommendations, entity extraction approaches, conflict classification taxonomy, geoparsing strategy, predictive analytics assessment, alert system design, and LLM integration planning.

---

## 1. Recommended NLP Pipeline Architecture

### Overview

The recommended architecture is a multi-stage pipeline combining efficient transformer-based models for high-throughput processing with LLM-based components for complex reasoning tasks:

```
Raw Text Ingestion
    |
    v
[Stage 1] Language Detection & Translation
    |
    v
[Stage 2] Named Entity Recognition (NER)
    |
    v
[Stage 3] Event Extraction & Classification
    |
    v
[Stage 4] Geoparsing & Geocoding
    |
    v
[Stage 5] Entity Resolution & Knowledge Graph Construction
    |
    v
[Stage 6] Conflict Classification & Severity Scoring
    |
    v
[Stage 7] Anomaly Detection & Alert Generation
    |
    v
[Stage 8] LLM Summarization & Briefing Generation
```

### Design Principles

1. **Hybrid Architecture**: Use spaCy transformer pipelines for high-throughput NER and dependency parsing; reserve LLM API calls (Claude, GPT-4) for complex reasoning, summarization, and low-resource extraction tasks.
2. **Modular & Swappable**: Each stage should be independently deployable and replaceable. Use standardized intermediate formats (JSON-LD, STIX, or custom schemas).
3. **Batch + Streaming**: Support both batch processing of historical data and real-time streaming of incoming news feeds.
4. **Cost Efficiency**: Run spaCy/transformer models on GPU for NER/classification (high volume, low cost per item); use LLM APIs selectively for high-value tasks (summarization, complex event extraction).

---

## 2. Named Entity Recognition (NER) — Model Recommendations

### Primary Recommendation: spaCy + Transformer Backends

spaCy v3+ with transformer backends provides the best balance of accuracy and production readiness for conflict news NER. The `spacy-transformers` extension enables transformer-level accuracy within spaCy's efficient pipeline.

#### Recommended Models

| Model | Use Case | Performance | Notes |
|-------|----------|-------------|-------|
| `en_core_web_trf` | English NER baseline | F1 ~0.90 | spaCy's transformer pipeline using RoBERTa |
| `xx_ent_wiki_sm` | Multilingual NER | F1 ~0.82 | Covers 100+ languages, good for initial pass |
| `roberta-large` (HuggingFace: `roberta-large`) | Fine-tuning base for English conflict NER | State-of-the-art | Requires domain-specific fine-tuning |
| `xlm-roberta-large` (HuggingFace: `xlm-roberta-large`) | Multilingual fine-tuning base | Strong cross-lingual transfer | Best for Arabic, Russian, Ukrainian |
| `dslim/bert-base-NER` | Off-the-shelf NER | F1 ~0.91 on CoNLL-2003 | Good for quick prototyping |
| `Jean-Baptiste/camembert-ner` | French NER | F1 ~0.90 | For French-language conflict reporting |

#### Custom Entity Types for Conflict Domain

Beyond standard NER categories (PERSON, ORG, GPE, LOC, DATE), the system should recognize:
- **WEAPON**: Specific weapon systems, munitions, platforms
- **MILITARY_UNIT**: Battalion, brigade, division designations
- **CONFLICT_EVENT**: Battle, airstrike, shelling, protest
- **INFRASTRUCTURE**: Bridges, power plants, hospitals (targets)
- **CASUALTY_FIGURE**: Numerical casualty mentions

#### Fine-Tuning Approach

1. **Annotation**: Use Prodigy or Doccano to annotate 2,000-5,000 conflict-specific sentences with custom entity types.
2. **Transfer Learning**: Fine-tune `xlm-roberta-large` on annotated conflict data. This requires less annotated data and achieves better results than training from scratch.
3. **Few-Shot with LLMs**: For rare entity types or new conflict domains, use GPT-4/Claude for few-shot NER (F1 ~0.93-0.94 with structured prompting) before investing in large-scale annotation.
4. **Evaluation**: Benchmark against BERT-based models (F1 ~0.91 on standard NER) and spaCy CPU pipeline (F1 ~0.83 on Russian cultural news texts).

### Benchmarks (from 2025 arXiv study on Russian news)

| Model | F1 Score | Precision | Notes |
|-------|----------|-----------|-------|
| GPT-4o (structured prompt) | 0.93 | 0.96 | Highest overall, API cost |
| GPT-4.1 | 0.94 | — | April 2025, rapid improvement |
| GPT-4 | — | 0.99 | Highest precision |
| DeepPavlov (RoBERTa) | ~0.88 | — | Self-hosted, efficient |
| spaCy Russian Pipeline | 0.83 | — | CPU-efficient, fast |

---

## 3. Multilingual NLP Support

### Language Coverage Requirements

Primary languages for conflict monitoring: English, Arabic, Russian, Ukrainian, French, Chinese, Spanish, Portuguese, Turkish.

### Recommended Multilingual Models (HuggingFace IDs)

| Model ID | Languages | Parameters | Use Case |
|----------|-----------|------------|----------|
| `xlm-roberta-large` | 100+ languages | 560M | Cross-lingual NER, classification |
| `xlm-roberta-base` | 100+ languages | 280M | Lighter cross-lingual tasks |
| `EuroLLM/EuroLLM-9B` | 32 languages incl. Arabic, Russian, Ukrainian, Turkish | 9B | Generative tasks, translation |
| `CohereForAI/aya-101` | 101 languages | 13B | Massively multilingual generation |
| `facebook/m2m100_1.2B` | 100 languages | 1.2B | Translation (Ukrainian <-> 100 langs) |
| `Helsinki-NLP/opus-mt-*` | Bilingual pairs | ~300M | Efficient pairwise translation |
| `facebook/xglm-4.5B` | 134 languages incl. Ukrainian | 4.5B | Multilingual autoregressive LM |
| `Goader/modern-liberta-large` | Ukrainian + English | ~400M | Ukrainian-specialized BERT, 8192 context |
| `xlm-roberta-base-uk` | Ukrainian + English | 280M | Truncated XLM-R for Ukrainian |
| `tabularisai/multilingual-sentiment-analysis` | 20+ languages incl. Russian | 66M | Sentiment analysis |

### Language-Specific Resources

**Arabic:**
- `aubmindlab/bert-base-arabertv02` — Arabic BERT for NER/classification
- `CAMeL-Lab/bert-base-arabic-camelbert-mix` — Arabic BERT with mixed dialect support

**Russian:**
- `DeepPavlov/rubert-base-cased` — Russian BERT
- spaCy Russian Pipeline (`ru_core_news_lg`) — CPU-efficient

**Ukrainian:**
- Kobza dataset: ~1.3TB uncompressed text, 60 billion tokens for pretraining
- UNLP shared tasks for Ukrainian NLP benchmarking
- `Goader/modern-liberta-large` — ModernBERT Large with Ukrainian tokenizer

### Translation Strategy

For non-English sources, use a two-phase approach:
1. **Phase 1**: Run multilingual NER models (XLM-RoBERTa) directly on source-language text to extract entities.
2. **Phase 2**: Translate full text to English using M2M-100 or OPUS-MT for downstream English-based processing (event extraction, summarization).

This preserves entity integrity (names, places) that can be lost in translation-first approaches.

---

## 4. Event Extraction

### Approaches (Ranked by Recommendation)

#### Approach A: LLM-Based Structured Extraction (Primary)

Use Claude API or GPT-4 with structured prompting to extract conflict events as JSON objects:

```json
{
  "event_type": "armed_clash",
  "sub_event_type": "shelling",
  "date": "2025-01-15",
  "location": "Kherson Oblast, Ukraine",
  "actors": [
    {"name": "Russian Armed Forces", "type": "state_military"},
    {"name": "Ukrainian Armed Forces", "type": "state_military"}
  ],
  "casualties": {"reported_killed": 3, "reported_injured": 12},
  "weapons_used": ["artillery", "MLRS"],
  "source_reliability": "high",
  "confidence": 0.85
}
```

**Advantages**: Handles complex, multi-sentence event descriptions; flexible schema; works across languages.
**Disadvantages**: API cost; latency; potential hallucination of event details.

#### Approach B: Fine-Tuned Transformer Event Extraction

Fine-tune a transformer model on conflict event extraction datasets (ACE 2005, ERE, CASE workshop data):

- **T-SEE** (Transformer-based Semantic Event Extraction): Fine-tuned transformer approach that leverages event ontologies for capturing multifaceted events.
- **Instruction-tuned LLMs**: Fine-tune smaller LLMs (Llama 3, Mistral) with annotation guidelines for domain-specific event extraction (ACL Findings 2025).

#### Approach C: RAG-Enhanced Event Extraction

**Political-RAG** framework (2025): Combines Retrieval-Augmented Generation with LLMs for extracting political event information from media content including Twitter and news articles. Enhances accuracy by incorporating external data retrieval for context.

### Event Schema Design

Based on ACLED methodology and conflict studies best practices, each extracted event should contain:
- **Event Type** (mapped to ACLED taxonomy)
- **Sub-Event Type** (25 ACLED sub-types)
- **Date/Time** (ISO 8601)
- **Location** (text + resolved coordinates)
- **Actors** (with type classification)
- **Interaction Type** (actor-to-actor relationship)
- **Fatality Estimate** (with confidence range)
- **Source** (with reliability rating)
- **Notes/Description** (free text summary)

---

## 5. Conflict Classification Taxonomy

### Recommended Taxonomy (Based on ACLED Codebook)

#### Level 1: Event Types (6 categories)

1. **Battles** — Armed clashes between organized groups
2. **Violence Against Civilians** — Deliberate targeting of civilians
3. **Explosions/Remote Violence** — Bombings, airstrikes, IEDs, shelling
4. **Riots** — Violent demonstrations and mob violence
5. **Protests** — Non-violent public demonstrations
6. **Strategic Developments** — Agreements, arrests, troop movements

#### Level 2: Sub-Event Types (25 categories)

Under **Battles**: Armed clash, Government regains territory, Non-state actor overtakes territory
Under **Violence Against Civilians**: Sexual violence, Attack, Abduction/forced disappearance
Under **Explosions/Remote Violence**: Chemical weapon, Air/drone strike, Suicide bomb, Shelling/artillery/missile attack, Remote explosive/landmine/IED, Grenade
Under **Riots**: Violent demonstration, Mob violence
Under **Protests**: Peaceful protest, Protest with intervention, Excessive force against protesters
Under **Strategic Developments**: Agreement, Arrests, Change to group/activity, Disrupted weapons use, Headquarters or base established, Looting/property destruction, Non-violent transfer of territory, Other

#### Level 3: Conflict Categories (5 meta-categories, new ACLED 2024+)

1. **Repression** — State violence against civilians and protesters
2. **Insurgency** — All events involving rebel group activity
3. **Atrocities** — Violence targeting civilians with 10+ reported fatalities
4. **Terrorism** — Systematic civilian targeting by armed groups (behavior-based, not list-based)
5. **Foreign Military Engagement** — Cross-border state violence (invasions, airstrikes, border clashes)

#### Severity Scoring

Adopt ACLED's Conflict Index methodology with four indicators:
- **Deadliness**: Fatality counts (weighted by recency)
- **Danger to Civilians**: Proportion of civilian-targeting events
- **Geographic Diffusion**: Spatial spread of conflict events
- **Armed Group Proliferation**: Number of distinct armed actors

### Implementation

Use a multi-label classifier fine-tuned on ACLED-coded data:
- **Model**: `xlm-roberta-large` fine-tuned for multi-label classification
- **Training Data**: ACLED event descriptions mapped to event types/sub-types
- **Evaluation**: Stratified cross-validation, macro-F1 as primary metric

---

## 6. Geoparsing & Geocoding Strategy

### Primary Tool: Mordecai 3

**Mordecai 3** (by Andrew Halterman, 2023) is the recommended geoparsing library for conflict event data. It is specifically designed for geoparsing news text and event geocoding.

#### Architecture

1. **Place Name Extraction**: Uses spaCy NER to identify location mentions in text.
2. **Candidate Generation**: Queries a local Geonames gazetteer (via Elasticsearch) to find candidate coordinates.
3. **Neural Ranking**: A trained neural model selects the best match using:
   - Features from the place name itself
   - Other places mentioned in the same text (context)
   - Document content embeddings
   - Elasticsearch query features
4. **Event Geocoding**: Uses a question-answering model to link events to the specific locations where they occurred (distinct from merely mentioned locations).

#### Installation & Dependencies

```bash
pip install mordecai3
# Requires local Elasticsearch + Geonames index
docker pull elasticsearch:7.10.1
# Load Geonames data into Elasticsearch
```

#### Strengths

- Designed specifically for conflict/political event geocoding
- Better handling of non-US place names than alternatives
- Neural context model resolves ambiguous toponyms (e.g., "Paris, France" vs "Paris, Texas")
- Event-specific geocoding (where did it happen, not just where is mentioned)

#### Limitations

- Primarily optimized for English-language text
- Requires local Elasticsearch infrastructure
- Limited support for colloquial place references and low-resource languages

### Supplementary Geoparsing Approaches

| Tool | Strengths | Use Case |
|------|-----------|----------|
| **Mordecai 3** | Conflict-specific, event geocoding | Primary geoparser |
| **spaCy NER (GPE/LOC)** | Fast, multilingual | Initial location extraction |
| **Google Geocoding API** | High accuracy, global coverage | Fallback for unresolved locations |
| **Nominatim (OpenStreetMap)** | Free, self-hostable | Bulk geocoding, offline use |
| **GeoNames API** | Comprehensive gazetteer | Entity resolution supplement |

### Geoparsing Pipeline

```
Text Input
    |
    v
[spaCy NER] --> Extract GPE, LOC, FAC entities
    |
    v
[Mordecai 3] --> Resolve to Geonames entries with coordinates
    |
    v
[Fallback: Nominatim/Google] --> For unresolved entities
    |
    v
[Confidence Scoring] --> Score each resolution (high/medium/low)
    |
    v
[Admin Level Assignment] --> Country > Admin1 > Admin2 > City
    |
    v
Output: {name, lat, lon, geonames_id, admin_level, confidence}
```

---

## 7. Entity Resolution & Knowledge Graph Construction

### Entity Resolution Strategy

Entity resolution links mentions across multiple sources to the same real-world entity. For conflict data, this is critical for tracking actors (armed groups, political figures, military units) across news sources.

#### Approach

1. **Candidate Generation**: For each extracted entity, generate candidates from a reference knowledge base (Wikidata, custom conflict actor database).
2. **Entity Linking**: Use LLM-based entity linking for disambiguation:
   - **ReFinED** (Amazon): Uses fine-grained entity types and descriptions for efficient end-to-end entity linking.
   - **ChatEL**: Three-step LLM framework — generate candidates, enhance context, multiple-choice selection.
3. **Coreference Resolution**: Use spaCy's neural coreference model or `neuralcoref` to resolve within-document references.
4. **Cross-Document Entity Linking**: Match entities across documents using embedding similarity (sentence-transformers) + string matching heuristics.

### Knowledge Graph Construction

#### Architecture

Use **Neo4j** as the graph database with an LLM-powered extraction pipeline:

**Node Types:**
- `Actor` (Person, Organization, Military Unit, Armed Group)
- `Location` (Country, Region, City, Facility)
- `Event` (Conflict Event, Political Event, Economic Event)
- `Weapon` (Weapon System, Munition Type)
- `Source` (News Article, Report, Social Media Post)

**Edge Types:**
- `PARTICIPATED_IN` (Actor -> Event)
- `OCCURRED_AT` (Event -> Location)
- `AFFILIATED_WITH` (Actor -> Actor/Organization)
- `TARGETED` (Actor -> Actor/Location)
- `REPORTED_BY` (Event -> Source)
- `USED_WEAPON` (Actor -> Weapon, in Event)
- `ALLIED_WITH` / `OPPOSED_TO` (Actor -> Actor)

#### LLM-Powered Extraction (2025 Best Practice)

Based on 2025 research, use LLMs for knowledge graph construction:
- Achievable precision: ~89.7%, recall: ~92.3% for entity extraction
- Use schema-guided extraction (KARMA framework approach) with multi-agent architecture
- Support dynamic schema evolution as new conflict actors and relationships emerge
- Consider **GraphRAG** or **LightRAG** for retrieval-augmented generation over the constructed knowledge graph (LightRAG achieves comparable accuracy with 10x token reduction vs. GraphRAG)

---

## 8. Predictive Analytics — Feasibility Assessment

### State of the Art: VIEWS System

The **VIEWS (Violence & Impacts Early-Warning System)** represents the gold standard for conflict prediction:

- **Scope**: Global forecasts at country-month level; sub-national (0.5 degree grid) for Africa and Middle East
- **Horizon**: 1-36 months ahead
- **Data Sources**: 12+ trusted providers, hundreds of features (conflict history, economic growth, political stability, natural resources, terrain, climate vulnerability)
- **Track Record**: Correctly identified 7/10 deadliest countries in 2024; 6/10 in 2023
- **Open Source**: Data and methodology publicly available

#### VIEWS Model Zoo (2025)

| Model | Type | Performance |
|-------|------|-------------|
| **Conflictology** (VIEWS benchmark) | Ensemble | Leading in 2023/24 Challenge |
| **Observed Markov Model** (Randahl & Vegelius) | Statistical | 2nd at country level |
| **Bayesian Negative Binomial GLMM** (Brandt) | Bayesian | 3rd at country level |
| **Forests of UncertainT(r)ees** (CCEW) | Random Forest ensemble | Top contender at sub-national level |
| **HydraNet** | Neural network | New, in development |
| **XGBoost** | Gradient boosting | Strong baseline in open-source reproduction |
| **AutoGluon** | AutoML | Competitive, easy to deploy |
| **TabPFN** | Meta-learned transformer | Novel approach, evaluated in 2025 thesis |

### Recommended Approach for Our System

#### Phase 1: Statistical Baseline (Months 1-3)
- Implement **XGBoost** model trained on UCDP GED + ACLED data
- Features: conflict history (lagged events, fatalities), economic indicators (GDP, inflation), political stability indices, population density, terrain features
- Prediction target: Monthly event counts and fatality estimates at country and sub-national level
- Use VIEWS open-source framework as reference implementation

#### Phase 2: Deep Learning Models (Months 3-6)
- **LSTM/GRU** time-series models for capturing temporal dynamics
- **Temporal Graph Neural Networks** for modeling spatial spillover effects
- **Transformer-based** sequence models for long-horizon forecasting

#### Phase 3: LLM-Enhanced Forecasting (Months 6-12)
- Integrate text-derived features from news NLP pipeline
- Use LLMs for narrative-based risk assessment
- Combine quantitative predictions with qualitative LLM analysis

### Key Challenges (from Literature)

1. **Onset Prediction is Hard**: Predicting when a new conflict begins is significantly harder than predicting continuation of existing conflicts.
2. **Spatio-Temporal Dominance**: Models heavily rely on "where violence happened before, it will happen again" — genuine predictive power for new locations is limited.
3. **Data Uncertainty**: Human-coded conflict data contains inherent biases, inconsistencies, and missing observations.
4. **Ethical Concerns**: Risk of surveillance misuse, algorithmic bias, and automation bias in decision-making.

### Feasibility Assessment

| Capability | Feasibility | Confidence | Timeline |
|-----------|-------------|------------|----------|
| Continuation forecasting (ongoing conflicts) | **HIGH** | 85% | 3 months |
| Escalation/de-escalation detection | **MEDIUM** | 65% | 6 months |
| New conflict onset prediction | **LOW-MEDIUM** | 40% | 12+ months |
| Fatality estimation | **MEDIUM** | 60% | 6 months |
| Geographic spread prediction | **MEDIUM** | 55% | 6 months |

---

## 9. Real-Time Alert System Design

### Anomaly Detection Architecture

Based on research from the Italian Ministry of Foreign Affairs collaboration (ScienceDirect 2024) and the strategic early warning system using GDELT data:

#### Detection Methods

1. **Structural Break Detection**: Identify long-term shifts in conflict patterns (regime changes, war onset/termination)
   - Method: CUSUM, Bayesian change-point detection
   - Window: Rolling 30-90 day analysis
   - Example: Detected Ukraine tensions one month before Russia's 2022 invasion

2. **Additive Outlier Detection**: Identify sudden, temporary spikes in event counts
   - Method: Z-score based, IQR-based, or Isolation Forest
   - Window: Rolling 7-14 day analysis
   - Threshold: Events exceeding 2.5 standard deviations from moving average

3. **Trend Acceleration**: Detect when conflict intensity is increasing faster than historical norms
   - Method: First/second derivative analysis of event time series
   - Window: Rolling 14-30 day analysis

4. **Spatial Clustering Anomalies**: Detect new geographic clusters of violence
   - Method: DBSCAN or HDBSCAN on geocoded events
   - Alert when new clusters emerge outside historical conflict zones

#### Alert Pipeline

```
Event Stream (ACLED, GDELT, News NLP)
    |
    v
[Time-Series Aggregation] --> Country-day, Grid-week event counts
    |
    v
[Parallel Anomaly Detectors]
    |-- Structural Break Detector
    |-- Spike Detector (Isolation Forest)
    |-- Trend Acceleration Detector
    |-- Spatial Anomaly Detector
    |
    v
[Alert Fusion & Deduplication]
    |
    v
[Priority Scoring]
    |-- P1 (Critical): Multiple detectors fire + high fatality
    |-- P2 (High): Single detector + conflict zone
    |-- P3 (Medium): Single detector + non-conflict zone
    |-- P4 (Low): Minor anomaly, informational
    |
    v
[False Positive Filtering]
    |-- Cross-reference with known events (holidays, elections)
    |-- Require minimum source corroboration (2+ sources)
    |-- Suppress duplicate alerts within cooldown window
    |
    v
[Alert Distribution]
    |-- WebSocket push to dashboard
    |-- Email/Slack notifications for P1/P2
    |-- Daily digest for P3/P4
```

#### Reducing False Positives

- **Source Corroboration**: Require events detected in 2+ independent sources before alerting
- **Event Deduplication**: Use entity resolution to merge duplicate event reports
- **Contextual Filtering**: Suppress alerts during expected high-activity periods (elections, religious holidays, seasonal patterns)
- **User Feedback Loop**: Allow analysts to mark false positives, feed back into model training
- **Cooldown Windows**: Suppress repeat alerts for same location/event type within configurable time window

### Reference Systems

| System | Organization | Approach |
|--------|-------------|----------|
| **VIEWS Early Warning** | Uppsala/PRIO | ML-based monthly forecasts |
| **ICEWS** | Lockheed Martin (DARPA) | Automated event coding + forecasting, 300+ event types, >90% claimed accuracy |
| **GDELT** | Google/Georgetown | Real-time global event monitoring |
| **ACLED CAST** | ACLED | Conflict analysis and scenario tools |

---

## 10. LLM Integration Strategy for Summarization

### Architecture

```
Conflict Events & News Articles (daily intake)
    |
    v
[Clustering] --> Group related events by conflict/region
    |
    v
[Context Assembly] --> Gather all relevant events, entities, and historical context
    |
    v
[LLM Summarization (Claude API)]
    |-- Daily SITREP per active conflict zone
    |-- Weekly intelligence briefing (all regions)
    |-- Flash reports for P1/P2 alerts
    |-- Monthly trend analysis
    |
    v
[Human Review & Distribution]
```

### Claude API Integration

#### Model Selection

| Task | Recommended Model | Rationale |
|------|-------------------|-----------|
| Daily SITREP generation | Claude Sonnet 4.5 | Good accuracy, cost-efficient for daily volume |
| Flash alert briefings | Claude Haiku 4.5 | Low latency, sufficient for structured summaries |
| Weekly intelligence briefing | Claude Opus 4 / Opus 4.6 | Complex multi-document reasoning, highest quality |
| Ad-hoc analysis requests | Claude Opus 4.6 | Best reasoning for complex geopolitical questions |

#### Summarization Strategy

**For documents within context window (< 200K tokens):**
- Single-pass summarization with structured output prompting
- Use system prompts that define SITREP format, required sections, and analytical standards

**For large document sets exceeding context window:**
- **Map-Reduce**: Chunk documents into segments, summarize each chunk, then synthesize chunk summaries into a final briefing
- **Hierarchical**: Summarize individual articles -> summarize per-conflict -> summarize per-region -> generate global briefing

#### SITREP Template (for Claude prompting)

```
SITUATION REPORT — [REGION/CONFLICT] — [DATE]

1. SITUATION OVERVIEW
   [2-3 sentence executive summary]

2. KEY DEVELOPMENTS (last 24 hours)
   - [Development 1 with location, actors, outcome]
   - [Development 2...]

3. CASUALTY & DISPLACEMENT UPDATE
   [Reported figures with confidence levels]

4. ACTOR ANALYSIS
   [Key actor movements, statements, capability changes]

5. TREND ASSESSMENT
   [Escalation/de-escalation indicators, comparison to prior period]

6. OUTLOOK
   [Short-term forecast: 24-72 hours]

7. SOURCES
   [List of sources with reliability ratings]
```

#### Cost Estimation

Assuming 500 news articles/day, average 2,000 tokens each:
- NER/Classification (self-hosted): ~$0/day (GPU cost only)
- Event Extraction via Claude Sonnet (500 calls/day): ~$7.50/day
- Daily SITREPs (10 conflict zones): ~$1.50/day
- Weekly briefing (1 call): ~$0.50/week
- **Total estimated LLM API cost**: ~$250-300/month

---

## 11. Training Data Sources & Requirements

### Available Datasets

| Dataset | Description | Size | Access |
|---------|-------------|------|--------|
| **ACLED** | Conflict events, global, 1997-present | 1M+ events | API (free for research) |
| **UCDP GED** | Georeferenced event data, 1989-present | 300K+ events | Open access |
| **GDELT** | Global event database, real-time | Billions of events | Open access (BigQuery) |
| **ICEWS** | Coded international events | Millions of events | Dataverse |
| **ACE 2005** | NLP event extraction benchmark | 599 documents | LDC license |
| **ERE (Entities, Relations, Events)** | NLP event extraction data | ~1,000 documents | LDC license |
| **CASE Workshop Data** | Socio-political event extraction shared tasks | Varies by year | Workshop proceedings |
| **Kobza** | Ukrainian text corpus | 1.3TB / 60B tokens | Open access |
| **Correlates of War** | Conflict datasets, historical | 200+ years | Open access |

### Annotation Requirements

For fine-tuning custom models:
- **NER Model**: 2,000-5,000 annotated sentences (custom conflict entities)
- **Event Classifier**: 10,000+ labeled event descriptions (ACLED data can serve as training data)
- **Severity Scorer**: 5,000+ events with severity annotations
- **Geoparsing**: Mordecai 3 provides pre-trained models; additional annotation needed only for non-English languages

### Annotation Tools

- **Prodigy** (by Explosion AI / spaCy team): Active learning annotation, integrates directly with spaCy training
- **Doccano**: Open-source annotation tool, used in Omdena conflict NER project
- **Label Studio**: Open-source, supports NER, classification, and relation annotation

---

## 12. Implementation Roadmap

### Phase 1: Foundation (Months 1-3)
- Deploy spaCy transformer pipeline for English NER
- Integrate Mordecai 3 for geoparsing
- Set up ACLED/GDELT data ingestion
- Build basic event classification using ACLED taxonomy
- Implement simple anomaly detection (Z-score based)

### Phase 2: Intelligence Layer (Months 3-6)
- Fine-tune XLM-RoBERTa for multilingual NER (Arabic, Russian, Ukrainian)
- Implement LLM-based event extraction (Claude API)
- Build Neo4j knowledge graph with entity resolution
- Deploy Claude-powered SITREP generation
- Implement XGBoost prediction baseline

### Phase 3: Advanced Analytics (Months 6-12)
- Train deep learning forecasting models (LSTM, temporal GNNs)
- Implement full alert system with priority scoring
- Build GraphRAG for analytical Q&A over knowledge graph
- Add LLM-enhanced forecasting narratives
- Cross-lingual entity resolution across all target languages

### Phase 4: Production Hardening (Months 12-18)
- Performance optimization and scaling
- Analyst feedback integration
- False positive reduction via active learning
- Dashboard integration and API hardening
- Model monitoring and drift detection

---

## 13. Technology Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| NER | spaCy + `xlm-roberta-large` | Production-ready, multilingual |
| Event Extraction | Claude API (Sonnet 4.5) | Flexible, high-quality structured output |
| Geoparsing | Mordecai 3 + Elasticsearch | Conflict-specific, neural ranking |
| Translation | M2M-100 / OPUS-MT | Open-source, 100+ languages |
| Knowledge Graph | Neo4j | Industry standard, graph queries |
| Entity Linking | ReFinED + LLM fallback | Efficient, disambiguating |
| Classification | Fine-tuned XLM-RoBERTa | Multi-label, multilingual |
| Anomaly Detection | Isolation Forest + CUSUM | Complementary detection methods |
| Prediction | XGBoost -> LSTM -> Temporal GNN | Phased complexity increase |
| Summarization | Claude API (Opus/Sonnet) | Best-in-class reasoning |
| Annotation | Prodigy / Doccano | Efficient labeling workflows |
| Orchestration | Apache Airflow / Prefect | DAG-based pipeline management |
| Message Queue | Apache Kafka / Redis Streams | Real-time event streaming |
| Vector Store | Pinecone / Weaviate | Embedding-based retrieval |

---

## References & Sources

### NER and NLP
- [Modern NER: Beyond Traditional NLP with Transformers and LLMs (2026)](https://akankshaonearth.medium.com/modern-named-entity-recognition-beyond-traditional-nlp-with-transformers-and-llms-2026-c935ef31e692)
- [spaCy NER for Actors in News Articles — Omdena](https://www.omdena.com/blog/spacy-named-entity-recognition)
- [Evaluating NER Models for Russian Cultural News Texts (arXiv 2025)](https://arxiv.org/html/2506.02589v1)
- [Complete Guide to NER (2025)](https://nanonets.com/blog/named-entity-recognition-with-nltk-and-spacy/)
- [spaCy Official Site](https://spacy.io/)

### Event Extraction
- [Event Extraction in LLMs: A Holistic Survey (arXiv 2025)](https://arxiv.org/html/2512.19537v1)
- [Transformer vs LLM Semantic Event Extraction (SAGE 2025)](https://journals.sagepub.com/doi/full/10.1177/22104968251363759)
- [Instruction-Tuning LLMs for Event Extraction (ACL Findings 2025)](https://aclanthology.org/2025.findings-acl.677.pdf)
- [EventExtractionPapers — GitHub](https://github.com/BaptisteBlouin/EventExtractionPapers)

### Multilingual Models
- [Analysis of Multilingual Models on HuggingFace](https://huggingface.co/blog/catherinearnett/hf-model-survey)
- [EuroLLM-9B/22B](https://huggingface.co/blog/eurollm-team/eurollm-22b)
- [Awesome Ukrainian NLP — GitHub](https://github.com/osyvokon/awesome-ukrainian-nlp)

### ACLED Methodology
- [ACLED Codebook](https://acleddata.com/methodology/acled-codebook)
- [ACLED Conflict Categories](https://acleddata.com/methodology/conflict-categories)
- [ACLED Methodology Knowledge Base](https://acleddata.com/conflict-data/knowledge-base/methodology)

### Geoparsing
- [Mordecai 3: Neural Geoparser — Andrew Halterman](https://andrewhalterman.com/post/mordecai3/)
- [Mordecai 3 — GitHub](https://github.com/ahalterman/mordecai3)
- [Mordecai (original) — GitHub](https://github.com/openeventdata/mordecai)

### Conflict Prediction & Early Warning
- [VIEWS Forecasting](https://viewsforecasting.org/)
- [VIEWS — Uppsala University](https://www.uu.se/en/department/peace-and-conflict-research/research/views)
- [VIEWS 2023/24 Prediction Challenge Results](https://journals.sagepub.com/doi/10.1177/00223433241300862)
- [Open-Source AI Framework for Forecasting Armed Conflict (2025)](https://euridice.eu/wp-content/uploads/2025/09/Beyond-Closed-Doors-An-Open-Source-AI-Framework-for-Forecasting-Armed-Conflict-1.pdf)
- [AI model warns of deadliest conflict zones in 2026 — PRIO](https://www.prio.org/news/3670)

### Anomaly Detection & Early Warning
- [Breaking the Trend: Anomaly Detection for Early Warning (ScienceDirect 2024)](https://www.sciencedirect.com/science/article/pii/S0040162524002919)
- [Strategic Early Warning System for Global Crisis (Medium 2025)](https://medium.com/thedeephub/developing-a-strategic-early-warning-system-to-predict-and-monitor-global-crisis-part-ii-a4585f0b5bc3)
- [ICEWS — Lockheed Martin](https://www.lockheedmartin.com/en-us/capabilities/research-labs/advanced-technology-labs/icews.html)
- [AI and Global Security: Early Warning to AI-Assisted Diplomacy (CIGI 2025)](https://www.cigionline.org/static/documents/TF4_De_Agostini_Giovanardi.pdf)

### Knowledge Graphs
- [LLM-Empowered Knowledge Graph Construction Survey (arXiv 2025)](https://arxiv.org/html/2510.20345v1)
- [From LLMs to Knowledge Graphs (Medium 2025)](https://medium.com/@claudiubranzan/from-llms-to-knowledge-graphs-building-production-ready-graph-systems-in-2025-2b4aff1ec99a)
- [Making Sense of News with Knowledge Graphs — Neo4j](https://neo4j.com/blog/developer/making-sense-of-news-the-knowledge-graph-way/)
- [LLM-TEXT2KG 2025 Workshop](https://aiisc.ai/text2kg2025/)

### LLM Summarization
- [Summarization with Claude — Anthropic Cookbook](https://platform.claude.com/cookbook/capabilities-summarization-guide)
- [Legal Summarization — Claude Docs](https://platform.claude.com/docs/en/about-claude/use-case-guides/legal-summarization)
- [Master LLM Summarization Strategies — Galileo](https://galileo.ai/blog/llm-summarization-strategies)

### Conflict Data Sources
- [ACLED](https://acleddata.com/)
- [UCDP](https://ucdp.uu.se/)
- [GDELT Project](https://www.gdeltproject.org/)
- [Anticipation Hub — Conflict Early Warning Catalogue](https://www.anticipation-hub.org/learn/emerging-topics/conflict/catalogue-of-early-warning-tools-for-anticipating-the-impact-of-conflict)
