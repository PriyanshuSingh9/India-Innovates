# Knowledge Graph & Vector Search

> Covers: F21 Neo4j Schema & Seed Data · F22 Automated Graph Writer · F23 GraphRAG Query Engine · F24 Qdrant Vector Store

---

## F21 — Neo4j Schema & Seed Data

### What Is Being Built

The Neo4j graph database schema and initial seed data that forms BIE's "institutional memory." The graph stores entities (people, organizations, countries, locations, military units, alliances) and their relationships, extracted from ingested intelligence. The seed data provides 1,000+ facts about India's global strategic environment to ensure the graph is useful from Day 1.

### Why It Is Needed

A knowledge graph enables queries that flat databases cannot: "Which Chinese military units have been mentioned near the LAC in the past month?" or "Show all entities connected to Gwadar Port within 2 hops." Without a pre-seeded graph, BIE starts empty — and GraphRAG has nothing to traverse.

### What Use It Provides

- **Relationship queries** between entities across global events
- **Multi-hop traversals** — connect seemingly unrelated entities through shared relationships
- **Pre-seeded intelligence** — 1,000 facts about India's strategic environment:
  - Indian military structure (commands, units, key personnel)
  - Chinese PLA structure (theater commands, PLAN fleets, key bases)
  - Pakistan military structure (corps, Air Force, ISI)
  - Key alliances and groupings (QUAD, SCO, BRICS, AUKUS, ASEAN)
  - BRI project locations and operators
  - Strategic infrastructure (ports, bases, chokepoints)
  - Key diplomatic relationships and rivalries

### How It Is Built

1. **Node labels & properties**:

   | Label | Properties | Constraints |
   |---|---|---|
   | `Person` | name, role, nationality, aliases[] | UNIQUE on name |
   | `Organization` | name, type, country, aliases[] | UNIQUE on name |
   | `Country` | name, code (ISO), region, aliases[] | UNIQUE on code |
   | `Location` | name, lat, lng, type, region, country | UNIQUE on name |
   | `Event` | id, title, summary, category, threat_level, timestamp | UNIQUE on id |
   | `MilitaryUnit` | name, branch, country, parent_command | UNIQUE on name |
   | `WeaponSystem` | name, type, country, operator | UNIQUE on name |
   | `Infrastructure` | name, type, lat, lng, operator, country | UNIQUE on name |
   | `Alliance` | name, type, members[], founded | UNIQUE on name |

2. **Relationship types**:

   | Type | From → To | Properties |
   |---|---|---|
   | `MENTIONED_IN` | Entity → Event | role, confidence |
   | `OPERATES_IN` | Org/Unit → Location | since |
   | `DEPLOYED_AT` | MilitaryUnit → Location | since, status |
   | `ALLIES_WITH` | Country/Org → Country/Org | — |
   | `RIVALS_WITH` | Country/Org → Country/Org | — |
   | `MEMBER_OF` | Country → Alliance | since |
   | `NEAR` | Location → Location | distance_km |
   | `PART_OF` | Location → Location | — (city → state → country) |
   | `COMMANDS` | Person → Org/Unit | title |
   | `OPERATES` | Country/Org → Infrastructure | since, capacity |
   | `RELATED_TO` | Entity → Entity | co_occurrence_count |

3. **Seed data** — `backend/scripts/seed_neo4j.py`:
   ```python
   SEED_DATA = {
       "countries": [
           {"name": "India", "code": "IN", "region": "SOUTH_ASIA"},
           {"name": "China", "code": "CN", "region": "EAST_ASIA"},
           {"name": "Pakistan", "code": "PK", "region": "SOUTH_ASIA"},
           {"name": "United States", "code": "US", "region": "AMERICAS"},
           # ... 30+ key countries
       ],
       "alliances": [
           {"name": "QUAD", "type": "strategic", "members": ["IN", "US", "JP", "AU"]},
           {"name": "SCO", "type": "multilateral", "members": ["CN", "RU", "IN", "PK", "KZ", "UZ", "KG", "TJ"]},
           {"name": "BRICS", "type": "economic", "members": ["BR", "RU", "IN", "CN", "ZA", "EG", "ET", "IR", "SA", "AE"]},
           {"name": "AUKUS", "type": "security", "members": ["AU", "UK", "US"]},
       ],
       "military_units": [
           {"name": "PLA Eastern Theater Command", "branch": "PLA", "country": "CN"},
           {"name": "PLA Western Theater Command", "branch": "PLA", "country": "CN"},
           {"name": "PLAN South Sea Fleet", "branch": "PLAN", "country": "CN"},
           {"name": "Indian Northern Command", "branch": "Indian Army", "country": "IN"},
           {"name": "Indian Eastern Naval Command", "branch": "Indian Navy", "country": "IN"},
           # ... 50+ units
       ],
       "infrastructure": [
           {"name": "Gwadar Port", "type": "PORT", "lat": 25.12, "lng": 62.33, "operator": "CN", "country": "PK"},
           {"name": "Hambantota Port", "type": "PORT", "lat": 6.12, "lng": 81.10, "operator": "CN", "country": "LK"},
           {"name": "PLA Djibouti Base", "type": "MILITARY_BASE", "lat": 11.59, "lng": 43.15, "operator": "CN", "country": "DJ"},
           {"name": "INS Kadamba", "type": "NAVAL_BASE", "lat": 14.79, "lng": 74.13, "operator": "IN", "country": "IN"},
           # ... 100+ infrastructure
       ],
   }
   ```

4. **Indexes**:
   ```cypher
   CREATE INDEX ON :Person(name)
   CREATE INDEX ON :Organization(name)
   CREATE INDEX ON :Country(code)
   CREATE INDEX ON :Location(name)
   CREATE INDEX ON :Event(id)
   CREATE INDEX ON :Event(timestamp)
   CREATE INDEX ON :Event(category)
   CREATE INDEX ON :Alliance(name)
   ```

---

## F22 — Automated Graph Writer

### What Is Being Built

A worker that reads NLP-enriched events from `feeds:processed` and writes extracted entities and relationships to Neo4j. For each event, it creates/updates entity nodes and creates `MENTIONED_IN` edges connecting entities to events.

### Why It Is Needed

Manual graph population doesn't scale. BIE ingests hundreds of events per day from global sources. The graph writer automates the knowledge graph population, ensuring every processed event enriches BIE's institutional memory.

### What Use It Provides

- **Automatic entity upserts** — new entities created, existing entities updated with co-occurrence counts
- **Relationship creation** — entities mentioned in the same event are connected
- **Event nodes** — each event becomes a node, linked to all entities mentioned in it
- **Idempotent** — processing the same event twice doesn't create duplicates (MERGE queries)

### How It Is Built

```python
class GraphWriter:
    def __init__(self, neo4j_driver):
        self.driver = neo4j_driver

    async def write_event(self, event):
        async with self.driver.session() as session:
            # Create event node
            await session.run("""
                MERGE (e:Event {id: $id})
                SET e.title = $title, e.summary = $summary,
                    e.category = $category, e.threat_level = $threat_level,
                    e.timestamp = datetime($timestamp)
            """, **event)

            # Create entity nodes and relationships
            for entity in event["entities"]:
                label = self._neo4j_label(entity["label"])
                await session.run(f"""
                    MERGE (n:{label} {{name: $name}})
                    WITH n
                    MATCH (e:Event {{id: $event_id}})
                    MERGE (n)-[:MENTIONED_IN {{confidence: $confidence}}]->(e)
                """, name=entity["text"], event_id=event["id"],
                   confidence=event.get("classification_confidence", 0.8))

            # Create co-occurrence edges between entities in same event
            entities = event["entities"]
            for i in range(len(entities)):
                for j in range(i+1, len(entities)):
                    await session.run("""
                        MATCH (a {name: $name_a}), (b {name: $name_b})
                        MERGE (a)-[r:RELATED_TO]-(b)
                        ON CREATE SET r.co_occurrence_count = 1
                        ON MATCH SET r.co_occurrence_count = r.co_occurrence_count + 1
                    """, name_a=entities[i]["text"], name_b=entities[j]["text"])
```

---

## F23 — GraphRAG Query Engine

### What Is Being Built

A retrieval-augmented generation (RAG) engine that combines Neo4j graph traversal, Qdrant vector similarity search, and Claude API generation to answer natural-language intelligence queries with citations. It acts as BIE's "intelligence analyst" — grounding LLM answers in real data from the knowledge graph.

### Why It Is Needed

An LLM without grounding hallucinates. A knowledge graph without an LLM is hard to query. GraphRAG combines both: user asks a question → graph provides relevant facts → LLM synthesizes an answer citing those facts. This is BIE's most impressive demo feature.

### What Use It Provides

- **Natural language queries** — "What is China doing in the Indian Ocean this week?"
- **Graph-grounded answers** — every claim backed by graph data and events
- **Citations** — clickable links to source events
- **Confidence score** — how well the graph supports the answer
- **Streaming** — response appears word-by-word for engagement
- **Fallback** — if Neo4j is down, falls back to Qdrant-only similarity search

### How It Is Built

1. **Query → Graph traversal → LLM**:
   ```python
   class GraphRAG:
       async def query(self, question: str) -> dict:
           # Step 1: Extract entities from question
           entities = self.nlp.extract_entities(question)

           # Step 2: Traverse graph for context
           graph_context = await self._traverse(entities)

           # Step 3: Get similar events from Qdrant
           similar_events = await self.qdrant.search(question, limit=10)

           # Step 4: Generate answer with Claude
           prompt = self._build_prompt(question, graph_context, similar_events)
           answer = await self.claude.generate(prompt, stream=True)

           return {
               "answer": answer,
               "citations": self._extract_citations(similar_events),
               "confidence": self._calculate_confidence(graph_context, similar_events),
               "graph_context": graph_context,
           }
   ```

2. **Claude system prompt** — tuned for Indian strategic analysis:
   ```
   You are an intelligence analyst advising Indian strategic leadership on global security
   developments. Answer questions using ONLY the provided graph context and event data.
   Cite specific events by ID. Assess the India impact of every finding.
   If the evidence is insufficient, say so — never fabricate.
   Focus on: how does this affect India's national security, diplomatic interests, or
   economic partnerships?
   ```

3. **Streaming**: response streamed via FastAPI's `StreamingResponse` to the frontend.

4. **Caching**: Claude responses cached in Redis with 24h TTL, keyed by MD5(prompt).

---

## F24 — Qdrant Vector Store

### What Is Being Built

A vector similarity search engine using Qdrant that stores embeddings of all processed events. Enables semantic search — finding events related to a query even without exact keyword matches.

### Why It Is Needed

Keyword search fails on paraphrased text. "Chinese military activity near Taiwan" should match "PLA exercises in the Taiwan Strait" even though they share few keywords. Vector embeddings capture semantic meaning, enabling fuzzy, intelligent search.

### What Use It Provides

- **Semantic similarity** — "BRI port investment" matches "Chinese-funded harbor development"
- **GraphRAG support** — provides relevant events to the RAG pipeline
- **Fallback search** — when Neo4j is unavailable, Qdrant alone can power basic Q&A
- **Filtered search** — combine vector similarity with metadata filters (category, region, threat level)

### How It Is Built

1. **Collection** — `bie_events`:
   ```python
   qdrant.create_collection(
       collection_name="bie_events",
       vectors_config=VectorParams(size=384, distance=Distance.COSINE),
   )
   ```

2. **Embedding model**: `all-MiniLM-L6-v2` (384 dimensions, fast, accurate for English text).

3. **Indexing**: every processed event is embedded and stored:
   ```python
   async def index_event(self, event):
       embedding = self.model.encode(f"{event['title']} {event['summary']}")
       self.qdrant.upsert(
           collection_name="bie_events",
           points=[PointStruct(
               id=event["id"],
               vector=embedding.tolist(),
               payload={
                   "title": event["title"],
                   "category": event["category"],
                   "threat_level": event["threat_level"],
                   "region": event.get("geo_focus", "GLOBAL"),
                   "timestamp": event["timestamp"],
                   "india_relevance": event.get("india_relevance", 0.5),
               }
           )]
       )
   ```

4. **Search**:
   ```python
   async def search(self, query, limit=10, filters=None):
       embedding = self.model.encode(query)
       return self.qdrant.search(
           collection_name="bie_events",
           query_vector=embedding.tolist(),
           query_filter=filters,
           limit=limit,
       )
   ```

---

← Back to [prd.md](./prd.md)
