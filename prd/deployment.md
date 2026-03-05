# Deployment & Operations

> Covers: F40 Demo Environment · F41 Demo Script & Rehearsal · F42 Production Hardening

---

## F40 — Demo Environment

### What Is Being Built

A dedicated deployment environment (`bie-demo`) on Railway (backend) and Vercel (frontend), separate from the development environment, pre-seeded with realistic data and configured for reliable demo performance. This environment is frozen 48 hours before the demo.

### Why It Is Needed

Demoing from a development environment guarantees failure:
- Dev has unstable branches, half-finished features, debug logging
- Data is sparse and inconsistent
- Environment variables might point to test services
- A developer pushing broken code can crash the demo minutes beforehand

The demo environment is a **production clone** with:
- Pre-seeded Neo4j (1,000+ curated global strategic facts — military, diplomatic, alliances, infrastructure)
- Pre-embedded Qdrant (10,000 documents from global sources)
- Scripted demo inject endpoint for guaranteed convergence events
- Frozen deployment — no pushes after Day 19

### What Use It Provides

- **Stability** — isolated from dev work; nothing breaks it
- **Pre-seeded data** — graph is rich with global strategic facts, vector search returns strong results
- **Demo inject** — `POST /demo/inject` seeds scripted scenarios (LAC_CONVERGENCE, IOR_SURGE, DIPLOMATIC_CRISIS, SCS_ESCALATION, RED_SEA_CRISIS)
- **Budget safety** — Anthropic API key daily spend limit set to $50
- **Backup plan** — screen-recorded .mp4 for each capability if live demo fails

### How It Is Built

1. **Railway setup**: create a `bie-demo` environment in Railway, cloning the `main` branch config. Separate Redis, Neo4j, and Qdrant instances for demo.

2. **Vercel setup**: create a `demo` branch deployment on Vercel pointing to demo backend URL.

3. **Data seeding**:
   ```bash
   # Seed Neo4j with 1,000 facts
   python scripts/seed_neo4j.py --env demo

   # Pre-embed 10,000 documents into Qdrant
   python scripts/seed_qdrant.py --env demo --count 10000
   ```

4. **Demo inject endpoint**:
   ```python
   @router.post("/demo/inject")
   async def inject_scenario(scenario: str):
       SCENARIOS = {
           "LAC_CONVERGENCE": [...],      # Military + Diplomatic + Economic near Aksai Chin
           "IOR_SURGE": [...],             # Chinese naval movement in Indian Ocean
           "DIPLOMATIC_CRISIS": [...],     # Ambassador recall, sanctions, UN vote
           "SCS_ESCALATION": [...],        # PLA exercises near Taiwan, QUAD response
           "RED_SEA_CRISIS": [...],        # Houthi attack, shipping disruption, Indian Navy response
       }
       events = SCENARIOS.get(scenario, [])
       for event in events:
           await redis.xadd("feeds:raw", event)
       return {"injected": len(events), "scenario": scenario}
   ```

5. **Verification checklist** (48 hours before demo):
   - [ ] All 6 core endpoints return 200
   - [ ] Neo4j has > 1,000 nodes
   - [ ] Qdrant has > 10,000 embeddings
   - [ ] Demo inject produces convergence alert within 2 minutes
   - [ ] Strategic brief generates successfully
   - [ ] NL query returns cited answers
   - [ ] Map loads with markers and heatmap
   - [ ] Full 5-minute run-through on demo hardware

---

## F41 — Demo Script & Rehearsal

### What Is Being Built

A minute-by-minute presentation script documented in `/docs/DEMO_SCRIPT.md`, with designated roles, camera presets, exact queries to type, and fallback procedures. Two full rehearsals ensure the demo runs flawlessly.

### Why It Is Needed

A 5-minute demo can make or break the hackathon result. Improvised demos ramble, miss key features, and look unprepared. A scripted demo is:
- **Timed** — every second accounted for
- **Rehearsed** — rough edges fixed before the live event
- **Recoverable** — fallback owner knows every endpoint if something breaks

### What Use It Provides

- **Minute-by-minute script** — every click, typed query, and layer toggle documented
- **Designated presenter** — one person drives, no awkward handoffs
- **Backup person** — AI2 is technical fallback, knows every API
- **Two rehearsals** — Day 20 full rehearsal + Day 21 morning polish
- **Fallback recordings** — .mp4 backups of each capability

### How It Is Built

The demo script is documented in `/docs/DEMO_SCRIPT.md`:

| Minute | Action | What It Proves |
|---|---|---|
| 0:00–0:30 | Open BIE. Globe view — dark map centered on India at zoom 3.5. Toggle LAC, LOC, and chokepoint layers. Zoom into Aksai Chin. | Real geospatial interface, global scope with India focus. |
| 0:30–1:15 | Live news item arrives in feed (Reuters global wire). Click it. Map flies to event location (could be Middle East, SCS, etc.). Show India-relevance 🇮🇳 flag. | Real-time global ingestion. India-relevance scoring. |
| 1:15–2:00 | Click event marker. Show EventDetailPanel with entities (PLA units, weapon systems), proximity to strategic infrastructure (e.g., "45km from Gwadar Port"). | NLP extraction with strategic NER. Global infrastructure proximity. |
| 2:00–3:00 | Type query: "How is China's military posture in the South China Sea affecting QUAD dynamics?" Watch streaming answer with citations. | GraphRAG working. Global scope. India-impact analysis. |
| 3:00–3:45 | Convergence alert fires (demo inject LAC_CONVERGENCE). Posture panel escalates from ELEVATED to HIGH. Show BII scores across theaters. | Anomaly detection + convergence. Global BII with India-proximity weighting. |
| 3:45–4:30 | Click Generate Brief. Show "India Neighborhood Watch" and "Indo-Pacific Theater" sections generating. | Auto-generated global intelligence product with India-impact analysis. |
| 4:30–5:00 | Zoom to IOR. Show vessel tracker with AIS data from multiple chokepoints. Flash regional presets (Persian Gulf → SCS → Horn of Africa). Closing: "Palantir charges $50M. We built it in 3 weeks — for India." | The pitch. Global reach, India purpose. |

**Rehearsal schedule**:
- **Day 20 AM**: full rehearsal, entire team watches, gives feedback
- **Day 20 PM**: fix all rough edges identified in rehearsal
- **Day 21 AM**: second rehearsal, polish
- **Day 21 PM**: final deploy, freeze at 6 PM

**Hardware checklist**:
- Bring own laptop + HDMI adapter
- Test display resolution before event
- Phone hotspot as internet backup
- Pre-cache Claude responses in Redis for demo queries

---

## F42 — Production Hardening

### What Is Being Built

Security, reliability, and performance hardening for the demo deployment: API key management, rate limiting, CORS locking, SSL verification, structured logging, memory monitoring, and a tagged release.

### Why It Is Needed

A hackathon demo needs to survive 5 high-pressure minutes without crashing, leaking keys, or running up API costs. Production hardening is the difference between "it worked on my machine" and "it works in front of judges."

### What Use It Provides

- **Security**: all API keys in environment variables, zero in code or git history
- **Rate limiting**: prevents cost explosion (20 req/min on POST /query, 10 req/min on GET /brief)
- **CORS lock**: production Vercel domain only, dev wildcards removed
- **SSL**: HTTPS on both frontend (Vercel) and backend (Railway)
- **Logging**: structured JSON logs with request_id and duration for every request
- **Memory safety**: verified under 2-hour stress test; Railway plan scaled if needed
- **Tagged release**: git tag `v1.0`, deployed and smoke tested

### How It Is Built

1. **Environment variables**: Railway dashboard → Environment Variables for all secrets. Verify none in code:
   ```bash
   git log -p | grep -i "api_key\|secret\|password" # Should return nothing
   ```

2. **Rate limiting** with `slowapi`:
   ```python
   from slowapi import Limiter
   from slowapi.util import get_remote_address

   limiter = Limiter(key_func=get_remote_address)

   @app.post("/api/query")
   @limiter.limit("20/minute")
   async def query(request: Request, body: QueryRequest):
       ...

   @app.get("/api/brief")
   @limiter.limit("10/minute")
   async def get_brief(request: Request):
       ...
   ```

3. **CORS**: lock to production domain:
   ```python
   app.add_middleware(
       CORSMiddleware,
       allow_origins=["https://bie.vercel.app"],  # Production only
       allow_methods=["GET", "POST"],
       allow_headers=["*"],
   )
   ```

4. **Structured logging**:
   ```python
   import logging, json

   class JSONFormatter(logging.Formatter):
       def format(self, record):
           return json.dumps({
               "timestamp": self.formatTime(record),
               "level": record.levelname,
               "message": record.getMessage(),
               "request_id": getattr(record, "request_id", None),
           })
   ```

5. **Final deploy checklist**:
   - [ ] `git tag v1.0`
   - [ ] Deploy to demo environment
   - [ ] Smoke test all 6 core endpoints
   - [ ] Verify SSL on both frontend and backend
   - [ ] Confirm $50 daily spend cap on Anthropic
   - [ ] Run 2-hour stress test, check memory
   - [ ] Green across the board

---

← Back to [prd.md](./prd.md)
