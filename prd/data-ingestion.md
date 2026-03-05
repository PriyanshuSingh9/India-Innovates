# Data Ingestion

> Covers: F08 BIE Feed Registry · F09 RSS Ingestion Worker · F10 External API Workers · F37 Data Freshness Monitor

---

## F08 — BIE Feed Registry

### What Is Being Built

A centralized, YAML-based registry of all data sources BIE monitors — spanning global RSS feeds, Indian government sources, international wire services, regional outlets, think tanks, and live APIs. Each entry specifies the source URL, polling interval, expected categories, geographic relevance to India, language, source tier, and reliability rating.

### Why It Is Needed

BIE monitors global events for Indian strategic personnel. This means ingesting from 100+ sources across multiple regions — South Asia, Indo-Pacific, Middle East, Central Asia, Africa, and beyond. Without a centralized registry:
- No way to know which feeds are active
- Polling intervals are inconsistent
- New sources can't be added without code changes
- No visibility into source reliability or geographic focus

### What Use It Provides

- **Single source of truth** for all data inputs across global regions
- **Hot-reloadable** — add new sources by editing YAML, no restart
- **Metadata** — each feed carries tier rating, propaganda risk, language, and India-relevance tags
- **Geographic prioritization** — feeds tagged by strategic region (SOUTH_ASIA, INDO_PACIFIC, MIDDLE_EAST, etc.)
- **Source tiering** — Tier 1 (wire services) through Tier 4 (aggregators) for alert-gating

### How It Is Built

1. **Registry file** — `backend/config/feeds.yaml`:
   ```yaml
   feeds:
     # ----- SOUTH ASIA (Tier 1 — India Sources) -----
     - id: pti_national
       name: "PTI (Press Trust of India)"
       type: rss
       url: "https://news.google.com/rss/search?q=site:ptinews.com&hl=en&gl=IN&ceid=IN:en"
       poll_interval_seconds: 120
       category: [MILITARY, DIPLOMATIC, INTERNAL]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 1
       reliability: 0.95
       propaganda_risk: low

     - id: ani_breaking
       name: "ANI News"
       type: rss
       url: "https://news.google.com/rss/search?q=site:aninews.in&hl=en&gl=IN&ceid=IN:en"
       poll_interval_seconds: 120
       category: [MILITARY, DIPLOMATIC, INTERNAL]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 1
       reliability: 0.90
       propaganda_risk: low

     - id: mea_india
       name: "Ministry of External Affairs"
       type: rss
       url: "https://www.mea.gov.in/rss.xml"
       poll_interval_seconds: 300
       category: [DIPLOMATIC]
       geo_focus: [SOUTH_ASIA, GLOBAL]
       language: en
       tier: 1
       reliability: 1.0
       propaganda_risk: low

     - id: hindu_national
       name: "The Hindu (National)"
       type: rss
       url: "https://www.thehindu.com/news/national/feeder/default.rss"
       poll_interval_seconds: 180
       category: [INTERNAL, DIPLOMATIC, MILITARY]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 2
       reliability: 0.90
       propaganda_risk: low

     - id: hindustan_times
       name: "Hindustan Times"
       type: rss
       url: "https://www.hindustantimes.com/feeds/rss/india-news/rssfeed.xml"
       poll_interval_seconds: 180
       category: [INTERNAL, MILITARY, DIPLOMATIC]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 2
       reliability: 0.85
       propaganda_risk: low

     # ----- SOUTH ASIA (Neighbors) -----
     - id: dawn_pakistan
       name: "Dawn (Pakistan)"
       type: rss
       url: "https://news.google.com/rss/search?q=site:dawn.com&hl=en"
       poll_interval_seconds: 300
       category: [MILITARY, DIPLOMATIC, TERRORISM]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 2
       reliability: 0.80
       propaganda_risk: medium

     # ----- INDO-PACIFIC -----
     - id: scmp_asia
       name: "South China Morning Post"
       type: rss
       url: "https://www.scmp.com/rss/91/feed/"
       poll_interval_seconds: 300
       category: [MILITARY, DIPLOMATIC, ECONOMIC]
       geo_focus: [INDO_PACIFIC]
       language: en
       tier: 2
       reliability: 0.80
       propaganda_risk: medium

     - id: nikkei_asia
       name: "Nikkei Asia"
       type: rss
       url: "https://news.google.com/rss/search?q=site:asia.nikkei.com"
       poll_interval_seconds: 300
       category: [ECONOMIC, DIPLOMATIC]
       geo_focus: [INDO_PACIFIC]
       language: en
       tier: 2
       reliability: 0.85
       propaganda_risk: low

     # ----- MIDDLE EAST -----
     - id: aljazeera_world
       name: "Al Jazeera"
       type: rss
       url: "https://www.aljazeera.com/xml/rss/all.xml"
       poll_interval_seconds: 300
       category: [MILITARY, DIPLOMATIC, TERRORISM]
       geo_focus: [MIDDLE_EAST]
       language: en
       tier: 2
       reliability: 0.80
       propaganda_risk: medium

     # ----- GLOBAL WIRE SERVICES -----
     - id: reuters_world
       name: "Reuters World"
       type: rss
       url: "https://news.google.com/rss/search?q=site:reuters.com+world"
       poll_interval_seconds: 120
       category: [MILITARY, DIPLOMATIC, ECONOMIC, MARITIME]
       geo_focus: [GLOBAL]
       language: en
       tier: 1
       reliability: 0.95
       propaganda_risk: low

     - id: bbc_world
       name: "BBC World"
       type: rss
       url: "https://feeds.bbci.co.uk/news/world/rss.xml"
       poll_interval_seconds: 180
       category: [MILITARY, DIPLOMATIC, INTERNAL]
       geo_focus: [GLOBAL]
       language: en
       tier: 2
       reliability: 0.90
       propaganda_risk: low

     # ----- DEFENCE & SECURITY -----
     - id: defense_one
       name: "Defense One"
       type: rss
       url: "https://www.defenseone.com/rss/all/"
       poll_interval_seconds: 600
       category: [MILITARY]
       geo_focus: [GLOBAL]
       language: en
       tier: 3
       reliability: 0.85
       propaganda_risk: low

     - id: bharat_shakti
       name: "Bharat Shakti"
       type: rss
       url: "https://bharatshakti.in/feed/"
       poll_interval_seconds: 600
       category: [MILITARY]
       geo_focus: [SOUTH_ASIA]
       language: en
       tier: 3
       reliability: 0.75
       propaganda_risk: low

     # ----- THINK TANKS -----
     - id: orf_india
       name: "Observer Research Foundation"
       type: rss
       url: "https://www.orfonline.org/feed"
       poll_interval_seconds: 900
       category: [DIPLOMATIC, MILITARY, ECONOMIC]
       geo_focus: [SOUTH_ASIA, INDO_PACIFIC]
       language: en
       tier: 3
       reliability: 0.85
       propaganda_risk: low

     - id: foreign_policy
       name: "Foreign Policy"
       type: rss
       url: "https://foreignpolicy.com/feed/"
       poll_interval_seconds: 900
       category: [DIPLOMATIC, MILITARY]
       geo_focus: [GLOBAL]
       language: en
       tier: 3
       reliability: 0.85
       propaganda_risk: low

     # ----- HINDI-LANGUAGE -----
     - id: bbc_hindi
       name: "BBC Hindi"
       type: rss
       url: "https://www.bbc.com/hindi/index.xml"
       poll_interval_seconds: 300
       category: [INTERNAL, DIPLOMATIC, MILITARY]
       geo_focus: [SOUTH_ASIA]
       language: hi
       tier: 2
       reliability: 0.85
       propaganda_risk: low

     - id: navbharat_times
       name: "Navbharat Times"
       type: rss
       url: "https://navbharattimes.indiatimes.com/rssfeedsdefault.cms"
       poll_interval_seconds: 300
       category: [INTERNAL]
       geo_focus: [SOUTH_ASIA]
       language: hi
       tier: 2
       reliability: 0.75
       propaganda_risk: low
   ```

2. **Geographic focus regions**: `SOUTH_ASIA`, `INDO_PACIFIC`, `MIDDLE_EAST`, `CENTRAL_ASIA`, `AFRICA`, `EUROPE`, `AMERICAS`, `GLOBAL`

3. **Source tier system** (inspired by World Monitor):
   | Tier | Description | Examples |
   |---|---|---|
   | 1 | Wire services, official government | PTI, ANI, Reuters, AP, MEA, PIB |
   | 2 | Major established outlets | The Hindu, BBC, SCMP, Al Jazeera, Dawn |
   | 3 | Specialty/niche | Defense One, ORF, Bharat Shakti, Foreign Policy |
   | 4 | Aggregators and blogs | Google News proxies, individual analysts |

4. **Full feed reference**: see [feeds/INDIA_FEEDS.md](./feeds/INDIA_FEEDS.md) for India-specific sources and [feeds/ALL_FEEDS.md](./feeds/ALL_FEEDS.md) for the complete global reference.

---

## F09 — RSS Ingestion Worker

### What Is Being Built

An async Python worker that continuously polls all RSS feeds registered in the BIE Feed Registry (F08), deduplicates incoming items, and publishes raw feed items to a Redis Stream for downstream NLP processing.

### Why It Is Needed

BIE's intelligence pipeline starts with data. The RSS worker is the first stage — it converts the outside world's news feeds into a normalized stream of items. Without it, there is no data flowing through the system.

With global scope, the worker must handle 100+ feeds across multiple languages, time zones, and formats — while respecting rate limits and handling failures gracefully.

### What Use It Provides

- **Continuous ingestion** from 100+ global RSS feeds
- **Deduplication** — same article published across AFP, PTI, and Reuters is ingested once
- **Normalization** — RSS and Atom formats unified into a single schema
- **Backpressure** — if the NLP worker is overwhelmed, the RSS worker backs off
- **Per-feed circuit breakers** — a broken feed doesn't take down the entire ingestion pipeline
- **Priority processing** — Tier 1 and India-adjacent feeds are polled more frequently

### How It Is Built

1. **Worker process** — `backend/app/workers/rss_worker.py`:
   ```python
   import aiohttp, feedparser, hashlib, asyncio, yaml

   class RSSWorker:
       def __init__(self, redis, config_path="config/feeds.yaml"):
           self.redis = redis
           self.feeds = self._load_feeds(config_path)
           self.circuit_breakers = {}  # per-feed failure tracking

       async def run(self):
           while True:
               tasks = []
               for feed in self.feeds:
                   if feed["type"] == "rss" and not self._is_circuit_open(feed["id"]):
                       tasks.append(self._poll_feed(feed))
               await asyncio.gather(*tasks, return_exceptions=True)
               await asyncio.sleep(60)  # base cycle interval

       async def _poll_feed(self, feed):
           try:
               async with aiohttp.ClientSession() as session:
                   async with session.get(feed["url"], timeout=aiohttp.ClientTimeout(total=10)) as resp:
                       text = await resp.text()
               parsed = feedparser.parse(text)
               for entry in parsed.entries[:20]:  # cap per feed
                   fingerprint = hashlib.md5(
                       (entry.get("title", "") + entry.get("link", "")).encode()
                   ).hexdigest()
                   if not await self.redis.sismember("bie:seen_hashes", fingerprint):
                       await self.redis.sadd("bie:seen_hashes", fingerprint)
                       await self.redis.expire("bie:seen_hashes", 172800)  # 48h TTL
                       item = {
                           "id": fingerprint,
                           "source_id": feed["id"],
                           "source_name": feed["name"],
                           "title": entry.get("title", ""),
                           "summary": entry.get("summary", ""),
                           "url": entry.get("link", ""),
                           "published_at": entry.get("published", ""),
                           "language": feed.get("language", "en"),
                           "tier": feed.get("tier", 4),
                           "geo_focus": feed.get("geo_focus", ["GLOBAL"]),
                           "category_hints": feed.get("category", []),
                       }
                       await self.redis.xadd("feeds:raw", item)
               self._reset_circuit(feed["id"])
           except Exception as e:
               self._record_failure(feed["id"])
               logging.warning(f"Feed {feed['id']} failed: {e}")
   ```

2. **Deduplication**: MD5 fingerprint of `title + url`, checked against a Redis SET with 48-hour TTL. This catches cross-publication duplicates (PTI wire picked up by 10 outlets).

3. **Per-feed circuit breaker**: after 2 consecutive failures, the feed enters a 5-minute cooldown (inspired by World Monitor's resilience patterns):
   ```python
   def _is_circuit_open(self, feed_id):
       cb = self.circuit_breakers.get(feed_id, {"failures": 0, "cooldown_until": 0})
       if cb["failures"] >= 2 and time.time() < cb["cooldown_until"]:
           return True
       return False

   def _record_failure(self, feed_id):
       cb = self.circuit_breakers.setdefault(feed_id, {"failures": 0, "cooldown_until": 0})
       cb["failures"] += 1
       if cb["failures"] >= 2:
           cb["cooldown_until"] = time.time() + 300  # 5-minute cooldown
   ```

4. **Run as**: dedicated process via `python -m app.workers.rss_worker`. Can scale horizontally by partitioning feeds across worker instances.

---

## F10 — External API Workers

### What Is Being Built

Dedicated worker processes that ingest data from non-RSS sources — live APIs, WebSocket streams, and structured data providers. Each API source has its own worker because each has unique authentication, rate limiting, data format, and failure modes.

### Why It Is Needed

RSS feeds cover news, but not real-time situational data. BIE needs live data on:
- **Vessel positions** (AIS) — who is sailing where in the Indian Ocean, South China Sea, Strait of Malacca
- **Military aircraft** (ADS-B) — unusual flight patterns near the LAC or in contested airspaces
- **Conflict events** (ACLED/GDELT) — structured conflict and protest data across regions of interest
- **Natural disasters** (USGS earthquakes, NASA FIRMS fires) — events that drive humanitarian and strategic responses
- **Internet outages** (Cloudflare Radar) — potential indicators of censorship or infrastructure attacks

Each API is unique. AIS needs persistent WebSocket connections. OpenSky requires OAuth2 authentication. ACLED requires access tokens. Generic workers don't work — each source needs a purpose-built adapter.

### What Use It Provides

- **AIS vessel tracking** for IOR, South China Sea, Persian Gulf, and key chokepoints
- **Aircraft monitoring** for unusual military flights near contested regions
- **Structured conflict data** with actor attribution and geographic coordinates
- **Disaster monitoring** for earthquakes, fires, and climate events
- **Internet outage detection** as an early indicator of crisis situations

### How It Is Built

1. **AIS Worker** — `backend/app/workers/ais_worker.py`:
   ```python
   class AISWorker:
       BOUNDING_BOXES = [
           # Indian Ocean Region
           [[35, -20], [100, 30]],
           # South China Sea
           [[100, 0], [125, 25]],
           # Persian Gulf / Strait of Hormuz
           [[48, 20], [60, 30]],
           # Strait of Malacca
           [[95, -2], [110, 8]],
       ]

       async def connect(self):
           uri = f"wss://stream.aisstream.io/v0/stream?APIKey={API_KEY}"
           async with websockets.connect(uri) as ws:
               await ws.send(json.dumps({
                   "APIKey": API_KEY,
                   "BoundingBoxes": self.BOUNDING_BOXES
               }))
               async for message in ws:
                   vessel = json.loads(message)
                   await self.redis.xadd("feeds:raw", {
                       "type": "ais",
                       "mmsi": vessel["MetaData"]["MMSI"],
                       "ship_name": vessel["MetaData"]["ShipName"],
                       "lat": vessel["MetaData"]["latitude"],
                       "lng": vessel["MetaData"]["longitude"],
                       "flag": vessel["MetaData"]["flag"],
                       "ship_type": vessel["MetaData"]["ShipType"],
                   })
   ```

2. **ACLED Worker** — polls the ACLED API for conflict events in regions of Indian interest:
   ```python
   class ACLEDWorker:
       REGIONS = ["Southern Asia", "South-Eastern Asia", "Eastern Asia",
                  "Western Asia", "Eastern Africa", "Central Asia"]

       async def poll(self):
           for region in self.REGIONS:
               resp = await self.fetch(
                   "https://api.acleddata.com/acled/read",
                   params={"region": region, "limit": 100, "event_date_where": ">=last_7_days"},
                   headers={"Authorization": f"Bearer {ACLED_TOKEN}"}
               )
               for event in resp["data"]:
                   await self.redis.xadd("feeds:raw", self._normalize(event))
   ```

3. **USGS Earthquake Worker** — public API, no auth:
   ```python
   class USGSWorker:
       async def poll(self):
           resp = await self.fetch(
               "https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_day.geojson"
           )
           for feature in resp["features"]:
               await self.redis.xadd("feeds:raw", self._normalize(feature))
   ```

4. **Worker registry**: all workers registered in `config/workers.yaml` with individual polling intervals, authentication config, and circuit breaker parameters.

---

## F37 — Data Freshness Monitor

### What Is Being Built

A monitoring system that tracks the last-received timestamp for every data source and reports which sources are fresh, stale, or dead. It powers the "intelligence gap" display — explicitly telling analysts what BIE *cannot* see right now.

### Why It Is Needed

Intelligence analysis is only as good as its data. If the AIS feed goes down and BIE silently stops showing vessel positions, analysts might conclude the IOR is quiet — when actually, BIE is blind to maritime activity. Explicit gap reporting prevents false confidence.

The freshness monitor also enables proactive alerting — if a Tier 1 source (PTI, Reuters) goes stale for > 30 minutes, something is wrong.

### What Use It Provides

- **Per-source freshness** — last update timestamp for every feed and API
- **Status categorization** — `fresh` (< 15 min), `stale` (< 1h), `very_stale` (< 6h), `dead` (> 6h)
- **Intelligence gap reporting** — "AIS feed is down. No maritime data for 45 minutes."
- **Frontend indicator** — status bar showing data health
- **Alerting** — Tier 1 sources stale > 30 min triggers a log warning

### How It Is Built

1. **Freshness tracking**: every time an item is ingested from a source, update the source's last-seen timestamp in Redis:
   ```python
   await redis.hset("bie:freshness", feed_id, datetime.utcnow().isoformat())
   ```

2. **Freshness endpoint** — `GET /api/freshness`:
   ```python
   @router.get("/api/freshness")
   async def get_freshness():
       sources = await redis.hgetall("bie:freshness")
       result = []
       for source_id, last_seen in sources.items():
           age = datetime.utcnow() - datetime.fromisoformat(last_seen)
           status = "fresh" if age < timedelta(minutes=15) else \
                    "stale" if age < timedelta(hours=1) else \
                    "very_stale" if age < timedelta(hours=6) else "dead"
           result.append({"source": source_id, "last_seen": last_seen,
                         "age_seconds": age.total_seconds(), "status": status})
       return sorted(result, key=lambda x: x["age_seconds"], reverse=True)
   ```

3. **Intelligence gap display**: the frontend extracts dead/stale sources and groups them by data type:
   - "⚠️ Maritime intelligence gap — AIS feed stale for 2h"
   - "⚠️ No conflict data — ACLED last updated 8h ago"
   - "✅ All Tier 1 news feeds healthy"

---

← Back to [prd.md](./prd.md)
