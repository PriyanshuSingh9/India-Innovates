# Data Schemas

> Central schema reference for all data models used across BIE subsystems. This document is the **single source of truth** — all backend Pydantic models and frontend TypeScript types are derived from these definitions.

---

## 1. API Response Schemas

### Event

The core intelligence unit — a classified, geolocated, NLP-enriched event.

```json
{
  "id": "string — unique event fingerprint (MD5 hash of title + url)",
  "title": "string — event headline",
  "summary": "string — event summary/description",
  "source": "string — source feed name (e.g. Reuters, PTI, SCMP)",
  "source_id": "string — feed registry ID",
  "url": "string — original source URL",
  "category": "enum — MILITARY | DIPLOMATIC | ECONOMIC | INTERNAL | MARITIME | CYBER | TERRORISM",
  "threat_level": "enum — LOW | MEDIUM | HIGH | CRITICAL",
  "location": {
    "lat": "float — latitude",
    "lng": "float — longitude",
    "name": "string — resolved location name",
    "country": "string — ISO 3166-1 alpha-2 country code",
    "region": "string — strategic region (see Region enum)"
  },
  "entities": [
    {
      "text": "string — entity surface text",
      "label": "enum — PERSON | ORG | GPE | LOC | MILITARY_UNIT | WEAPON_SYSTEM | INFRASTRUCTURE | BORDER_LANDMARK | INDIAN_ORG | STRATEGIC_ORG | STRATEGIC_CORRIDOR",
      "start": "int — character offset start",
      "end": "int — character offset end"
    }
  ],
  "classification_confidence": "float — 0.0 to 1.0",
  "india_relevance": "float — 0.0 to 1.0 — how directly this affects Indian interests",
  "geo_focus": "string — strategic region of the source feed",
  "tier": "int — source tier (1=wire service, 4=aggregator)",
  "timestamp": "ISO 8601 datetime — event publication time",
  "ingested_at": "ISO 8601 datetime — BIE ingestion time",
  "processed_at": "ISO 8601 datetime — NLP processing time"
}
```

### FeedItem (raw — pre-NLP)

```json
{
  "id": "string — fingerprint (MD5 of title + url)",
  "source_id": "string — feed registry ID",
  "source_name": "string — human-readable source name",
  "title": "string",
  "summary": "string",
  "url": "string — original article URL",
  "published_at": "ISO 8601 datetime",
  "ingested_at": "ISO 8601 datetime",
  "language": "string — en | hi | ur | ar | zh | fr | ...",
  "tier": "int — source tier (1–4)",
  "geo_focus": ["string — geographic focus regions"],
  "category_hints": ["string — categories from feed registry"]
}
```

### InstabilityScore (BII)

```json
{
  "region": "string — strategic region (see Region enum)",
  "country": "string — optional, ISO country code for country-level scoring",
  "score": "float — 0.0 to 1.0",
  "trend": "enum — RISING | STABLE | FALLING",
  "contributing_factors": ["string — event IDs driving the score"],
  "event_count": "int — events in current 24h window",
  "baseline_event_count": "float — 7-day rolling average",
  "floor_active": "boolean — true if conflict-zone floor is elevating the score",
  "india_proximity_weight": "float — multiplier applied (1.5 for SOUTH_ASIA, 0.6 for AMERICAS)",
  "last_updated": "ISO 8601 datetime"
}
```

### QueryResponse (GraphRAG)

```json
{
  "answer": "string — LLM-generated analysis (streamed)",
  "citations": [
    {
      "event_id": "string",
      "title": "string",
      "relevance": "float"
    }
  ],
  "confidence": "float — 0.0 to 1.0",
  "graph_context": {
    "nodes": ["..."],
    "edges": ["..."]
  },
  "fallback_used": "boolean — true if Qdrant-only fallback was used"
}
```

### StrategicBrief

```json
{
  "id": "string — UUID",
  "generated_at": "ISO 8601 datetime",
  "period_start": "ISO 8601 datetime",
  "period_end": "ISO 8601 datetime",
  "sections": [
    {
      "title": "string — section name (Executive Summary, India Neighborhood Watch, etc.)",
      "content": "string — markdown content",
      "events_referenced": ["string — event IDs"]
    }
  ],
  "instability_scores": {
    "LAC_WEST": 0.75,
    "SOUTH_CHINA_SEA": 0.55,
    "PERSIAN_GULF": 0.42,
    "RED_SEA": 0.68
  },
  "overall_posture": "enum — NORMAL | ELEVATED | HIGH | CRITICAL"
}
```

### Anomaly

```json
{
  "id": "string — UUID",
  "type": "enum — FREQUENCY_SPIKE | CATEGORY_ANOMALY | ENTITY_ANOMALY",
  "region": "string — strategic region",
  "score": "float — deviation from baseline (in σ units)",
  "event_ids": ["string — triggering events"],
  "detected_at": "ISO 8601 datetime",
  "expires_at": "ISO 8601 datetime — 24h TTL"
}
```

### Convergence

```json
{
  "id": "string — UUID",
  "cell": "string — geographic cell ID (lat_lng format)",
  "region": "string — strategic region containing the cell",
  "categories": ["string — converging categories"],
  "event_count": "int",
  "anomaly_linked": "boolean — convergence coincides with active anomaly",
  "severity": "enum — MEDIUM | HIGH",
  "detected_at": "ISO 8601 datetime"
}
```

---

## 2. Enumerations

### Category

```
MILITARY | DIPLOMATIC | ECONOMIC | INTERNAL | MARITIME | CYBER | TERRORISM
```

### ThreatLevel

```
LOW | MEDIUM | HIGH | CRITICAL
```

### Region (Strategic Regions)

India's strategic landscape, organized by proximity:

| Group | Regions |
|---|---|
| **India Borders** | `LAC_WEST`, `LAC_EAST`, `LOC_KASHMIR`, `SIACHEN`, `NORTHEAST` |
| **Indian Ocean** | `IOR` |
| **Indo-Pacific** | `SOUTH_CHINA_SEA`, `TAIWAN_STRAIT`, `SOUTHEAST_ASIA`, `EAST_CHINA_SEA` |
| **Middle East** | `PERSIAN_GULF`, `RED_SEA` |
| **Extended** | `HORN_OF_AFRICA`, `CENTRAL_ASIA`, `EUROPE`, `AMERICAS` |
| **Internal** | `INTERNAL_CENTRAL`, `INTERNAL_SOUTH` |

### GeoFocus (Feed Source Regions)

```
SOUTH_ASIA | INDO_PACIFIC | MIDDLE_EAST | CENTRAL_ASIA | AFRICA | EUROPE | AMERICAS | GLOBAL
```

### EntityLabel

```
PERSON | ORG | GPE | LOC | MILITARY_UNIT | WEAPON_SYSTEM | INFRASTRUCTURE
| BORDER_LANDMARK | INDIAN_ORG | STRATEGIC_ORG | STRATEGIC_CORRIDOR
```

### SourceTier

| Tier | Description | Examples |
|---|---|---|
| 1 | Wire services, official government | PTI, ANI, Reuters, AP, MEA, PIB |
| 2 | Major established outlets | The Hindu, BBC, SCMP, Al Jazeera, Dawn |
| 3 | Specialty/niche | Defense One, ORF, Bharat Shakti, Foreign Policy |
| 4 | Aggregators and blogs | Google News proxies, analyst blogs |

### PropagandaRisk

```
low | medium | high
```

State-controlled sources (Xinhua, TASS, RT, CGTN) are `high`. State-affiliated but editorially independent (Al Jazeera, France 24, DW) are `medium`. Independent (Reuters, AP, BBC, Bellingcat) are `low`.

### Posture

```
NORMAL | ELEVATED | HIGH | CRITICAL
```

---

## 3. Neo4j Graph Schema

### Node Labels & Properties

| Label | Properties | Constraints |
|---|---|---|
| `Person` | name (string), role (string), nationality (string), aliases (string[]) | UNIQUE on name |
| `Organization` | name (string), type (string), country (string), aliases (string[]) | UNIQUE on name |
| `Country` | name (string), code (string), region (string), aliases (string[]) | UNIQUE on code |
| `Location` | name (string), lat (float), lng (float), type (string), region (string), country (string) | UNIQUE on name |
| `Event` | id (string), title (string), summary (string), category (string), threat_level (string), timestamp (datetime), india_relevance (float) | UNIQUE on id |
| `MilitaryUnit` | name (string), branch (string), country (string), parent_command (string) | UNIQUE on name |
| `WeaponSystem` | name (string), type (string), country (string), operator (string) | UNIQUE on name |
| `Infrastructure` | name (string), type (string), lat (float), lng (float), operator (string), country (string) | UNIQUE on name |
| `Alliance` | name (string), type (string), members (string[]), founded (string) | UNIQUE on name |

### Relationship Types

| Type | From → To | Properties |
|---|---|---|
| `MENTIONED_IN` | Entity → Event | role (string), confidence (float) |
| `OPERATES_IN` | Org/Unit → Location | since (date) |
| `DEPLOYED_AT` | MilitaryUnit → Location | since (date), status (string) |
| `ALLIES_WITH` | Country/Org → Country/Org | — |
| `RIVALS_WITH` | Country/Org → Country/Org | — |
| `MEMBER_OF` | Country → Alliance | since (date) |
| `NEAR` | Location → Location | distance_km (float) |
| `PART_OF` | Location → Location | — (hierarchy: city → state → country) |
| `COMMANDS` | Person → Org/Unit | title (string) |
| `OPERATES` | Country/Org → Infrastructure | since (date), capacity (string) |
| `RELATED_TO` | Entity → Entity | co_occurrence_count (int) |

### Indexes

```cypher
CREATE INDEX ON :Person(name)
CREATE INDEX ON :Organization(name)
CREATE INDEX ON :Country(code)
CREATE INDEX ON :Location(name)
CREATE INDEX ON :Location(region)
CREATE INDEX ON :Event(id)
CREATE INDEX ON :Event(timestamp)
CREATE INDEX ON :Event(category)
CREATE INDEX ON :Alliance(name)
CREATE INDEX ON :MilitaryUnit(country)
CREATE INDEX ON :Infrastructure(type)
```

---

## 4. Qdrant Collection Schema

### Collection: `bie_events`

| Field | Type | Description |
|---|---|---|
| **vector** | float[384] | Embedding from `all-MiniLM-L6-v2` |
| **payload.id** | string | Event ID (matches Event.id) |
| **payload.title** | string | Event title |
| **payload.summary** | string | Event summary |
| **payload.category** | string | Event category (7-value enum) |
| **payload.threat_level** | string | Threat level |
| **payload.region** | string | Strategic region |
| **payload.country** | string | ISO country code |
| **payload.geo_focus** | string | Feed source region |
| **payload.india_relevance** | float | India-relevance score |
| **payload.timestamp** | string | ISO 8601 timestamp |
| **payload.entities** | string[] | Extracted entity names |

### Indexes

- Payload index on `category` (keyword)
- Payload index on `threat_level` (keyword)
- Payload index on `region` (keyword)
- Payload index on `country` (keyword)
- Payload index on `india_relevance` (float range)
- Payload index on `timestamp` (datetime range)

---

## 5. Redis Data Structures

### Streams

| Stream | Purpose | Producers | Consumers |
|---|---|---|---|
| `feeds:raw` | Raw ingested items | RSS Worker, API Workers | NLP Worker |
| `feeds:processed` | NLP-enriched items | NLP Worker | Graph Writer, Event API |
| `feeds:dead_letter` | Failed processing items | NLP Worker | Manual review |
| `alerts:convergence` | Convergence alerts | Convergence Detector | Posture Panel (SSE) |
| `events:new` | New events for SSE | Event API | Frontend (SSE) |

### Hashes

| Key Pattern | Purpose |
|---|---|
| `bie:freshness` | Per-source last-seen timestamps |
| `bie:instability:{region}` | Current BII scores per region |
| `bie:instability:country:{code}` | Current BII scores per country |
| `bie:baseline:{region}:{category}:{weekday}` | Welford baseline stats (count, mean, M2) |

### Sets

| Key Pattern | Purpose |
|---|---|
| `bie:seen_hashes` | Dedup fingerprints (48h TTL) |

### Strings (cached responses)

| Key Pattern | TTL | Purpose |
|---|---|---|
| `bie:cache:events:{hash}` | 60s | Event API response cache |
| `bie:cache:instability:{hash}` | 60s | Instability API response cache |
| `bie:cache:llm:{hash}` | 24h | Claude API response cache |
| `bie:cache:brief:latest` | 6h | Latest generated brief |

---

## 6. Feed Registry Schema (YAML)

```yaml
feeds:
  - id: string                    # Unique feed identifier
    name: string                   # Human-readable name
    type: enum                     # rss | api | websocket | scraper
    url: string                    # Feed URL
    poll_interval_seconds: int     # Polling frequency
    category: [string]             # Expected categories (MILITARY, DIPLOMATIC, etc.)
    geo_focus: [string]            # Geographic focus regions (SOUTH_ASIA, INDO_PACIFIC, etc.)
    language: string               # en | hi | ur | ar | zh | fr | ...
    tier: int                      # Source tier (1–4)
    reliability: float             # 0.0 to 1.0
    propaganda_risk: enum          # low | medium | high
    api_key_env: string            # Optional env var name for API key
    rate_limit: int                # Optional max requests per minute
```

---

## 7. Global Strategic Hotspot Schema (JSON)

```json
{
  "id": "string — unique identifier (snake_case)",
  "name": "string — primary name",
  "aliases": ["string — alternate names, including Devanagari/Chinese/Arabic"],
  "lat": "float — latitude",
  "lng": "float — longitude",
  "region": "string — strategic region (see Region enum)",
  "type": "enum — BORDER_LANDMARK | MILITARY_BASE | NAVAL_BASE | AIRFIELD | PORT | CITY | PASS | GLACIER | STRAIT | CHOKEPOINT | BRI_PROJECT | NUCLEAR | RADAR",
  "tier": "int — proximity tier (1=India borders, 2=primary interest, 3=extended)",
  "baseline_threat": "enum — LOW | MEDIUM | HIGH",
  "country": "string — ISO country code, or 'DISPUTED' for contested areas",
  "operator": "string — optional, entity operating the infrastructure (e.g. 'CN' for Chinese-operated ports)",
  "description": "string — brief strategic significance for Indian interests"
}
```

---

← Back to [prd.md](./prd.md)
