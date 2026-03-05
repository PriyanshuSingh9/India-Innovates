# Bharat Intelligence Engine (BIE) — Product Requirements Document

> *India's strategic community deserves world-class intelligence tools. BIE delivers a global intelligence platform — built for India, by India.*

## 1. Executive Summary

The **Bharat Intelligence Engine (BIE)** is a real-time, open-source geospatial intelligence platform that delivers global situational awareness to Indian strategic personnel. It ingests live news feeds, government data, and open-source intelligence (OSINT) from around the world; processes them through an NLP and knowledge-graph pipeline; and surfaces actionable intelligence on an interactive map-based dashboard — all through the lens of India's national interests.

BIE replaces the manual workflow of reading yesterday's PDF reports with a **live, queryable, AI-powered global intelligence layer** tailored for Indian decision-makers. It monitors threats and developments across the Indo-Pacific, Middle East, Central Asia, Africa, and beyond — analysing how global events impact India's security, diplomatic interests, and economic partnerships. It delivers strategic briefs, instability scores, anomaly alerts, and natural-language Q&A grounded in real data.

### Why BIE Exists

| Problem | BIE's Answer |
|---|---|
| Intelligence reports are stale PDFs delivered hours or days late | Live ingestion pipeline with < 2-minute classification lag |
| No spatial awareness — global events lack geographic context for India | Every event geo-located on an interactive globe with India-priority analysis |
| Analysts manually correlate signals across global sources | Automated knowledge graph + convergence detection across theaters |
| No natural-language querying of intelligence databases | GraphRAG-powered Q&A with citations, tuned for India's strategic context |
| Instability trends are subjective and region-limited | Algorithmic BIE Instability Index (BII) covering global regions of Indian interest |
| Strategic briefs take hours to write | Auto-generated global briefs every 6 hours via LLM, with India-impact analysis |
| Existing global OSINT tools (Palantir, Janes) cost $50M+ | Open-source, built in 3 weeks, purpose-built for Indian strategic needs |

### Target Users

- Defence analysts and strategic affairs researchers
- Government intelligence consumers and MEA diplomats
- Foreign policy advisors and NSA staff
- Journalists covering national security and geopolitics
- Academic researchers in geopolitics, security studies, and international relations

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | Next.js 16, React 19, Tailwind CSS v4, MapLibre GL JS, Deck.gl |
| **Backend** | Python, FastAPI, Uvicorn |
| **NLP** | spaCy (English + Hindi/Urdu), custom NER models |
| **LLM** | Anthropic Claude API (strategic brief generation, GraphRAG) |
| **Knowledge Graph** | Neo4j |
| **Vector Store** | Qdrant |
| **Cache** | Redis |
| **Deployment** | Vercel (frontend), Railway (backend services) |
| **Database** | Neo4j (graph), Qdrant (vectors), Redis (cache/streams) |

---

## 3. Feature Registry — All 42 Features

Every feature is documented in detail in the linked sub-documents. Each feature includes: **what** is to be built, **why** it is needed, **what use** it provides, and **how** it is to be built.

### 🏗️ Architecture & Foundation

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F01 | Monorepo Initialization | BE1 | [architecture.md](./architecture.md#f01-monorepo-initialization) |
| F02 | API Contract & Shared Schemas | BE1 | [architecture.md](./architecture.md#f02-api-contract--shared-schemas) |
| F15 | Week 1 Integration Test | ALL | [architecture.md](./architecture.md#f15-week-1-integration-test) |

### ⚙️ Backend API

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F03 | FastAPI Foundation (stubs) | BE1 | [backend-api.md](./backend-api.md#f03-fastapi-foundation) |
| F11 | Backend Event/Feed Endpoints | BE1 | [backend-api.md](./backend-api.md#f11-backend-eventfeed-endpoints) |
| F35 | Caching Architecture | BE1 | [backend-api.md](./backend-api.md#f35-caching-architecture) |
| F36 | Error Handling & Degraded Mode | BE1+FE | [backend-api.md](./backend-api.md#f36-error-handling--degraded-mode) |

### 📡 Data Ingestion

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F08 | BIE Feed Registry | BE1 | [data-ingestion.md](./data-ingestion.md#f08-bie-feed-registry) |
| F09 | RSS Ingestion Worker | BE1 | [data-ingestion.md](./data-ingestion.md#f09-rss-ingestion-worker) |
| F10 | External API Workers | BE1 | [data-ingestion.md](./data-ingestion.md#f10-external-api-workers) |
| F37 | Data Freshness Monitor | FE2+BE1 | [data-ingestion.md](./data-ingestion.md#f37-data-freshness-monitor) |

### 🗺️ Geospatial Frontend

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F04 | MapLibre GL Integration | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f04-maplibre-gl-integration) |
| F05 | Deck.gl Layer System | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f05-deckgl-layer-system) |
| F06 | Strategic Geo Layers | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f06-strategic-geo-layers) |
| F07 | Regional Preset System | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f07-regional-preset-system) |
| F13 | Map Event Markers | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f13-map-event-markers) |
| F30 | Instability Heatmap Layer | FE1 | [geospatial-frontend.md](./geospatial-frontend.md#f30-instability-heatmap-layer) |

### 🖥️ UI Panels

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F12 | News Feed Panel | FE2 | [ui-panels.md](./ui-panels.md#f12-news-feed-panel) |
| F28 | NL Query Interface | FE2 | [ui-panels.md](./ui-panels.md#f28-nl-query-interface) |
| F29 | Knowledge Graph Panel | FE2 | [ui-panels.md](./ui-panels.md#f29-knowledge-graph-panel) |
| F32 | Strategic Posture Panel | FE2+AI3 | [ui-panels.md](./ui-panels.md#f32-strategic-posture-panel) |
| F34 | IOR Theater Dashboard | FE1+AI3 | [ui-panels.md](./ui-panels.md#f34-ior-theater-dashboard) |

### 🧠 NLP Pipeline

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F16 | spaCy Pipeline Setup | AI1 | [nlp-pipeline.md](./nlp-pipeline.md#f16-spacy-pipeline-setup) |
| F17 | Strategic NER (India-Priority) | AI1 | [nlp-pipeline.md](./nlp-pipeline.md#f17-strategic-ner) |
| F18 | Hindi / Urdu NLP | AI1 | [nlp-pipeline.md](./nlp-pipeline.md#f18-hindi--urdu-nlp) |
| F19 | Hybrid Threat Classifier | AI1 | [nlp-pipeline.md](./nlp-pipeline.md#f19-hybrid-threat-classifier) |
| F20 | NLP Worker Integration | AI1+BE1 | [nlp-pipeline.md](./nlp-pipeline.md#f20-nlp-worker-integration) |

### 🔗 Knowledge Graph & Vector Search

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F21 | Neo4j Schema & Seed Data | AI2 | [knowledge-graph.md](./knowledge-graph.md#f21-neo4j-schema--seed-data) |
| F22 | Automated Graph Writer | AI2 | [knowledge-graph.md](./knowledge-graph.md#f22-automated-graph-writer) |
| F23 | GraphRAG Query Engine | AI2+AI3 | [knowledge-graph.md](./knowledge-graph.md#f23-graphrag-query-engine) |
| F24 | Qdrant Vector Store | AI3 | [knowledge-graph.md](./knowledge-graph.md#f24-qdrant-vector-store) |

### 📊 Analytics Engine

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F14 | Global Strategic Hotspot Database | AI1 | [analytics.md](./analytics.md#f14-global-strategic-hotspot-database) |
| F25 | BIE Instability Index (BII) | AI3 | [analytics.md](./analytics.md#f25-bie-instability-index) |
| F26 | Anomaly Detection Engine | AI3 | [analytics.md](./analytics.md#f26-anomaly-detection-engine) |
| F27 | Signal Convergence Detector | AI3 | [analytics.md](./analytics.md#f27-signal-convergence-detector) |
| F33 | Infrastructure Proximity | AI2+FE1 | [analytics.md](./analytics.md#f33-infrastructure-proximity) |

### 📋 Intelligence Products

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F31 | Strategic Brief Generator | AI3+FE2 | [intelligence-products.md](./intelligence-products.md#f31-strategic-brief-generator) |

### 🎨 Design & UX

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F38 | BIE Design Language | FE1+FE2 | [design-system.md](./design-system.md#f38-bie-design-language) |
| F39 | Mobile Optimization | FE2 | [design-system.md](./design-system.md#f39-mobile-optimization) |

### 🚀 Deployment & Operations

| ID | Feature | Owner | Doc |
|---|---|---|---|
| F40 | Demo Environment | ALL | [deployment.md](./deployment.md#f40-demo-environment) |
| F41 | Demo Script & Rehearsal | ALL | [deployment.md](./deployment.md#f41-demo-script--rehearsal) |
| F42 | Production Hardening | BE1 | [deployment.md](./deployment.md#f42-production-hardening) |

---

## 4. Event Categories

BIE classifies events into 7 categories that cover the full spectrum of India's strategic concerns:

| Category | Color | Examples |
|---|---|---|
| `MILITARY` | Red | Troop movements, exercises, deployments, arms deals |
| `DIPLOMATIC` | Blue | Summits, treaties, sanctions, ambassador actions |
| `ECONOMIC` | Green | Trade deals, sanctions impact, BRI investments, supply chain disruptions |
| `INTERNAL` | Orange | Domestic unrest, policy changes, elections in key countries |
| `MARITIME` | Teal | Naval movements, chokepoint disruptions, piracy, port activities |
| `CYBER` | Purple | Cyber attacks, espionage, critical infrastructure breaches |
| `TERRORISM` | Dark Red | Terrorist incidents, counter-terrorism ops, radicalization |

---

## 5. Critical Path Analysis

These five dependency chains are where a delay cascades into multiple blocked engineers.

| Critical Path | Chain | Risk if Delayed | Mitigation |
|---|---|---|---|
| **API Contract** | F02 → F03 → F11 → F12/F13 | FE team blocked without real endpoints | Stub data from Day 1 |
| **NLP → Graph** | F16 → F20 → F22 → F23 | GraphRAG has no data | Seed Neo4j manually (F21) as parallel track |
| **Classifier → BII** | F19 → F25 → F30 | Instability heatmap stays empty | Hardcode BII scores as fallback |
| **GraphRAG → Query UI** | F23 → F28 | Query interface shows nothing | Wire to `/similar` (Qdrant) as fallback |
| **Demo Environment** | F40 → F41 → F42 | No rehearsal time | Allocate Day 19 fully to F40, no new features |

---

## 6. Data Schemas

See [schemas.md](./schemas.md) for the complete data model covering all API contracts, Neo4j graph schema, Qdrant collections, and Redis caching structures.

---

## 7. Risk Register

| Risk | Likelihood | Mitigation | Owner |
|---|---|---|---|
| NLP pipeline too slow for real-time | High | Batch 20 items. Run on Railway GPU. Accept 2-min lag. | AI1 |
| Neo4j graph sparse at demo time | High | Seed 1,000 global strategic facts manually by Day 8. | AI2 |
| Claude API costs spike | Medium | Cache all responses 24h. Set $50 daily spend cap. | BE1 |
| MapLibre performance on demo laptop | Medium | Cap ScatterplotLayer at 500 markers. Disable HeatmapLayer on low GPU. | FE1 |
| Hindi NLP inaccurate | Medium | Accept 75% accuracy. Focus demo on English feeds. Hindi is bonus. | AI1 |
| Railway service crashes mid-demo | High | Auto-restart policy. Pre-warm all services 10 min before demo. | BE1 |
| FE/BE API contract drift | High | BE1 owns `/backend/schemas.py`. Any change requires team announcement. | BE1 |
| Week 1 overrun bleeds into Week 2 | High | Cut scope: if F13/F14 not done by Day 6, drop and continue. | ALL |
| GraphRAG answers are wrong | Medium | Add confidence score. If < 0.6, show disclaimer. Qdrant `/similar` as fallback. | AI2+AI3 |
| Demo internet fails | Low | Phone hotspot. Pre-cache all Claude API responses for demo queries in Redis. | BE1 |
| Global feed volume overwhelms pipeline | Medium | Prioritize Tier 1–2 sources. India-adjacent events processed first. | BE1 |

---

## 8. Sub-Document Index

| Document | Covers |
|---|---|
| [architecture.md](./architecture.md) | Monorepo structure, API contracts, shared schemas, integration tests |
| [backend-api.md](./backend-api.md) | FastAPI foundation, event/feed endpoints, caching, error handling |
| [data-ingestion.md](./data-ingestion.md) | Feed registry, RSS worker, external APIs, data freshness |
| [geospatial-frontend.md](./geospatial-frontend.md) | MapLibre, Deck.gl layers, geo layers, presets, markers, heatmap |
| [ui-panels.md](./ui-panels.md) | News feed, NL query, knowledge graph, posture panel, IOR dashboard |
| [nlp-pipeline.md](./nlp-pipeline.md) | spaCy setup, strategic NER, Hindi/Urdu, threat classifier, NLP worker |
| [knowledge-graph.md](./knowledge-graph.md) | Neo4j schema, graph writer, GraphRAG, Qdrant vector store |
| [analytics.md](./analytics.md) | Hotspot DB, instability index, anomaly detection, convergence, infrastructure proximity |
| [intelligence-products.md](./intelligence-products.md) | Strategic brief generator |
| [design-system.md](./design-system.md) | BIE design language, mobile optimization |
| [deployment.md](./deployment.md) | Demo environment, demo script, production hardening |
| [schemas.md](./schemas.md) | All data models: API, Neo4j, Qdrant, Redis |

### Feed References

| Document | Covers |
|---|---|
| [feeds/INDIA_FEEDS.md](./feeds/INDIA_FEEDS.md) | India-specific RSS feeds, government APIs, Hindi sources |
| [feeds/ALL_FEEDS.md](./feeds/ALL_FEEDS.md) | Complete global feed reference (from World Monitor) — 170+ RSS, live APIs, Telegram, webcams |
