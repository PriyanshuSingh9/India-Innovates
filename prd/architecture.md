# Architecture & Foundation

> Covers: F01 Monorepo Initialization · F02 API Contract & Shared Schemas · F15 Week 1 Integration Test

---

## F01 — Monorepo Initialization

### What Is Being Built

A unified monorepo structure that houses the entire BIE platform — frontend, backend, AI/ML services, and shared configuration — in a single Git repository with clear directory boundaries.

```
india-innovates/
├── frontend/          # Next.js 16 application
├── backend/           # FastAPI Python application
│   ├── app/
│   │   ├── main.py
│   │   ├── routers/
│   │   ├── services/
│   │   ├── models/
│   │   └── workers/
│   ├── requirements.txt
│   └── .env.example
├── shared/            # Shared schemas, constants, types
│   └── schemas/
├── data/              # Seed data, GeoJSON, hotspot databases
├── docs/              # Documentation, demo scripts
├── prd/               # Product requirements (this folder)
├── .gitignore
├── README.md
└── LICENSE
```

### Why It Is Needed

BIE has four engineering roles (BE1, FE1, FE2, AI1, AI2, AI3) working simultaneously across Python and JavaScript. Without a monorepo:

- **API contracts drift** — backend and frontend evolve independently, breaking integrations every merge
- **Shared types are duplicated** — the same `Event` schema gets defined differently in Python and TypeScript
- **CI/CD is fragmented** — separate repos mean separate pipelines, making integration testing impossible without cross-repo orchestration
- **Onboarding is slow** — new contributors must clone multiple repos, configure multiple environments

### What Use It Provides

- **Single clone** — `git clone` gives every engineer the entire platform
- **Atomic commits** — a backend schema change and the corresponding frontend update ship in one commit
- **Shared constants** — region codes, event categories, and API paths live in one place
- **Unified CI** — a single GitHub Actions workflow can lint, test, and build all subsystems

### How It Is Built

1. **Initialize the repo** with the directory structure above
2. **Frontend**: `npx create-next-app@latest ./frontend` with App Router, Tailwind CSS v4, ESLint
3. **Backend**: create a Python virtual environment (`python -m venv .venv`), install FastAPI + Uvicorn, scaffold `main.py` with a health-check endpoint
4. **Shared schemas**: create `shared/schemas/` with JSON Schema files that both frontend (via TypeScript codegen) and backend (via Pydantic) can consume
5. **Scripts**: add `Makefile` or `scripts/` with commands:
   - `make dev-frontend` → `cd frontend && npm run dev`
   - `make dev-backend` → `cd backend && uvicorn app.main:app --reload`
   - `make dev` → runs both in parallel
6. **Git hooks**: add pre-commit hook that runs linting on staged files

---

## F02 — API Contract & Shared Schemas

### What Is Being Built

A single source-of-truth API contract that defines every endpoint, request/response shape, and data model shared between the frontend and backend. This contract is enforced through Pydantic models on the backend and TypeScript types on the frontend, all derived from a shared schema definition.

### Why It Is Needed

BIE has six engineers making parallel changes. Without a locked API contract:

- **Frontend builds against imagined endpoints** that don't match what backend actually ships
- **Field names diverge** — backend uses `threat_level`, frontend uses `threatLevel`, causing silent runtime failures
- **Breaking changes go unnoticed** — a backend engineer renames a field and the frontend breaks in production
- **Integration testing is impossible** — there's no baseline to test against

The roadmap explicitly identifies "FE/BE API contract drift" as a **High** risk, and designates BE1 as the owner of `/backend/schemas.py` with the rule: "Any change requires team announcement and FE migration."

### What Use It Provides

- **Single source of truth** — `schemas.py` is the canonical definition; everything else is derived
- **Type safety end-to-end** — Python gets Pydantic validation, TypeScript gets generated types
- **Self-documenting API** — FastAPI auto-generates OpenAPI (Swagger) docs from the Pydantic models
- **Contract tests** — CI can verify that frontend mock data matches backend response shapes

### How It Is Built

1. **Define Pydantic models** in `backend/app/models/schemas.py`:

   ```python
   from pydantic import BaseModel
   from datetime import datetime
   from typing import Optional

   class GeoLocation(BaseModel):
       lat: float
       lng: float

   class Event(BaseModel):
       id: str
       title: str
       summary: str
       source: str
       category: str  # MILITARY | DIPLOMATIC | ECONOMIC | INTERNAL | MARITIME
       threat_level: str  # LOW | MEDIUM | HIGH | CRITICAL
       location: GeoLocation
       entities: list[str]
       timestamp: datetime
       raw_url: Optional[str] = None

   class FeedItem(BaseModel):
       id: str
       source_name: str
       title: str
       content: str
       published_at: datetime
       ingested_at: datetime
       processed: bool = False

   class QueryResponse(BaseModel):
       answer: str
       citations: list[dict]
       confidence: float
       graph_context: Optional[dict] = None

   class StrategicBrief(BaseModel):
       id: str
       generated_at: datetime
       period_start: datetime
       period_end: datetime
       sections: list[dict]
       instability_scores: dict[str, float]

   class InstabilityScore(BaseModel):
       region: str
       score: float  # 0.0 to 1.0
       trend: str  # RISING | STABLE | FALLING
       contributing_factors: list[str]
       last_updated: datetime
   ```

2. **Export OpenAPI spec** from FastAPI automatically — available at `/docs` and `/openapi.json`

3. **Generate TypeScript types** using `openapi-typescript` or a similar tool:
   ```bash
   npx openapi-typescript http://localhost:8000/openapi.json -o frontend/types/api.ts
   ```

4. **CI check**: add a GitHub Actions step that regenerates types and fails if the output differs from what's committed (drift detection)

5. **Versioning**: API paths include no version prefix for v1 (implied). If breaking changes are needed, create `/v2/` routes alongside existing ones.

---

## F15 — Week 1 Integration Test

### What Is Being Built

A comprehensive end-to-end integration test suite that validates the full data flow from ingestion through processing to display across all subsystems built in Week 1 (F01–F14).

### Why It Is Needed

By the end of Week 1, BIE has:
- A frontend with a map, layers, markers, and a news feed panel
- A backend with feed ingestion, NLP stubs, and event endpoints
- A Global Strategic Hotspot Database covering 500+ locations

These are built by different engineers. Without an integration test at this boundary:
- **Silent failures** go undetected — the RSS worker ingests data, but the frontend panel shows nothing because field names don't match
- **Geo-location mismatches** — markers appear in the wrong location because coordinate systems differ
- **Performance blind spots** — the map may be loading 10,000 markers when it can only handle 500

### What Use It Provides

- **Confidence checkpoint** — the team knows Week 1's foundation is solid before building Week 2's intelligence layer on top
- **Regression baseline** — future changes can't silently break the ingestion → display pipeline
- **Demo rehearsal** — the Week 1 internal demo ("3-min walk-through, document all gaps") has a checklist to validate

### How It Is Built

1. **Backend integration tests** (pytest):
   - Test that `/api/events` returns valid `Event` objects matching the schema
   - Test that `/api/feeds` returns `FeedItem` objects with correct timestamps
   - Test that the RSS ingestion worker can fetch, parse, and store a known RSS feed
   - Test that health-check endpoints return 200 OK

2. **Frontend integration tests** (Playwright or Cypress):
   - Load the map → verify MapLibre canvas renders
   - Verify strategic geo layers (LAC, LOC, chokepoints, BRI corridors) load without errors
   - Verify regional presets (12+ global presets) change the map viewport correctly
   - Verify the news feed panel displays items from the backend
   - Verify clicking a map marker opens the detail panel

3. **End-to-end smoke test**:
   - Inject a test RSS item → wait for ingestion → verify it appears in the news feed panel and as a map marker within 2 minutes

4. **Execution**: run all tests with `make test-integration` which starts both backend and frontend, runs the test suite, and tears down

5. **CI**: integration tests run on every PR targeting `main`, blocking merge on failure

---

← Back to [prd.md](./prd.md)
