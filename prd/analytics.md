# Analytics Engine

> Covers: F14 Global Strategic Hotspot Database · F25 BIE Instability Index (BII) · F26 Anomaly Detection Engine · F27 Signal Convergence Detector · F33 Infrastructure Proximity

---

## F14 — Global Strategic Hotspot Database

### What Is Being Built

A curated database of 500+ strategic locations worldwide that are relevant to Indian national security. Each location has precise coordinates, alternate names (including Hindi/Devanagari), a baseline threat level, and the strategic region it belongs to. This database serves as BIE's geocoder — resolving mentions of "Pangong Tso", "Hambantota", "Scarborough Shoal", or "Strait of Hormuz" to precise map coordinates.

### Why It Is Needed

Generic geocoders (Google Maps, Nominatim) fail on strategic locations. "Aksai Chin" returns ambiguous results. "CPEC" doesn't geocode to a route. "Galwan Valley" might resolve to the wrong country depending on provider. BIE needs a purpose-built gazetteer that understands strategic geography from India's perspective.

The database is organized in 3 tiers by proximity and relevance to India:

| Tier | Scope | Count | Examples |
|---|---|---|---|
| **Tier 1** | India & immediate borders | ~200 | LAC landmarks, LOC posts, military bases, Naxal corridors, IOR ports |
| **Tier 2** | Primary strategic interest | ~200 | SCS features, Gulf chokepoints, BRI ports, Chinese bases, Quad nations' key locations |
| **Tier 3** | Extended strategic interest | ~100 | African BRI projects, Central Asian bases, Pacific Island naval facilities |

### What Use It Provides

- **Precision geocoding** for 500+ locations generic geocoders miss
- **Alias resolution** — "PLA" → "People's Liberation Army", "SCS" → "South China Sea"
- **Strategic context** — each location carries metadata: baseline threat, region, type, country
- **NER enhancement** — locations in the hotspot DB are added to the NER model's training data
- **Proximity analysis** — "this event happened 45km from Gwadar Port"

### How It Is Built

1. **Data file** — `backend/data/hotspots.json`:
   ```json
   [
     {
       "id": "pangong_tso",
       "name": "Pangong Tso",
       "aliases": ["Pangong Lake", "पांगोंग त्सो", "班公错"],
       "lat": 33.75, "lng": 78.65,
       "region": "LAC_WEST",
       "type": "BORDER_LANDMARK",
       "tier": 1,
       "baseline_threat": "HIGH",
       "country": "IN/CN",
       "description": "Contested lake on the LAC. Site of 2020 India-China standoff."
     },
     {
       "id": "gwadar_port",
       "name": "Gwadar Port",
       "aliases": ["گوادر بندرگاہ"],
       "lat": 25.12, "lng": 62.33,
       "region": "IOR",
       "type": "PORT",
       "tier": 2,
       "baseline_threat": "MEDIUM",
       "country": "PK",
       "description": "CPEC terminus. Chinese-operated deep-water port with potential dual-use military capability."
     },
     {
       "id": "scarborough_shoal",
       "name": "Scarborough Shoal",
       "aliases": ["Bajo de Masinloc", "黄岩岛"],
       "lat": 15.18, "lng": 117.76,
       "region": "SOUTH_CHINA_SEA",
       "type": "MARITIME_FEATURE",
       "tier": 2,
       "baseline_threat": "MEDIUM",
       "country": "DISPUTED",
       "description": "Contested feature in the SCS. China-Philippines flashpoint affecting regional stability."
     },
     {
       "id": "djibouti_base",
       "name": "PLA Support Base Djibouti",
       "aliases": ["China Djibouti Base", "吉布提保障基地"],
       "lat": 11.59, "lng": 43.15,
       "region": "HORN_OF_AFRICA",
       "type": "MILITARY_BASE",
       "tier": 2,
       "baseline_threat": "MEDIUM",
       "country": "DJ",
       "description": "China's first overseas military base. Monitors India's IOR activity."
     },
     {
       "id": "strait_of_hormuz",
       "name": "Strait of Hormuz",
       "aliases": ["Hormuz Strait", "تنگه هرمز"],
       "lat": 26.57, "lng": 56.25,
       "region": "PERSIAN_GULF",
       "type": "CHOKEPOINT",
       "tier": 2,
       "baseline_threat": "MEDIUM",
       "country": "IR/OM",
       "description": "20% of global oil transit. India imports ~60% of oil through Gulf routes."
     }
   ]
   ```

2. **Regions**: `LAC_WEST`, `LAC_EAST`, `LOC_KASHMIR`, `SIACHEN`, `NORTHEAST`, `IOR`, `SOUTH_CHINA_SEA`, `TAIWAN_STRAIT`, `PERSIAN_GULF`, `RED_SEA`, `HORN_OF_AFRICA`, `CENTRAL_ASIA`, `SOUTHEAST_ASIA`, `INTERNAL_CENTRAL`, `INTERNAL_SOUTH`

3. **Lookup** — `backend/app/analytics/hotspots.py`:
   ```python
   class HotspotDB:
       def __init__(self):
           self.hotspots = json.load(open("data/hotspots.json"))
           self.alias_map = {}
           for h in self.hotspots:
               for alias in [h["name"]] + h.get("aliases", []):
                   self.alias_map[alias.lower()] = h

       def resolve(self, text: str) -> Optional[dict]:
           return self.alias_map.get(text.lower())

       def nearest(self, lat: float, lng: float, radius_km: float = 200) -> list:
           return [h for h in self.hotspots
                   if haversine(lat, lng, h["lat"], h["lng"]) <= radius_km]
   ```

---

## F25 — BIE Instability Index (BII)

### What Is Being Built

A real-time composite instability score calculated for every strategic region and key country that India monitors. The BII aggregates event frequency, threat level severity, category diversity, and anomaly signals into a 0.0–1.0 score per region/country, updated every 5 minutes.

### Why It Is Needed

Raw event counts don't tell analysts "how worried should I be?" about a region. A single CRITICAL military event near the LAC might be more significant than 20 LOW-level diplomatic events in Southeast Asia. The BII weights events by threat level, adjusts for regional baselines, and accounts for conflict zones where instability is the norm.

### What Use It Provides

- **Per-region scores** for all strategic regions (LAC, SCS, Gulf, IOR, etc.)
- **Per-country scores** for key countries (China, Pakistan, Afghanistan, Myanmar, Iran, etc.)
- **Trend indicators** — RISING / STABLE / FALLING per region
- **Conflict-zone floor scores** — Kashmir never drops below 0.5, Afghanistan never below 0.6
- **India-weighted** — events closer to India or more directly impacting Indian interests carry higher weight
- **Historical tracking** — 7-day rolling baselines for trend detection

### How It Is Built

1. **Core formula**:
   ```
   BII(region) = w₁·EventFrequency + w₂·ThreatSeverity + w₃·CategoryDiversity + w₄·AnomalyBoost
   ```
   Where:
   - `EventFrequency` = events in 24h / 7-day rolling average (capped at 3.0)
   - `ThreatSeverity` = weighted sum: LOW×1 + MEDIUM×2 + HIGH×4 + CRITICAL×8
   - `CategoryDiversity` = unique categories ÷ 7 (more diverse = more concerning)
   - `AnomalyBoost` = 0.2 if active anomaly in region, 0 otherwise
   - Weights: w₁=0.3, w₂=0.35, w₃=0.15, w₄=0.2

2. **Conflict-zone floor scores** — prevent regions with chronic instability from appearing "calm" during lulls:
   ```python
   FLOOR_SCORES = {
       "LOC_KASHMIR": 0.50,
       "LAC_WEST": 0.35,
       "LAC_EAST": 0.30,
       "SOUTH_CHINA_SEA": 0.35,
       "PERSIAN_GULF": 0.30,
       "RED_SEA": 0.40,       # Houthi threat zone
       "HORN_OF_AFRICA": 0.35,
       "CENTRAL_ASIA": 0.30,  # Afghanistan instability
   }

   def calculate_bii(self, region):
       raw_score = self._raw_score(region)
       floor = FLOOR_SCORES.get(region, 0.0)
       return max(raw_score, floor)
   ```

3. **India-proximity weighting** — events in India's immediate neighborhood carry 1.5× weight:
   ```python
   PROXIMITY_WEIGHTS = {
       "SOUTH_ASIA": 1.5,
       "IOR": 1.3,
       "INDO_PACIFIC": 1.1,
       "MIDDLE_EAST": 1.0,
       "CENTRAL_ASIA": 1.0,
       "AFRICA": 0.8,
       "EUROPE": 0.7,
       "AMERICAS": 0.6,
   }
   ```

4. **Country-level scoring**: for key countries, aggregate all events mentioning that country:
   ```python
   KEY_COUNTRIES = ["CN", "PK", "AF", "MM", "LK", "NP", "BD", "IR", "SA", "TR", "RU", "US", "JP", "AU"]
   ```

5. **Storage**: Redis hash `bie:instability:{region}` with score, trend, event count, last update.

---

## F26 — Anomaly Detection Engine

### What Is Being Built

A statistical engine that detects unusual event patterns — spikes in frequency, unexpected categories, or new entities appearing — relative to learned baselines. When a region shows significantly more activity than its historical norm, an anomaly alert fires.

### Why It Is Needed

Not all event spikes are meaningful. Political events spike every election cycle — that's normal. But a sudden military event spike near the LAC *outside* election season, or an unexpected surge in cyber events targeting India, is genuinely anomalous and warrants attention.

The engine learns what "normal" looks like for each region×category×time combination, then flags deviations.

### What Use It Provides

- **Welford's online algorithm** — streaming mean/variance calculation (O(1) time and space)
- **Multi-dimensional baselines** — per region, per category, per weekday, per time-of-day
- **Configurable thresholds** — default 2σ deviation triggers alert
- **Anomaly types**: frequency spike, category anomaly, entity anomaly
- **24-hour TTL** — anomalies auto-expire to prevent alert fatigue

### How It Is Built

1. **Welford's online algorithm** for streaming stats:
   ```python
   class WelfordBaseline:
       def __init__(self):
           self.count = 0
           self.mean = 0.0
           self.M2 = 0.0

       def update(self, value):
           self.count += 1
           delta = value - self.mean
           self.mean += delta / self.count
           delta2 = value - self.mean
           self.M2 += delta * delta2

       @property
       def variance(self):
           return self.M2 / max(self.count, 1)

       @property
       def std_dev(self):
           return self.variance ** 0.5

       def is_anomalous(self, value, sigma=2.0):
           if self.count < 7:  # Need minimum baseline
               return False
           return value > self.mean + sigma * self.std_dev
   ```

2. **Baseline keys**: `{region}:{category}:{weekday}` — e.g., "LAC_WEST:MILITARY:tuesday"

3. **Detection cycle**: every 5 minutes, count events in each region×category for the last hour, compare to baseline, fire anomaly if > 2σ deviation.

---

## F27 — Signal Convergence Detector

### What Is Being Built

An engine that detects when multiple independent signals converge in the same geographic area within a short time window — a strong indicator of escalation or a developing situation. When 3+ different event categories cluster within a geographic cell, a convergence alert fires.

### Why It Is Needed

Individual events may not be alarming. But when MILITARY, DIPLOMATIC, and ECONOMIC signals all spike near the same location simultaneously, it suggests a coordinated escalation that individual-event analysis would miss.

Example: Near Aksai Chin, within 6 hours: (1) PLA troop deployment reported, (2) MEA issues diplomatic note, (3) India halts trade at Lipulekh pass. Each alone is routine. Together, they indicate a developing LAC crisis.

### What Use It Provides

- **Geographic cell grid** — 1°×1° latitude/longitude cells worldwide
- **Multi-category detection** — 3+ different categories in same cell within 6 hours
- **Anomaly enhancement** — convergence + existing anomaly = HIGH severity alert
- **Auto-expire** — alerts expire after 12 hours unless renewed by new signals
- **Real-time notification** — convergence fires a Redis pub/sub event for the posture panel

### How It Is Built

1. **Geographic binning** — events are binned into 1°×1° cells:
   ```python
   def cell_id(lat, lng):
       return f"{int(lat)}_{int(lng)}"
   ```

2. **Convergence check** — every 5 minutes, scan all active cells:
   ```python
   class ConvergenceDetector:
       def check(self, events_last_6h):
           cells = defaultdict(lambda: {"categories": set(), "events": [], "count": 0})
           for event in events_last_6h:
               cell = cell_id(event["lat"], event["lng"])
               cells[cell]["categories"].add(event["category"])
               cells[cell]["events"].append(event["id"])
               cells[cell]["count"] += 1

           alerts = []
           for cell, data in cells.items():
               if len(data["categories"]) >= 3:
                   severity = "HIGH" if self._has_active_anomaly(cell) else "MEDIUM"
                   alerts.append({
                       "cell": cell,
                       "categories": list(data["categories"]),
                       "event_count": data["count"],
                       "severity": severity,
                   })
           return alerts
   ```

---

## F33 — Infrastructure Proximity

### What Is Being Built

An analysis layer that enriches each event with proximity data to strategic infrastructure — military bases, ports, airfields, nuclear facilities, chokepoints, and BRI projects. When an event occurs within 200km of strategic infrastructure, the proximity is flagged in the event detail.

### Why It Is Needed

Context changes everything. "Military exercise reported" is routine. "Military exercise reported 40km from Hambantota Port (Chinese-operated)" is intelligence. Proximity analysis adds this "so what?" factor to every event, connecting it to the strategic infrastructure landscape.

### What Use It Provides

- **200km proximity radius** — every event checked against the infrastructure database
- **Infrastructure types**: military bases, ports, airfields, nuclear facilities, chokepoints, BRI projects, radar installations
- **Global coverage** — not just Indian infrastructure, but Chinese overseas bases, BRI ports, and NATO installations relevant to India's strategic environment
- **Cascade analysis** — "if this chokepoint closes, which supply routes are affected?"
- **Display in EventDetailPanel** — "⚡ 45km from Gwadar Port (Chinese-operated)"

### How It Is Built

1. **Infrastructure database** — subset of the hotspot DB (F14) filtered to `type in [MILITARY_BASE, PORT, AIRFIELD, NUCLEAR, CHOKEPOINT, BRI_PROJECT]`.

2. **Proximity check**:
   ```python
   def check_proximity(event_lat, event_lng, infrastructure_db, radius_km=200):
       nearby = []
       for infra in infrastructure_db:
           dist = haversine(event_lat, event_lng, infra["lat"], infra["lng"])
           if dist <= radius_km:
               nearby.append({
                   "name": infra["name"],
                   "type": infra["type"],
                   "distance_km": round(dist, 1),
                   "country": infra["country"],
                   "description": infra["description"],
               })
       return sorted(nearby, key=lambda x: x["distance_km"])
   ```

3. **Haversine formula**:
   ```python
   from math import radians, sin, cos, asin, sqrt

   def haversine(lat1, lng1, lat2, lng2):
       lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
       dlat = lat2 - lat1
       dlng = lng2 - lng1
       a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
       return 6371 * 2 * asin(sqrt(a))  # km
   ```

---

← Back to [prd.md](./prd.md)
