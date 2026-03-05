# UI Panels

> Covers: F12 News Feed Panel · F28 NL Query Interface · F29 Knowledge Graph Panel · F32 Strategic Posture Panel · F34 IOR Theater Dashboard

---

## F12 — News Feed Panel

### What Is Being Built

A real-time scrolling news feed panel on the left side of the BIE interface. It displays ingested and NLP-classified intelligence items from global sources in reverse chronological order, with filtering by category, threat level, source region, and India-relevance. Clicking an item flies the map to the event's location and opens its detail panel.

### Why It Is Needed

The map shows *where* events are; the news feed shows *what* they are. Together they create the core BIE experience: a spatial + textual intelligence view. Analysts need to scan global events quickly, spot patterns by category, and drill into specific items.

### What Use It Provides

- **Real-time global feed** — new items appear at the top as they are ingested (via SSE)
- **Category badges** — MILITARY (red), DIPLOMATIC (blue), ECONOMIC (green), INTERNAL (orange), MARITIME (teal), CYBER (purple), TERRORISM (dark red)
- **Threat-level indicators** — colored dot next to each item (green/yellow/red/pulsing red)
- **Source attribution** — "Reuters", "PTI", "SCMP", "Al Jazeera", "Defense One", "BBC Hindi" labels
- **Region tags** — "SOUTH_ASIA", "INDO_PACIFIC", "MIDDLE_EAST" region badges
- **India-relevance indicator** — 🇮🇳 flag icon on events with India-relevance score > 0.5
- **Click-to-fly** — clicking an item triggers `map.flyTo()` to the event's coordinates
- **Filter bar** — filter by category, threat level, region, source, India-relevance threshold
- **Infinite scroll** — paginated loading via cursor-based API

### How It Is Built

1. **Component structure**:
   ```
   components/
   ├── NewsFeedPanel/
   │   ├── NewsFeedPanel.js     — Container with filter bar + list
   │   ├── FeedItem.js          — Individual news item card
   │   ├── FeedFilter.js        — Filter controls (category, threat, source, region)
   │   └── feedPanel.css        — Panel-specific styles
   ```

2. **Data flow**:
   - Initial load: `GET /api/feeds/items?limit=50`
   - Real-time updates: subscribe to `EventSource("/api/events/stream")`
   - New items are prepended to the list with a slide-in animation

3. **FeedItem component**:
   ```jsx
   function FeedItem({ item, onSelect }) {
     return (
       <div className="feed-item" onClick={() => onSelect(item)}>
         <div className="feed-item-header">
           <span className={`badge badge-${item.category.toLowerCase()}`}>
             {item.category}
           </span>
           <span className={`threat-dot threat-${item.threat_level.toLowerCase()}`} />
           {item.india_relevance > 0.5 && <span className="india-flag">🇮🇳</span>}
           <span className="feed-time">{timeAgo(item.timestamp)}</span>
         </div>
         <h4 className="feed-title">{item.title}</h4>
         <p className="feed-summary">{item.summary}</p>
         <div className="feed-meta">
           <span className="feed-source">{item.source}</span>
           <span className="feed-region">{item.geo_focus}</span>
         </div>
       </div>
     );
   }
   ```

4. **Panel layout**: fixed-width left panel (380px), full height, scrollable. Dark background matching the map theme. Semi-transparent with glassmorphism effect.

---

## F28 — NL Query Interface

### What Is Being Built

A natural-language query bar where analysts type intelligence questions in plain English. The query is processed by the GraphRAG engine (F23), which searches the Neo4j knowledge graph and Qdrant vector store, then generates a streaming LLM response grounded in real data, with citations.

### Why It Is Needed

Traditional intelligence systems require analysts to write SQL queries or navigate complex filter UIs. BIE lets them ask questions like a conversation:

- *"What is China doing in the Indian Ocean this week?"*
- *"How is the South China Sea situation affecting QUAD dynamics?"*
- *"Show me all recent events near BRI ports in the past week"*
- *"What is Pakistan's military posture along the LOC?"*
- *"How do Red Sea disruptions impact India's energy security?"*

### What Use It Provides

- **Natural language input** — type questions in plain English, no query language needed
- **Streaming response** — answer appears word-by-word, like ChatGPT, maintaining engagement
- **Citations** — every factual claim links back to its source event or graph entity
- **India-focused analysis** — answers always contextualize findings for India's strategic interests
- **Confidence score** — shows how confident the system is in the answer (< 0.6 triggers a disclaimer)
- **Graph context** — optionally shows the subgraph traversed to generate the answer
- **Query history** — recent queries are saved for quick re-access

### How It Is Built

1. **Query bar UI** — centered at the top of the map:
   ```jsx
   function QueryBar({ onSubmit }) {
     const [query, setQuery] = useState("");
     return (
       <form onSubmit={(e) => { e.preventDefault(); onSubmit(query); }}>
         <input
           type="text"
           value={query}
           onChange={(e) => setQuery(e.target.value)}
           placeholder="Ask BIE about global security developments..."
           className="query-input"
         />
         <button type="submit" className="query-submit">Ask</button>
       </form>
     );
   }
   ```

2. **Streaming response** — use `fetch` with `ReadableStream`:
   ```javascript
   async function streamQuery(query, onChunk) {
     const response = await fetch("/api/query", {
       method: "POST",
       body: JSON.stringify({ query }),
       headers: { "Content-Type": "application/json" },
     });
     const reader = response.body.getReader();
     const decoder = new TextDecoder();
     while (true) {
       const { done, value } = await reader.read();
       if (done) break;
       onChunk(decoder.decode(value));
     }
   }
   ```

3. **Response panel** — displays below the query bar:
   - Streaming text with markdown rendering
   - Citation chips: clickable, fly-to the cited event on the map
   - Confidence bar: colored green (>0.8), yellow (0.6-0.8), red (<0.6 with ⚠️ disclaimer)
   - "Show graph context" toggle: reveals the Neo4j subgraph used

4. **Fallback**: if GraphRAG fails (Neo4j down, Claude timeout), fall back to Qdrant `/similar` endpoint — returns semantically similar events without LLM synthesis, with a message: "Full analysis unavailable. Showing related events."

---

## F29 — Knowledge Graph Panel

### What Is Being Built

A visual panel that renders a subset of the Neo4j knowledge graph as an interactive node-link diagram. Users can explore entities (people, organizations, countries, locations, alliances, military units), see their relationships, and click nodes to drill into detail.

### Why It Is Needed

The knowledge graph is BIE's "brain" — it stores relationships between entities extracted from global events. But a graph database is useless if analysts can't see and explore it. The KG panel makes the invisible visible:
- Which Chinese organizations are active near India's borders?
- What connects Gwadar Port to the PLA Navy?
- How are QUAD countries responding to SCS developments?

### What Use It Provides

- **Entity visualization** — nodes colored by type (person=blue, org=red, country=gold, location=green, event=yellow, alliance=purple)
- **Relationship edges** — labeled connections (MENTIONED_IN, OPERATES_IN, MEMBER_OF, ALLIES_WITH)
- **Interactive exploration** — click a node to expand its connections; double-click to center the graph on it
- **Search** — type an entity name to find and highlight it in the graph
- **Context-aware** — when viewing an event, the KG panel shows the subgraph related to that event

### How It Is Built

1. **Visualization library**: `react-force-graph-2d` for interactive graph rendering.

2. **Data source**: `GET /api/graph/search?entity={name}` returns a subgraph:
   ```json
   {
     "nodes": [
       { "id": "org_pla", "label": "PLA", "type": "organization" },
       { "id": "loc_djibouti", "label": "Djibouti Base", "type": "infrastructure" },
       { "id": "country_cn", "label": "China", "type": "country" },
       { "id": "alliance_sco", "label": "SCO", "type": "alliance" }
     ],
     "edges": [
       { "source": "org_pla", "target": "loc_djibouti", "label": "OPERATES" },
       { "source": "country_cn", "target": "alliance_sco", "label": "MEMBER_OF" }
     ]
   }
   ```

3. **Panel placement**: right-side collapsible panel, below the query interface.

---

## F32 — Strategic Posture Panel

### What Is Being Built

A dashboard panel showing the current global strategic posture from India's perspective. It displays the overall threat level, per-theater BII scores, active anomalies, convergence alerts, and India-neighborhood status. It answers: "How worried should Indian strategic leadership be right now?"

### Why It Is Needed

The map shows *geography*, the feed shows *events*, but the posture panel shows *assessment*. It transforms raw data into an actionable threat picture — the kind of summary a National Security Advisor would want on their desk.

### What Use It Provides

- **Overall posture indicator** — NORMAL / ELEVATED / HIGH / CRITICAL with color coding
- **Per-theater scores** — bar chart showing BII scores for each strategic theater, sorted by severity
- **India Neighborhood status** — dedicated section for LAC, LOC, IOR, Northeast
- **Active alerts** — list of current anomaly and convergence alerts with timestamps
- **Trend indicators** — ↑ RISING / → STABLE / ↓ FALLING per theater
- **Contributing factors** — for each high-scoring theater, list the events driving the score
- **Historical mini-chart** — sparkline showing posture over the last 24 hours

### How It Is Built

1. **Data source**:
   - `GET /api/instability` — per-region BII scores
   - `GET /api/instability/posture` — overall posture level
   - `GET /api/anomalies/active` — current anomaly alerts
   - `GET /api/convergence/active` — current convergence alerts

2. **Posture level logic**:
   ```
   NORMAL:   all regions < 0.4
   ELEVATED: any region 0.4–0.6
   HIGH:     any region 0.6–0.8
   CRITICAL: any region > 0.8 OR convergence alert active
   ```

3. **Theater grouping** for the posture panel:
   ```javascript
   const THEATER_GROUPS = {
     "India Neighborhood": ["LAC_WEST", "LAC_EAST", "LOC_KASHMIR", "SIACHEN", "NORTHEAST"],
     "Indian Ocean": ["IOR"],
     "Indo-Pacific": ["SOUTH_CHINA_SEA", "TAIWAN_STRAIT", "SOUTHEAST_ASIA"],
     "Middle East": ["PERSIAN_GULF", "RED_SEA"],
     "Extended": ["HORN_OF_AFRICA", "CENTRAL_ASIA", "EUROPE"],
   };
   ```

4. **Real-time updates**: poll every 30 seconds. On posture level change, trigger a subtle pulse animation on the badge.

---

## F34 — IOR Theater Dashboard

### What Is Being Built

A specialized sub-dashboard focused on the Indian Ocean Region (IOR). It combines vessel tracking data (AIS), maritime event markers, chokepoint monitoring, and Chinese naval presence indicators into a dedicated maritime intelligence view. This is positioned as one of several potential theater dashboards, showcasing what deep-dive analysis looks like.

### Why It Is Needed

India's maritime domain awareness is a critical national security priority. The IOR spans from the Strait of Hormuz to the Strait of Malacca — controlling supply routes for 80% of India's oil imports. China's expanding naval presence (String of Pearls strategy) makes IOR monitoring essential.

The IOR dashboard demonstrates BIE's ability to provide theater-specific deep dives — a template that can be replicated for other theaters (SCS, Gulf, etc.).

### What Use It Provides

- **Vessel positions** — real-time dots showing ship locations from AIS data
- **Vessel filtering** — filter by type (military, cargo, fishing), flag state (China, India, Pakistan)
- **Chokepoint monitoring** — traffic counts at Malacca, Hormuz, Bab el-Mandeb, Lombok
- **Chinese naval overlay** — highlighted markers for Chinese naval vessels and bases (Djibouti, Gwadar, Hambantota)
- **India's assets** — Indian naval facilities (INS Kadamba, Andaman & Nicobar Command, Agaléga)
- **Maritime events** — piracy, illegal fishing, unusual military movements
- **Region preset** — one click to switch to the IOR maritime view

### How It Is Built

1. **Data sources**:
   - AIS API worker (F10) for vessel positions
   - Maritime event markers from RSS feeds categorized as MARITIME
   - Static data for Chinese and Indian base locations

2. **Vessel layer** (Deck.gl IconLayer):
   ```javascript
   const vesselLayer = new IconLayer({
     id: "vessels",
     data: vessels,
     getPosition: (d) => [d.lng, d.lat],
     getIcon: (d) => vesselIcon(d.type, d.flag),
     getSize: 24,
     pickable: true,
     onClick: ({ object }) => setSelectedVessel(object),
   });
   ```

3. **Panel UI**: when IOR preset is active, a side panel shows:
   - Vessel count by flag state
   - Chokepoint traffic summary
   - Recent maritime events
   - BII score for the IOR theater

---

← Back to [prd.md](./prd.md)
