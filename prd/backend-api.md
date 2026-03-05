# Backend API

> Covers: F03 FastAPI Foundation · F11 Backend Event/Feed Endpoints · F35 Caching Architecture · F36 Error Handling & Degraded Mode

---

## F03 — FastAPI Foundation

### What Is Being Built

The core backend application skeleton using FastAPI — Python's fastest async web framework. This provides all API stubs that the frontend and AI services will call, a health-check system, CORS configuration, and request logging middleware.

### Why It Is Needed

BIE's backend is the **central nervous system**: every data flow passes through it. The frontend fetches events, the NLP workers post processed data, the GraphRAG engine queries the graph — all through FastAPI endpoints. Without a solid foundation from Day 1:

- **The frontend team is blocked** — they can't build panels without endpoints to call (even stubs)
- **AI workers have no integration target** — processed NLP results have nowhere to go
- **There's no observability** — without structured logging, debugging production issues during the demo is impossible

### What Use It Provides

- **Immediate unblocking** — stub endpoints return mock data, letting FE1 and FE2 build UI from Day 1
- **Async by default** — FastAPI's async handlers support concurrent connections during demo
- **Auto-documentation** — Swagger UI at `/docs` lets every engineer explore available endpoints
- **Middleware pipeline** — request logging, CORS, and rate limiting are configured once, applied everywhere

### How It Is Built

1. **Project setup**:
   ```bash
   cd backend
   python -m venv .venv
   source .venv/bin/activate
   pip install fastapi uvicorn pydantic python-dotenv
   ```

2. **Application scaffold** — `backend/app/main.py`:
   ```python
   from fastapi import FastAPI
   from fastapi.middleware.cors import CORSMiddleware
   import logging, time, uuid

   app = FastAPI(title="BIE Backend", version="0.1.0")

   app.add_middleware(
       CORSMiddleware,
       allow_origins=["http://localhost:3000"],  # Lock to Vercel domain in prod
       allow_methods=["*"],
       allow_headers=["*"],
   )

   @app.middleware("http")
   async def log_requests(request, call_next):
       request_id = str(uuid.uuid4())[:8]
       start = time.time()
       response = await call_next(request)
       duration = time.time() - start
       logging.info(f"[{request_id}] {request.method} {request.url.path} → {response.status_code} ({duration:.2f}s)")
       return response

   @app.get("/health")
   async def health():
       return {"status": "ok", "service": "bie-backend"}
   ```

3. **Router structure**:
   - `routers/events.py` — `/api/events`, `/api/events/{id}`
   - `routers/feeds.py` — `/api/feeds`, `/api/feeds/sources`
   - `routers/query.py` — `/api/query`
   - `routers/brief.py` — `/api/brief`, `/api/brief/latest`
   - `routers/instability.py` — `/api/instability`, `/api/instability/{region}`
   - `routers/graph.py` — `/api/graph/search`, `/api/graph/entity/{id}`

4. **Stub pattern** — every router starts by returning mock data:
   ```python
   @router.get("/api/events")
   async def get_events():
       return [MOCK_EVENT_1, MOCK_EVENT_2]  # Real implementation in F11
   ```

5. **Run**: `uvicorn app.main:app --reload --port 8000`

---

## F11 — Backend Event/Feed Endpoints

### What Is Being Built

The real (non-stub) implementations of the event and feed API endpoints. These replace the mock data from F03 with live queries against the ingested and processed data stores, supporting filtering, pagination, geo-queries, and real-time updates via SSE (Server-Sent Events).

### Why It Is Needed

Once the RSS ingestion worker (F09) and external API workers (F10) are running, processed events need to flow to the frontend. The stub endpoints from F03 return static mock data — fine for initial UI development, but the demo requires live data. These endpoints are the **bridge between ingestion and display**.

The critical path analysis identifies this feature as central: `F02 → F03 → F11 → F12/F13` — the FE team's news feed panel (F12) and map markers (F13) both depend on real endpoints.

### What Use It Provides

- **Live event feed** — the news panel shows real ingested events, not mock data
- **Geo-filtered events** — the map can request only events within its current viewport (bounding box)
- **Category filtering** — filter by `MILITARY`, `DIPLOMATIC`, `ECONOMIC`, `INTERNAL`, `MARITIME`, `CYBER`, `TERRORISM`
- **Threat-level filtering** — show only `HIGH` or `CRITICAL` events
- **Region filtering** — filter by strategic region (e.g., `SOUTH_ASIA`, `INDO_PACIFIC`, `MIDDLE_EAST`)
- **Country filtering** — filter by ISO country code (e.g., `CN`, `PK`, `AF`)
- **India-relevance filtering** — filter by `india_relevance` threshold (e.g., `>0.5`)
- **Pagination** — efficient cursor-based pagination for large result sets
- **Real-time streaming** — SSE endpoint pushes new events to connected clients without polling

### How It Is Built

1. **Event endpoints**:

   | Method | Path | Description |
   |---|---|---|
   | `GET` | `/api/events` | List events, supports `?category=`, `?threat_level=`, `?region=`, `?country=`, `?india_relevance_min=`, `?bbox=`, `?limit=`, `?cursor=` |
   | `GET` | `/api/events/{id}` | Get a single event with full detail including entities, proximity analysis |
   | `GET` | `/api/events/stream` | SSE stream — pushes new events as they arrive |
   | `GET` | `/api/events/stats` | Aggregated stats: event counts by category, threat level, region |

2. **Feed endpoints**:

   | Method | Path | Description |
   |---|---|---|
   | `GET` | `/api/feeds` | List all active feed sources and their status |
   | `GET` | `/api/feeds/items` | Recent feed items, supports `?source=`, `?limit=` |
   | `POST` | `/api/feeds/refresh` | Manually trigger a feed refresh cycle |

3. **Bounding-box geo-query** for viewport filtering:
   ```python
   @router.get("/api/events")
   async def get_events(
       bbox: Optional[str] = None,  # "min_lng,min_lat,max_lng,max_lat"
       category: Optional[str] = None,
       threat_level: Optional[str] = None,
       region: Optional[str] = None,
       country: Optional[str] = None,
       india_relevance_min: Optional[float] = None,
       limit: int = 50,
       cursor: Optional[str] = None,
   ):
       # Parse bbox, build query, apply filters, paginate
       ...
   ```

4. **SSE streaming** using `sse-starlette`:
   ```python
   from sse_starlette.sse import EventSourceResponse

   @router.get("/api/events/stream")
   async def event_stream():
       async def generate():
           async for event in redis_stream.listen("events:new"):
               yield {"data": event.json()}
       return EventSourceResponse(generate())
   ```

5. **Data source**: events are stored in Redis Streams (for real-time) and optionally persist to a lightweight store for historical queries.

---

## F35 — Caching Architecture

### What Is Being Built

A multi-layer caching system using Redis that prevents redundant API calls, reduces LLM costs, and ensures the demo runs smoothly even under load. Caching is applied at three levels: response caching, data caching, and LLM response caching.

### Why It Is Needed

BIE makes expensive external calls:
- **Claude API** for brief generation and GraphRAG answers ($$$)
- **Neo4j** for graph traversals (latency)
- **Qdrant** for vector similarity searches (compute)
- **External RSS/API feeds** (rate limits, latency)

Without caching:
- **Demo costs explode** — every query to the NL interface calls Claude API. 20 demo queries × $0.50 = $10. But if the same demo questions are asked across rehearsals and the live demo, costs multiply 5× unnecessarily.
- **Response latency spikes** — GraphRAG traversals take 2-5 seconds. Caching reduces repeat queries to ~10ms.
- **Rate limits hit** — external APIs (AIS, weather) have daily quotas. Re-fetching the same data wastes quota.

The risk register flags "Claude API costs spike" as a Medium risk, with the mitigation: "Cache all responses 24h. Set $50 daily spend cap."

### What Use It Provides

- **Sub-50ms response times** for cached queries during the demo
- **24-hour response caching** for Claude API calls — same question = cached answer
- **Feed deduplication** — ingested items are fingerprinted; duplicates are skipped
- **Graceful degradation** — if Redis is down, the system falls back to direct queries (slower, costlier, but functional)

### How It Is Built

1. **Redis setup**: Redis instance on Railway, connected via `redis-py` async client.

2. **Three cache layers**:

   | Layer | TTL | What's Cached |
   |---|---|---|
   | **API Response Cache** | 60s | Event list responses, instability scores |
   | **Data Cache** | 5 min | Feed items, graph query results |
   | **LLM Response Cache** | 24h | Claude API responses keyed by query hash |

3. **Cache key strategy**:
   ```python
   import hashlib

   def cache_key(prefix: str, params: dict) -> str:
       param_str = json.dumps(params, sort_keys=True)
       hash = hashlib.md5(param_str.encode()).hexdigest()
       return f"bie:{prefix}:{hash}"
   ```

4. **Cache decorator**:
   ```python
   def cached(prefix: str, ttl: int = 60):
       def decorator(func):
           async def wrapper(*args, **kwargs):
               key = cache_key(prefix, kwargs)
               cached = await redis.get(key)
               if cached:
                   return json.loads(cached)
               result = await func(*args, **kwargs)
               await redis.setex(key, ttl, json.dumps(result))
               return result
           return wrapper
       return decorator
   ```

5. **Cache invalidation**: Manual invalidation via admin endpoint `POST /api/cache/clear?prefix=events`. Auto-invalidation on new event ingestion for event-related caches.

---

## F36 — Error Handling & Degraded Mode

### What Is Being Built

A comprehensive error handling framework that ensures BIE never shows a blank screen or crashes during the demo. When a service fails (Neo4j down, Claude API timeout, RSS feed unreachable), the system degrades gracefully — showing stale data, fallback answers, or informative status messages instead of errors.

### Why It Is Needed

BIE depends on multiple external services (Claude API, RSS feeds, AIS data) and internal services (Neo4j, Qdrant, Redis). During a live demo, any of these can fail:

- **Internet drops** — Claude API and external feeds become unreachable
- **Railway service crashes** — backend containers restart, causing 30-second outages
- **Neo4j timeout** — complex graph traversals exceed timeout limits
- **Rate limits** — Claude API or external feeds throttle requests

The risk register identifies "Railway service crashes mid-demo" as a **High** risk. The mitigation is architectural: auto-restart + degraded mode.

### What Use It Provides

- **No blank screens** — every failure case has a fallback UI state
- **Stale data > no data** — cached responses are served even if stale, with a "last updated" indicator
- **Service health dashboard** — the frontend shows which services are up/down
- **Automatic recovery** — when a failed service comes back, the system automatically reconnects

### How It Is Built

1. **Service health registry**:
   ```python
   class ServiceHealth:
       services = {
           "neo4j": {"status": "unknown", "last_check": None},
           "qdrant": {"status": "unknown", "last_check": None},
           "redis": {"status": "unknown", "last_check": None},
           "claude": {"status": "unknown", "last_check": None},
       }

       async def check_all(self):
           for name, config in self.services.items():
               try:
                   await self._ping(name)
                   config["status"] = "healthy"
               except Exception:
                   config["status"] = "degraded"
               config["last_check"] = datetime.utcnow()
   ```

2. **Fallback chain** for each capability:

   | Capability | Primary | Fallback 1 | Fallback 2 |
   |---|---|---|---|
   | NL Query | GraphRAG (Claude + Neo4j) | Qdrant `/similar` | "Service temporarily unavailable" |
   | Strategic Brief | Claude API generation | Cached last brief | Static template brief |
   | Event Feed | Live RSS ingestion | Cached events | Pre-seeded demo events |
   | Instability Index | Real-time calculation | Cached scores | Hardcoded baseline scores |

3. **Circuit breaker pattern**: After 3 consecutive failures to a service, stop calling it for 60 seconds (prevents cascading timeouts):
   ```python
   class CircuitBreaker:
       def __init__(self, failure_threshold=3, reset_timeout=60):
           self.failures = 0
           self.threshold = failure_threshold
           self.reset_timeout = reset_timeout
           self.last_failure = None
           self.state = "closed"  # closed = normal, open = failing

       async def call(self, func, *args, **kwargs):
           if self.state == "open":
               if time.time() - self.last_failure > self.reset_timeout:
                   self.state = "half-open"
               else:
                   raise CircuitOpenError()
           try:
               result = await func(*args, **kwargs)
               self.failures = 0
               self.state = "closed"
               return result
           except Exception as e:
               self.failures += 1
               self.last_failure = time.time()
               if self.failures >= self.threshold:
                   self.state = "open"
               raise
   ```

4. **Health endpoint**: `GET /api/health/detailed` returns the status of every service — consumed by the frontend's status indicator.

5. **Frontend integration**: the frontend shows a subtle status bar. Green = all services healthy. Yellow = degraded (some cached data). Red = critical (core features unavailable).

---

← Back to [prd.md](./prd.md)
