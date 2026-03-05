# NLP Pipeline

> Covers: F16 spaCy Pipeline Setup · F17 Strategic NER (India-Priority) · F18 Hindi / Urdu NLP · F19 Hybrid Threat Classifier · F20 NLP Worker Integration

---

## F16 — spaCy Pipeline Setup

### What Is Being Built

The core NLP processing engine using spaCy — a production-grade NLP library for entity extraction, language detection, and text analysis. This processes every ingested item: extracting entities (people, organizations, locations, military units, weapon systems), detecting language, and preparing text for downstream classification and graph population.

### Why It Is Needed

Raw news text is unstructured. BIE needs to convert "China deployed J-20 fighters near the Senkaku Islands" into structured data: `{entity: "J-20", type: WEAPON_SYSTEM}`, `{entity: "China", type: GPE}`, `{entity: "Senkaku Islands", type: LOCATION}`. Without NLP, there's no data for the knowledge graph, no entity-based search, and no structured analytics.

### What Use It Provides

- **Entity extraction** from global news — people, organizations, locations, military, weapons
- **Multi-language support** — English, Hindi, Urdu processing
- **Batch processing** — `nlp.pipe()` handles 20+ items per second
- **Extensible** — custom NER model adds domain-specific entities that base models miss

### How It Is Built

1. **Install**:
   ```bash
   pip install spacy
   python -m spacy download en_core_web_sm
   ```

2. **Pipeline** — `backend/app/nlp/pipeline.py`:
   ```python
   import spacy

   class BIENLPPipeline:
       def __init__(self):
           self.nlp = spacy.load("en_core_web_sm")

       def process(self, text: str) -> dict:
           doc = self.nlp(text)
           entities = [
               {"text": ent.text, "label": ent.label_, "start": ent.start_char, "end": ent.end_char}
               for ent in doc.ents
           ]
           return {"entities": entities, "language": self._detect_language(text)}

       def batch_process(self, texts: list) -> list:
           return [self.process(doc.text) for doc in self.nlp.pipe(texts, batch_size=20)]
   ```

---

## F17 — Strategic NER (India-Priority)

### What Is Being Built

A custom Named Entity Recognition model that recognizes entities of strategic importance to Indian intelligence — entities that default spaCy models miss. This spans Indian military units, Pakistani military, Chinese PLA designations, weapon systems, border landmarks, strategic infrastructure, and key international organizations.

### Why It Is Needed

Default NER models handle generic entities (person, org, location) but miss domain-specific ones. "3 Bihar Regiment" is an Indian Army unit, not a location. "BrahMos" is a weapon system, not a person. "Pangong Tso" is a strategic border lake, not a generic location. "PLA Eastern Theater Command" is a Chinese military group, not a generic organization. "CPEC" is a strategic corridor, not an acronym.

BIE must recognize these to populate the knowledge graph accurately and provide India-relevant intelligence analysis of global events.

### What Use It Provides

Custom entity labels extending the base spaCy NER:

| Label | Examples |
|---|---|
| `MILITARY_UNIT` | 3 Bihar Regiment, INS Vikrant, PLA Eastern Theater Command, PLAN South Sea Fleet, ISI, RAW, 2nd Artillery Corps |
| `WEAPON_SYSTEM` | BrahMos, S-400, DF-21D, Agni-V, J-20, Rafale, INS Arihant, INS Kalvari |
| `BORDER_LANDMARK` | Pangong Tso, Galwan Valley, Aksai Chin, Doklam, Siachen, Senkaku/Diaoyu, Scarborough Shoal |
| `INFRASTRUCTURE` | CPEC, Gwadar Port, Hambantota Port, Chabahar Port, Djibouti Base, Agaléga, Duqm Base |
| `INDIAN_ORG` | DRDO, ISRO, NSA, MEA, NTRO, HAL, BEL, ONGC |
| `STRATEGIC_ORG` | QUAD, SCO, BRICS, AUKUS, ASEAN, IORA, BIMSTEC, NATO, AIIB |
| `STRATEGIC_CORRIDOR` | CPEC, BRI, INSTC (Int'l North-South Transport Corridor), Chabahar-Zahedan Railway |

### How It Is Built

1. **Training data**: 500+ labeled examples sourced from Indian defence news (Bharat Shakti, Indian Defence Review), global defence outlets (Janes, Defense One), and think tank analyses (IDSA, IISS, CSIS).

2. **Custom model**:
   ```python
   # Add labels to NER pipe
   ner = nlp.get_pipe("ner")
   for label in ["MILITARY_UNIT", "WEAPON_SYSTEM", "BORDER_LANDMARK",
                  "INFRASTRUCTURE", "INDIAN_ORG", "STRATEGIC_ORG", "STRATEGIC_CORRIDOR"]:
       ner.add_label(label)
   ```

3. **Alias resolution**: "PLA" → "People's Liberation Army", "SCS" → "South China Sea", "IOR" → "Indian Ocean Region" — normalizes abbreviations used across different sources.

---

## F18 — Hindi / Urdu NLP

### What Is Being Built

Support for processing Hindi and Urdu text from Indian and Pakistani news sources, enabling BIE to ingest and analyze non-English intelligence. This includes Devanagari and Nastaliq script processing, transliteration to Latin script for graph consistency, and Hindi NER models.

### Why It Is Needed

India's Hindi-language media (Navbharat Times, Dainik Jagran, BBC Hindi) carries significant intelligence value — especially for domestic security, border developments, and defence procurement coverage. Pakistani Urdu-language media (Dawn Urdu, Jang) provides perspective on Pakistan's strategic thinking. Ignoring these languages means ignoring critical sources.

### What Use It Provides

- **Hindi/Urdu text processing** using IndicNER or equivalent models
- **Transliteration** — Hindi entities rendered in Latin script for graph consistency
- **Language detection** — auto-detect language and route to appropriate pipeline
- **70%+ entity accuracy** on Hindi defence/security text (acceptable for augmenting English-primary coverage)

### How It Is Built

1. **Hindi model**: `ai4bharat/IndicNER` or fine-tuned XLM-RoBERTa
2. **Transliteration**: `indic-transliteration` library → Latin script
3. **Language routing**:
   ```python
   def process(self, text, language=None):
       if language == "hi" or self._detect_hindi(text):
           return self._process_hindi(text)
       elif language == "ur":
           return self._process_urdu(text)
       return self._process_english(text)
   ```

---

## F19 — Hybrid Threat Classifier

### What Is Being Built

A classification system that assigns each event a **category** (MILITARY, DIPLOMATIC, ECONOMIC, INTERNAL, MARITIME, CYBER, TERRORISM) and a **threat level** (LOW, MEDIUM, HIGH, CRITICAL). It uses a two-stage hybrid approach: instant keyword rules for speed, with an ML model for nuanced classification.

### Why It Is Needed

Classification drives everything downstream: marker colors on the map, category filtering in the news feed, instability index calculation, and convergence detection. Without accurate classification, the entire analytics pipeline produces garbage.

The 7-category system covers the full spectrum of India's strategic concerns — from traditional military threats to emerging cyber warfare and terrorism.

### What Use It Provides

- **Instant classification** — keyword rules deliver results in < 10ms, displayed immediately
- **ML-enhanced accuracy** — neural classifier runs asynchronously, upgrades results
- **7 categories** covering military, diplomatic, economic, internal, maritime, cyber, and terrorism events
- **4 threat levels** — LOW, MEDIUM, HIGH, CRITICAL
- **Confidence score** — enables downstream "low confidence" disclaimers
- **India-relevance scoring** — how directly does this global event affect Indian interests

### How It Is Built

1. **Stage 1 — Keyword rules** (instant):
   ```python
   KEYWORD_RULES = {
       "MILITARY": {
           "keywords": ["troops", "deployed", "missile", "fighter", "exercise", "naval",
                       "battalion", "brigade", "PLA", "regiment", "airstrike", "artillery",
                       "aircraft carrier", "submarine", "arms deal", "defence pact"],
           "weight": 1.0
       },
       "DIPLOMATIC": {
           "keywords": ["summit", "treaty", "ambassador", "sanctions", "bilateral",
                       "diplomatic", "foreign minister", "state visit", "UN resolution",
                       "ceasefire", "peace talks", "expelled diplomat"],
           "weight": 1.0
       },
       "ECONOMIC": {
           "keywords": ["trade deal", "tariff", "investment", "GDP", "currency",
                       "supply chain", "BRI", "CPEC", "port deal", "infrastructure investment",
                       "sanctions", "embargo", "FDI"],
           "weight": 1.0
       },
       "INTERNAL": {
           "keywords": ["protest", "election", "riot", "coup", "reform", "legislation",
                       "unrest", "political crisis", "emergency", "martial law"],
           "weight": 1.0
       },
       "MARITIME": {
           "keywords": ["vessel", "naval", "shipping", "piracy", "strait", "chokepoint",
                       "port", "coast guard", "maritime", "fishing dispute", "EEZ",
                       "freedom of navigation"],
           "weight": 1.0
       },
       "CYBER": {
           "keywords": ["cyber attack", "hack", "data breach", "ransomware", "espionage",
                       "APT", "malware", "critical infrastructure", "CISA", "cyber warfare"],
           "weight": 1.0
       },
       "TERRORISM": {
           "keywords": ["terrorist", "bombing", "IED", "insurgent", "militant",
                       "radicalization", "counter-terrorism", "hostage", "suicide attack",
                       "ISIS", "Al-Qaeda", "Jaish", "LeT", "Hizbul"],
           "weight": 1.0
       },
   }
   ```

2. **Stage 2 — ML classifier** (async):
   ```python
   class ThreatClassifier:
       def classify(self, text, entities):
           # Stage 1: instant keyword match
           keyword_result = self._keyword_classify(text)

           # Stage 2: ML model (runs async, upgrades classification)
           ml_result = self._ml_classify(text)

           # Merge: ML takes precedence if confidence > 0.7
           if ml_result["confidence"] > 0.7:
               return ml_result
           return keyword_result
   ```

3. **Threat level assignment**:
   ```python
   THREAT_RULES = {
       "CRITICAL": ["nuclear", "war declared", "invasion", "missile launch", "genocide"],
       "HIGH": ["airstrike", "troops deployed", "military escalation", "border clash",
                "terrorist attack", "critical infrastructure breach"],
       "MEDIUM": ["military exercise", "diplomatic tension", "sanctions", "protest",
                 "cyber incident", "maritime dispute"],
       "LOW": ["bilateral talks", "trade agreement", "joint statement", "port visit"],
   }
   ```

4. **India-relevance scoring**: calculates how directly a global event affects Indian interests based on geography, entities mentioned, and category:
   ```python
   def india_relevance(self, event):
       score = 0.0
       if any(e in event["entities"] for e in INDIA_ENTITIES):
           score += 0.4  # Directly mentions India-related entity
       if event["geo_focus"] in ["SOUTH_ASIA"]:
           score += 0.3  # In India's neighborhood
       if event["geo_focus"] in ["INDO_PACIFIC", "MIDDLE_EAST"]:
           score += 0.2  # In India's broader strategic interest
       if event["category"] in ["MILITARY", "MARITIME"]:
           score += 0.1  # Security-relevant category
       return min(score, 1.0)
   ```

---

## F20 — NLP Worker Integration

### What Is Being Built

A persistent worker that bridges raw data ingestion and NLP processing. It reads from the `feeds:raw` Redis Stream, runs each item through the NLP pipeline (entity extraction, classification, India-relevance scoring), and publishes enriched items to `feeds:processed`.

### Why It Is Needed

The RSS worker (F09) produces raw feed items. The NLP pipeline (F16–F19) processes them. The NLP Worker connects these two stages, running continuously to ensure every ingested item is classified and enriched before reaching the frontend or knowledge graph.

### What Use It Provides

- **Continuous processing** — listens to `feeds:raw`, never stops
- **Consumer groups** — multiple NLP workers can run in parallel (horizontal scaling)
- **Dead-letter queue** — items that fail processing go to `feeds:dead_letter` for review
- **Enrichment** — raw item → NLP-enriched event with entities, category, threat level, India-relevance score

### How It Is Built

```python
class NLPWorker:
    def __init__(self, redis, pipeline, classifier):
        self.redis = redis
        self.pipeline = pipeline
        self.classifier = classifier

    async def run(self):
        group = "nlp-workers"
        try:
            await self.redis.xgroup_create("feeds:raw", group, mkstream=True)
        except Exception:
            pass  # Already exists

        while True:
            items = await self.redis.xreadgroup(group, "worker-1", {"feeds:raw": ">"}, count=10)
            for stream, messages in items:
                for msg_id, data in messages:
                    try:
                        nlp_result = self.pipeline.process(data["title"] + " " + data.get("summary", ""))
                        classification = self.classifier.classify(
                            data["title"] + " " + data.get("summary", ""),
                            nlp_result["entities"]
                        )
                        enriched = {
                            **data,
                            "entities": json.dumps(nlp_result["entities"]),
                            "category": classification["category"],
                            "threat_level": classification["threat_level"],
                            "classification_confidence": classification["confidence"],
                            "india_relevance": classification.get("india_relevance", 0.5),
                            "processed_at": datetime.utcnow().isoformat(),
                        }
                        await self.redis.xadd("feeds:processed", enriched)
                        await self.redis.xack("feeds:raw", group, msg_id)
                    except Exception as e:
                        await self.redis.xadd("feeds:dead_letter", {**data, "error": str(e)})
                        await self.redis.xack("feeds:raw", group, msg_id)
```

---

← Back to [prd.md](./prd.md)
