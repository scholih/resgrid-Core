# Connexio-Ops Platform — Architecture Design

**Date:** 2026-02-23
**Status:** Draft — awaiting commercial structure confirmation
**Author:** Relogiks

---

## 1. Context & Opportunity

### Background

Three existing assets converge into a single strategic opportunity:

| Asset | Owner | Status |
|-------|-------|--------|
| **Resgrid** | OSS | Mature C#/.NET CAD platform — dispatch, personnel, units, incidents |
| **Connexio** | Business Fundamentals | Deployed KM solution at Kustwacht + Marechaussee (BM25-based, no GenAI) |
| **kustw8-core** | Relogiks | Working RAG/AI demo — 2,300+ Kustwacht documents, GraphRAG, vessel tracking, already demoed |

### The Gap

Kustwacht (Dutch Coast Guard) currently uses Connexio for knowledge management but has **no CAD system** — no computer-aided dispatch, no incident management, no resource coordination. This is a confirmed, immediate pain point at an existing customer.

### The Opportunity

Combine all three into **Connexio-Ops**: a defense-grade operational platform providing CAD + Knowledge Management + AI Intelligence in a single, air-gappable system — purpose-built for Dutch defense and emergency services.

**Differentiator:** Every other CAD vendor provides dispatch. Nobody provides dispatch + knowledge management + AI intelligence that understands your documents, your vessels, your personnel relationships — all air-gappable, all in Dutch.

### Motivations

- **Vendor independence** — no Microsoft SQL Server, no Azure, no proprietary lock-in
- **Hosting flexibility** — single-command self-hosted install, scales to Kubernetes
- **OSS alignment** — 100% open source components, auditable, community-extensible
- **Defense-grade** — air-gapped operation, data never leaves the network

### Commercial Structure

Currently under exploration between Relogiks and Business Fundamentals. Architecture is designed to support any arrangement:
- Resgrid as CAD engine inside a Connexio product
- Joint venture / co-owned platform
- Relogiks-owned tech, Business Fundamentals as reseller

---

## 2. Platform Identity

```
┌─────────────────────────────────────────────────────┐
│                  CONNEXIO-OPS                        │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  CONNEXIO    │  │     CAD      │  │    AI     │  │
│  │  Knowledge   │  │   Dispatch   │  │Intelligence│  │
│  │  Management  │  │  & Incident  │  │ & GraphRAG│  │
│  └──────────────┘  └──────────────┘  └───────────┘  │
└─────────────────────────────────────────────────────┘
```

### Target Beachhead

**Kustwacht operations center** — confirmed pain (no CAD), confirmed trust (uses Connexio), confirmed AI interest (kustw8-core demo). First deployment target before expanding to Marechaussee, Navy, Airforce, Special Forces.

---

## 3. System Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CONNEXIO-OPS PLATFORM                     │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐  │
│  │ connexio-web│    │connexio-api │    │  connexio-ai    │  │
│  │             │    │             │    │                 │  │
│  │ Go / chi    │    │ Go / chi    │    │ Python/FastAPI  │  │
│  │ Unified UI  │    │ REST + WS   │    │ RAG + GraphRAG  │  │
│  │ CAD board   │    │ Core logic  │    │ Ollama / Qwen   │  │
│  │ KM search   │    │ Dispatch    │    │ kustw8-core     │  │
│  │ AI chat     │    │ Incidents   │    │ Vessel tracking │  │
│  └──────┬──────┘    └──────┬──────┘    └────────┬────────┘  │
│         │                  │                    │           │
│  ┌──────▼──────────────────▼────────────────────▼────────┐  │
│  │                    NATS JetStream                      │  │
│  │              (internal message bus)                    │  │
│  └──────────────────────────┬─────────────────────────────┘  │
│                             │                               │
│  ┌──────────────────────────▼─────────────────────────────┐  │
│  │                      DATA LAYER                        │  │
│  │                                                        │  │
│  │  PostgreSQL + pgvector   Neo4j          Redis          │  │
│  │  ├── CAD operational     GraphRAG       Sessions       │  │
│  │  ├── KM documents        Relationships  Real-time      │  │
│  │  └── Vector embeddings   Vessels/units  presence       │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │               CONTROLLED INGRESS                     │   │
│  │  AIS feed │ Emergency alerts │ External agency APIs  │   │
│  │           └── all gated, all optional                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Key Principle

Every external dependency is optional. The platform runs fully air-gapped. Ingress points (AIS, external alerts, agency APIs) are pluggable — present when the network allows, absent when it doesn't. CAD and KM never block on AI availability. AI degrades gracefully — never blocks dispatch.

### Technology Choices

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| CAD / API runtime | Go 1.22+ | Static binaries, low footprint, OSS, self-host friendly |
| AI service runtime | Python 3.12+ / FastAPI | Superior ML ecosystem (LlamaIndex, GraphRAG, Ollama) |
| Primary database | PostgreSQL + pgvector | OSS, vector search built-in, replaces SQL Server |
| Graph database | Neo4j Community | GraphRAG, relationship traversal, already in kustw8-core |
| Cache / pub-sub | Redis | Sessions, real-time presence, lightweight |
| Message bus | NATS JetStream | Single binary, OSS, simpler than RabbitMQ for self-hosters |
| LLM inference | Ollama + Qwen 2.5 | Fully local, no API keys, Dutch language capable |
| Embeddings | bge-m3 via Ollama | Multilingual (NL/EN/FR), already used in kustw8-core |
| HTTP router (Go) | chi | Stdlib-compatible, composable, no magic |
| DB queries (Go) | sqlc | Type-safe, generated from SQL, no ORM magic |
| DB migrations | golang-migrate | Simple CLI + library |
| Identity / OIDC | Zitadel | OSS, self-hostable, replaces OpenIddict |

---

## 4. CAD Core (Go)

### Resgrid Relationship

The Go CAD core is a **clean-break rewrite** of Resgrid — not a port. The C# codebase serves as domain knowledge and reference only. Go idioms and maritime domain requirements drive the design from scratch.

**What carries over (domain logic):** Personnel management, roles, certifications, shift scheduling, incident lifecycle, dispatch, run logs, notifications, department/unit hierarchy.

**What is added (maritime/defense):** Maritime incident types, AIS vessel integration, sea area/zone awareness, multi-agency coordination primitives, NATO/STANAG-aware data structures.

### Go Project Structure

```
/cmd/api        ← connexio-api entrypoint
/cmd/web        ← connexio-web entrypoint
/cmd/worker     ← connexio-worker entrypoint
/cmd/migrate    ← standalone migration CLI

/internal/domain        ← core business logic (zero framework deps)
/internal/store         ← PostgreSQL via sqlc
/internal/services      ← orchestration: domain + store + events
/internal/transport     ← HTTP handlers (chi), WebSocket
/internal/adapters      ← AIS, external alert ingress adapters

/pkg/           ← shared utilities (potentially importable)
```

### Domain Model — Maritime First

```
Incident
├── type: SAR | pollution | vessel_in_distress | border | event
├── location: coordinates + sea area (North Sea zones)
├── severity: P1–P4
├── status: new → active → coordinating → resolved → closed
└── timeline: append-only event log (full audit trail)

Resource
├── Vessel    (Kustwacht ships, KNRM lifeboats)
├── Aircraft  (helicopters, surveillance)
├── Personnel (coordinators, specialists, commanders)
└── Unit      (composed: vessel + crew + equipment)

Assignment
├── resource → incident
├── status: dispatched → en_route → on_scene → returning
└── ETA, position updates (AIS auto or manual)

ExternalAgency
├── KNRM, Defensie, port authorities, foreign coast guards
└── mutual aid requests, coordination messages
```

### Real-Time Dispatch Board

WebSocket feed to operations center UI:
- All active incidents on map
- All resource positions (AIS auto-update or manual)
- Assignment status per resource
- Incoming alerts feed
- AI recommendation panel (optional right rail, dismissable)

### API Surface

```
/api/v1/dispatch/    ← incidents, assignments, alerts
/api/v1/personnel/   ← users, roles, certifications, availability
/api/v1/units/       ← vessels, aircraft, apparatus, AVL
/api/v1/comms/       ← messaging, chat, notifications
/api/v1/km/          ← document search, SOP retrieval
/api/v1/ai/          ← dispatch recommendations, queries
/ws/dispatch         ← real-time WebSocket feed
```

### Authentication

OIDC/OAuth2 via Zitadel (self-hosted). Mobile apps, web UI, API all authenticate through Zitadel. Go API validates JWTs — stateless, no server-side sessions.

---

## 5. Knowledge Management Evolution (Connexio)

### Current → Future

```
Current Connexio
  Document upload → BM25 keyword index → keyword search → results list

Connexio-Ops
  Document upload → OCR (Docling) → chunk + embed (bge-m3)
                 → PostgreSQL + pgvector
                 → GraphRAG entity extraction → Neo4j

  Query → hybrid search (BM25 + semantic)
        → graph traversal (related entities, procedures)
        → Qwen inference → cited answer in Dutch
```

### What Carries Over from kustw8-core

- 2,300+ Kustwacht documents already indexed — **zero re-ingestion needed**
- bge-m3 multilingual embeddings (NL/EN/FR/DE/ES/PL)
- GraphRAG entity graph (vessels, personnel, procedures, zones)
- Persona-aware retrieval (coordinator / specialist / commander)
- Air-gapped Ollama/Qwen inference pipeline
- Spider/ingestion system for web sources (when network available)

### CAD ↔ KM Integration — The Flywheel

```
Incident created
→ connexio-ai surfaces relevant SOPs automatically
→ "3 procedures match this incident type"
→ SOPs attached to incident record, visible to coordinator

Incident closed
→ run log fed back into knowledge base
→ lessons learned become searchable
→ Neo4j graph updated with new entity relationships

Coordinator query: "brandstoflekkage procedures nabij windmolenpark"
→ RAG retrieves SOP + past incidents + vessel availability
→ Single cited answer in Dutch
```

Every incident makes the knowledge base smarter. Every closed case is a new document. Over time, Connexio-Ops becomes the institutional memory of the entire organization — something no BM25 system can ever achieve.

---

## 6. AI Intelligence Layer (`connexio-ai`)

kustw8-core promoted from demo to production service — not rewritten, productionized.

### Demo → Production Changes

| Aspect | Demo (kustw8-core) | Production (connexio-ai) |
|--------|-------------------|--------------------------|
| UI | Streamlit | FastAPI headless service |
| State | Local/SQLite | PostgreSQL (shared) |
| Search | OpenSearch standalone | pgvector (consolidated) |
| Ingestion | Manual scripts | Event-driven via NATS |
| Users | Single-user demo | Multi-tenant, role-aware |
| AIS | Mock data | Real feed (when available) |

### Four AI Capability Phases

**Phase 1 — Dispatch Intelligence** *(immediate, kustw8-core is ~80% there)*
```
New incident created
→ embed incident description → pgvector similarity search
→ Neo4j: which resources were effective in similar incidents?
→ Qwen reasoning → structured recommendation:
   "Based on 12 similar incidents, recommend:
    - Vessel Guardian (closest, SAR-certified crew)
    - Helicopter NH90 (if visibility < 500m)
    - Notify KNRM Terschelling station"
→ Displayed as suggestion — coordinator always decides
```

**Phase 2 — Operational Knowledge Retrieval**
```
Coordinator query (in Dutch)
→ bge-m3 embed → hybrid search (BM25 + semantic)
→ GraphRAG traversal (related procedures, responsible units)
→ Qwen answers in Dutch, cites source documents
→ Relevant SOP attached to active incident
```

**Phase 3 — Pattern Intelligence**
```
Commander: "Waarom hebben weekend SAR-incidenten in Q3
            altijd meer middelen nodig?"
→ GraphRAG traversal over multi-year incident history
→ Neo4j: seasonal patterns, resource correlations
→ Qwen: structured insight with supporting evidence
→ Exportable as briefing document
```

**Phase 4 — Predictive & Proactive** *(Defensie-ready)*
```
Continuous background monitoring:
→ AIS anomaly detection (shadow fleet, unusual patterns)
→ Weather data + historical incidents → SAR risk scoring per zone
→ Proactive alert: "Noordzee zone 4 verhoogd risico —
                   aanbeveling: Guardian voorpositioneren"
```

### Air-Gap Compliance

```
All inference runs locally via Ollama:
├── Qwen 2.5 7B   → general reasoning, Dutch language (default)
├── bge-m3        → multilingual embeddings
└── Qwen 2.5 72B  → deep analysis (when GPU available, optional)
```

No API keys. No cloud calls. No data leaves the network. Ever.

---

## 7. Deployment & Operations

### Self-Hosted — Single Command

```bash
docker-compose up
```

Full platform stack via Docker Compose:

```yaml
services:
  # Application
  connexio-api:       # Go binary
  connexio-web:       # Go binary
  connexio-worker:    # Go binary
  connexio-ai:        # Python / FastAPI
  connexio-migrate:   # Go binary (runs once, exits)

  # Data layer
  postgres:           # + pgvector extension
  neo4j:              # Community edition (OSS)
  redis:              # Alpine
  nats:               # JetStream enabled

  # Optional ingress adapters (enable/disable per deployment)
  ais-adapter:        # AIS feed → NATS
  alert-adapter:      # External alerts → NATS
```

Kustwacht sysadmin does not need to understand Kubernetes. Images ship as a bundle for air-gapped environments:

```bash
# Relogiks prepares release bundle
docker save connexio-ops:v1.0.0 | gzip > connexio-ops-v1.0.0.tar.gz

# Operator installs on Defensienet
docker load < connexio-ops-v1.0.0.tar.gz
./update.sh v1.0.0   # zero-downtime rolling update
```

### Cloud / Kubernetes

Same containers, Helm chart provided:
- Each service → Deployment with HPA
- PostgreSQL → CloudNativePG or managed RDS
- Neo4j → StatefulSet
- NATS → StatefulSet with JetStream persistence

### Configuration

Single `.env` file controls the entire deployment:

```bash
# Deployment profile
DEPLOYMENT_MODE=airgapped|connected|hybrid
TENANT_NAME=kustwacht
LANGUAGE_PRIMARY=nl
LANGUAGE_SECONDARY=en

# AI models (all local)
OLLAMA_MODEL=qwen2.5:7b
EMBEDDING_MODEL=bge-m3

# Ingress (all optional, all off by default)
AIS_ENABLED=false
AIS_API_KEY=
EXTERNAL_ALERTS_ENABLED=false

# Classification (future Defensie integration)
DATA_CLASSIFICATION=restricted|confidential
```

### Observability (all local)

```
Grafana + Prometheus  → metrics and dashboards
Loki                  → log aggregation
OpenTelemetry         → distributed traces
(Langfuse already integrated in connexio-ai)
```

---

## 8. Implementation Roadmap

### Phase 0 — Foundation *(now)*
- [ ] Commercial structure agreed with Business Fundamentals
- [ ] New repository created (`connexio-ops` or agreed name)
- [ ] Go project scaffold: `/cmd`, `/internal`, `/pkg` structure
- [ ] Docker Compose with PostgreSQL + pgvector + Neo4j + NATS + Redis
- [ ] Zitadel identity provider running
- [ ] kustw8-core migrated from Streamlit demo to FastAPI headless service

### Phase 1 — Kustwacht MVP *(beachhead)*
- [ ] CAD core: incident lifecycle, resource management, assignments
- [ ] Real-time dispatch board (WebSocket)
- [ ] AIS vessel position integration
- [ ] Connexio KM: hybrid search over existing 2,300+ document corpus
- [ ] Phase 1 AI: dispatch recommendations on new incidents
- [ ] Air-gapped Docker Compose bundle
- [ ] **Demo to Kustwacht operations center**

### Phase 2 — Production Hardening
- [ ] Full personnel management (from Resgrid domain)
- [ ] Shift scheduling and availability
- [ ] Multi-agency coordination (KNRM, Defensie)
- [ ] Phase 2 AI: Dutch-language SOP retrieval
- [ ] Run log → knowledge base feedback loop
- [ ] Helm chart for Kubernetes deployments
- [ ] Secure update mechanism for air-gapped environments

### Phase 3 — Expand
- [ ] Marechaussee deployment (border/event incident types)
- [ ] Phase 3 AI: pattern intelligence, briefing generation
- [ ] Navy / Airforce / Special Forces evaluation
- [ ] NATO/STANAG data structure alignment
- [ ] Phase 4 AI: predictive SAR risk scoring

---

## 9. Open Questions

| Question | Impact | Owner |
|----------|--------|-------|
| Commercial structure (A/B/C) | Branding, IP, revenue split | Relogiks + Business Fundamentals |
| Defensienet ingress constraints | AIS feed architecture | Kustwacht IT |
| Marechaussee CAD requirements | Incident domain model scope | Business Fundamentals |
| Connexio document corpus access | KM migration planning | Business Fundamentals |
| Classification level requirements | Data architecture, encryption at rest | Kustwacht IT / Defensie |
| Existing Connexio API contracts | Migration / backwards compatibility | Business Fundamentals |

---

*Document status: draft — validated through design brainstorm, pending commercial confirmation and Kustwacht IT requirements gathering.*
