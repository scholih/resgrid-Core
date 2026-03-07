# Connexio-Ops Platform Design

**Date:** 2026-03-07
**Status:** Validated — ready for implementation planning
**Author:** Relogiks

---

## 1. Context & Strategic Position

### The Gap This Platform Fills

Dutch defense and security organizations (Kustwacht, Marechaussee, Navy, Korops Mariniers, Airforce, Landmacht) are operationally information-starved. They have large systems of record (SAP and equivalents) for HR, equipment, and logistics. What they lack is a **system of action** — something that answers "what do we do right now, with what we have, given what's happening?" under pressure, in Dutch, in 30 seconds.

The adoption of Connexio (a BM25 keyword search app) across all of these organizations is not a signal that they like knowledge management. It is a signal that the gap underneath is enormous.

JIVC (Joint Informatievoorziening Commando) already has **DefGPT Pro** — an air-gapped LLM inference infrastructure. Dutch defense has the AI layer. What they do not have is the operational application layer on top of it.

### Positioning

This platform is not a new ERP. It is the **nervous system between existing systems** — connecting systems of record (SAP, legacy HR, equipment inventories) to real operational workflows (dispatch, situational awareness, knowledge retrieval) in real time.

It does not replace HR, logistics, communications, or identity systems. It integrates with them and makes their data operationally useful under pressure.

### Market Scope

Two converging domains:

**Emergency services (reactive):** Fire, EMS, police, coast guard — classic CAD. Agencies that need incident management, dispatch, and resource coordination.

**Defense / public safety (proactive):** Military command operations, threat monitoring, seabed infrastructure protection, shadow fleet surveillance, multi-agency coordination. Organizations that need a persistent operational picture and predictive intelligence, not just reactive dispatch.

These are not separate products. They are two modules on a shared platform.

---

## 2. Platform Architecture

### Core Concept

An opinionated, AI-native operational platform structured as:

```
┌─────────────────────────────────────────────────────┐
│                      MODULES                         │
│   ┌─────────────────┐    ┌─────────────────────┐    │
│   │   CAD module    │    │    COP module        │    │
│   │ Incident mgmt   │    │ Operational picture  │    │
│   │ Dispatch        │    │ Threat monitoring    │    │
│   └─────────────────┘    └─────────────────────┘    │
│               ┌───────────────┐                      │
│               │   KM module   │                      │
│               │ Docs + GenAI  │                      │
│               └───────────────┘                      │
├─────────────────────────────────────────────────────┤
│                      CORE                            │
│   Geo primitives · Event bus (NATS) · API gateway   │
│   Identity federation · Audit log · Real-time (WS)  │
├─────────────────────────────────────────────────────┤
│               INTEGRATION LAYER                      │
│   SAP · AIS · GPS/AVL · NL-Alert · Radio · IdP      │
└─────────────────────────────────────────────────────┘
```

### Design Principles

1. **Vibe coding throughout** — AI-assisted development exclusively. Go's explicitness and minimal magic frameworks keep the codebase readable and navigable for AI tooling.
2. **Opinionated domain model** — no no-code extensibility, no custom fields, no form builders. Every field exists because it serves a specific operational need. Connexio's infinite configurability is the anti-pattern.
3. **GenAI as first-class capability** — not bolted on. Knowledge retrieval, SOP surfacing, run log ingestion, and pattern intelligence are core behaviors.
4. **Spider-in-the-web integrations** — the platform sits at the center of existing systems, consuming their data without replacing them.
5. **Lean, fast, stable, secure** — don't try to replace multiple systems. Own only what must be owned.
6. **Air-gappable by default** — all inference local via Ollama. No data leaves the network. Single `docker-compose up` for self-hosted deployment.

---

## 3. The Three Owned Domains

### 3.1 Incidents (CAD layer)

The incident is the atomic unit of the platform.

**Lifecycle (fixed, opinionated):**
```
new → active → coordinating → resolved → closed
```

Every state transition is timestamped and immutable. No deleting history.

**Incident model:**
- Type: from a curated taxonomy per deployment (maritime SAR, border event, fire, EMS, etc.)
- Severity: P1–P4
- Location: coordinates + zone (configurable per deployment — North Sea sectors, border zones, etc.)
- Timeline: append-only event log (full audit trail)

**Dispatch:** An assignment links a resource to an incident with its own status:
```
dispatched → en_route → on_scene → returning
```

Resources are consumed from the integration layer. The platform tracks their operational state, not their master record.

**KM integration:**
- On incident create → relevant SOPs surface automatically
- On incident close → run log is ingested into the knowledge base

### 3.2 Operational Picture (COP layer)

A persistent, real-time map of what is happening and where.

- Active incidents on map
- Resource positions (from AIS, GPS/AVL, or manual update)
- Incoming alerts feed
- Zone awareness and overlays
- AI recommendation panel (optional, dismissable)

The COP never blocks on a data source. If AIS is unavailable, last known position is shown with a staleness indicator. The operational picture is always alive, always degrading gracefully.

### 3.3 Knowledge + GenAI (KM layer)

Documents are ingested from any source: uploaded manually, pulled via API, crawled from internal systems.

**Ingest pipeline:**
```
Source → OCR (Docling) → chunk → embed (bge-m3) → pgvector
                                → entity extraction → Neo4j (GraphRAG)
```

**Query pipeline:**
```
Query → embed → hybrid search (BM25 + semantic, pgvector)
              → graph traversal (Neo4j — related entities, procedures)
              → Qwen reasoning → cited answer in operator's language
```

**The flywheel:**
Every incident creates a run log that feeds back into the knowledge base. Every closed case is a new document. The system gets smarter with every operation.

---

## 4. Integration Layer — Spider-in-the-Web

Each adapter is a small, independent service — a NATS publisher. It speaks one external language and translates to the internal event model.

```
External sources                 Adapters              Internal model
────────────────                 ────────              ──────────────
SAP HR          ──── pull ────►  personnel-adapter  ►  Personnel
SAP Equipment   ──── pull ────►  asset-adapter      ►  Resource
AIS feed        ── stream ────►  ais-adapter        ►  Position update
GPS/AVL         ── stream ────►  avl-adapter        ►  Position update
NL-Alert/P2000  ── stream ────►  alert-adapter      ►  Incoming alert
Radio/C2 comms  ── stream ────►  comms-adapter      ►  Communication log
Internal docs   ──── push ────►  ingest-adapter     ►  Knowledge base
```

**Rules:**
- Adapters are optional at runtime — the platform starts and runs without any of them
- Adapters only publish to NATS — they never import internal packages
- Adding a new integration is an adapter problem, not a platform problem
- Enabled per deployment via config

---

## 5. Tech Stack

### Runtime

| Layer | Technology | Rationale |
|---|---|---|
| API + adapters | Go 1.22+ | Static binaries, low footprint, explicit, excellent for concurrent streams |
| AI service | Python 3.12 + FastAPI | Best ML ecosystem — LlamaIndex, GraphRAG, Ollama |
| Web UI | HTMX + Go templates | No JS framework complexity, server-rendered, AI-navigable |

### Data

| Store | Technology | Owns |
|---|---|---|
| Primary | PostgreSQL 16 + pgvector | Incidents, resources, assignments, document chunks, embeddings |
| Graph | Neo4j Community | Entity relationships, GraphRAG, pattern traversal |
| Cache / presence | Redis 7 | Sessions, real-time state |
| Message bus | NATS JetStream | All internal events, adapter → platform communication |

### AI Inference (all local)

| Model | Purpose |
|---|---|
| Qwen 2.5 7B via Ollama | Reasoning, Dutch-language answers (default) |
| bge-m3 via Ollama | Multilingual embeddings (NL/EN/FR/DE) |
| Qwen 2.5 72B | Deep analysis when GPU available — optional |

### Supporting Services

| Concern | Technology |
|---|---|
| Identity | Zitadel (OSS, self-hosted) — federates with existing IdPs via OIDC |
| DB queries | sqlc (type-safe, generated from SQL — no ORM) |
| DB migrations | golang-migrate |
| Observability | Grafana + Prometheus + Loki + OpenTelemetry + Langfuse (all local) |

---

## 6. Project Structure

Go monorepo with clean internal boundaries. Modules are logical separations within the same deployable — not microservices.

```
connexio-ops/
├── cmd/
│   ├── api/          ← main API server (all modules behind one gateway)
│   ├── worker/       ← background jobs, NATS subscribers
│   ├── migrate/      ← database migrations (runs once, exits)
│   └── adapters/     ← one binary per adapter (ais, sap, alerts...)
│
├── internal/
│   ├── core/         ← shared domain: identity context, geo, events
│   ├── cad/          ← incidents, assignments, dispatch logic
│   ├── cop/          ← operational picture, resource positions, zones
│   ├── km/           ← document ingest, search, GenAI retrieval
│   ├── store/        ← PostgreSQL (sqlc generated queries)
│   └── transport/    ← HTTP handlers, WebSocket, routing (chi)
│
├── services/
│   └── connexio-ai/  ← Python FastAPI (RAG, GraphRAG, Ollama)
│
└── infra/
    ├── postgres/     ← init SQL, migrations
    ├── nats/         ← JetStream config
    └── docker/       ← Dockerfiles
```

**Boundary rules:**
- `cad` and `cop` never import each other — they communicate via NATS events
- Both consume from `core` (geo primitives, identity context)
- `km` is triggered by events from `cad` (incident created → surface SOPs, incident closed → ingest run log)
- `store` is the only package that touches PostgreSQL
- Adapters only publish to NATS — they never call internal packages

---

## 7. Build Sequence

### Phase 0 — Walking skeleton
Full stack running, nothing functional yet. Every service starts, connects, stays healthy.
- Go API with `/health`
- Docker Compose: postgres+pgvector, neo4j, redis, nats, zitadel
- connexio-ai FastAPI with `/health`
- Adapter plumbing wired to NATS (no real sources yet)

### Phase 1 — Knowledge foundation
Start with KM, not CAD. Fastest path to demonstrable value.
- Document ingest pipeline (upload → chunk → embed → pgvector + neo4j)
- Hybrid search (BM25 + semantic)
- Dutch-language cited answers via Qwen/Ollama
- Migrate kustw8-core's existing Kustwacht document corpus — zero re-ingestion

*Rationale:* A working GenAI knowledge layer is a credible standalone product and the warmest conversation with BF/Kustwacht.

### Phase 2 — Incident core (CAD)
- Incident lifecycle (create → dispatch → resolve → close)
- Resource assignments and run log
- SAP adapter (read personnel + equipment)
- KM integration: SOPs on incident create, run log ingested on close

### Phase 3 — Operational picture (COP)
- Real-time map with resource positions (WebSocket to browser)
- AIS adapter (vessel positions)
- Active incidents on map
- Zone awareness

### Phase 4 — Intelligence layer
- Dispatch recommendations on new incidents
- Pattern analysis across incident history
- Proactive alerts (anomaly detection, zone risk scoring)

**Each phase is a usable, demonstrable product.** Phase 1 alone is a better Connexio. Phase 1+2 is a CAD with AI. Phase 1+2+3 is the full platform.

---

## 8. Standards Compliance

Multi-agency coordination and interoperability are first-class requirements. The platform speaks the language of every system it connects to.

### Standards Map

| Concept | ICS/CAD | IAMSAR/Maritime | NATO/Military | Platform entity |
|---|---|---|---|---|
| Something happening | Incident | SAR Case | Action/Event | **Incident** |
| Who's in charge | Incident Commander | SMC | Commanding Officer | **Command** |
| On-scene lead | Ops Section Chief | OSC | Task Group Commander | **Command member** |
| Participating orgs | Agency | MRCC / assisting units | Organisations | **Agency** |
| Who/what responds | Resource | SAR facility | Materiel/Person | **Resource** |
| Work assigned | Assignment | Tasking | Task | **Assignment** |
| Where things are | AVL / map | AIS / GMDSS | Operational picture | **Position** |
| Map symbol | — | — | APP-6D SIDC | **Resource.app6_sidc** |
| Geographic context | Division/Zone | Search area / SRR | Area of Operations | **Zone** |

### Standards Implemented

| Standard | Body | What it governs | Implementation |
|---|---|---|---|
| EIDO (NENA-STA-021) | NENA | CAD-to-CAD incident data exchange | Incident identifier format + EIDO JSON export |
| APCO Common Incident Types | APCO | Incident type taxonomy (fire/EMS/law) | Base incident type codes, maritime extensions added |
| ICS / NIMS | FEMA | Incident command structure, unified command | Command entity, resource status codes |
| IAMSAR | IMO/ICAO | Maritime SAR coordination, SMC/OSC roles | Incident phases (uncertainty/alert/distress), command roles |
| APP-6D / MIL-STD-2525D | NATO/DoD | Military map symbology (SIDC) | Resource.app6_sidc, COP symbol rendering |
| ITU-R M.1371 (AIS) | ITU | Vessel position reporting (NMEA/AIVDM) | AIS adapter, Resource.ais_mmsi, Position.ais_fields |
| JC3IEDM / MIM | NATO MIP | C2 information exchange data model | Entity naming and relationship model |
| NIEM | US DoJ/DHS | Data exchange foundation | EIDO compliance, interoperability schemas |

---

## 9. Domain Model

Designed from outside-in research across ICS, IAMSAR, NATO C2, EIDO, AIS, and APP-6D standards. Multi-agency coordination is first-class — not an afterthought.

```
Agency
├── id
├── name
├── type: coast_guard | military | police | fire | ems
│         port_authority | foreign_coast_guard
└── contact

Incident
├── id              ← EIDO-compatible globally unique identifier
├── type            ← APCO taxonomy code (fire/ems/law/maritime/border/threat)
├── subtype         ← domain-specific (SAR/pollution/vessel_distress/shadow_fleet)
├── phase           ← maritime: uncertainty → alert → distress
│                      all:      new → active → coordinating → resolved → closed
├── severity        ← P1–P4
├── location        ← coordinates + zone_id
├── command_type    ← single_agency | unified_command
├── lead_agency_id
├── agencies[]      ← all participating agencies (unified command)
└── timeline[]      ← append-only event log (immutable audit trail)

Command             ← models ICS Unified Command + IAMSAR SMC/OSC structure
├── incident_id
└── members[]
    ├── agency_id
    ├── role: IC | SMC | OSC | deputy | liaison | observer
    └── since

Resource            ← vessel, aircraft, vehicle, person, unit
├── id
├── type: vessel | aircraft | vehicle | person | team
├── name
├── agency_id       ← owning agency
├── capabilities[]  ← from SAP/HR integration
├── status: available | staged | dispatched | en_route
│           on_scene | returning | out_of_service
├── position        ← current + timestamp + source + staleness_seconds
├── ais_mmsi        ← vessels: links to AIS stream (ITU-R M.1371)
└── app6_sidc       ← APP-6D / MIL-STD-2525D 20-char symbol code

Assignment          ← resource committed to incident
├── incident_id
├── resource_id
├── requesting_agency_id
├── providing_agency_id   ← may differ (KNRM supports Kustwacht incident)
├── role: primary | support | staging | liaison
├── status: dispatched | en_route | on_scene | returning
└── eta

Position            ← time-series feed, drives the COP
├── resource_id
├── timestamp
├── coordinates
├── source: ais | gps_avl | manual
└── ais_fields      ← heading, speed, nav_status (AIS msg type 1/2/3)

Zone
├── id
├── name
├── type: sea_area | srr | border_zone | jurisdiction
│         search_area | aoo
├── geometry        ← GeoJSON
└── agency_id       ← responsible agency

Alert               ← inbound from external sources
├── id
├── source: nl_alert | p2000 | ais_distress | mayday | manual
│           external_agency | sensor
├── type
├── location
└── incident_id     ← nullable: linked when dispatcher creates incident
```

### Three multi-agency properties that matter

**`providing_agency_id` ≠ `requesting_agency_id` on Assignment** — KNRM lifeboat responds to a Kustwacht incident. Each agency retains ownership of their resource while it is operationally tasked to another agency's incident. This is Unified Command expressed in the data model.

**`app6_sidc` on Resource** — every resource renders as a correct APP-6D military symbol on the COP map without manual configuration. A Kustwacht vessel, a Mariniers unit, a Landmacht helicopter — all display with proper NATO symbology to a military operator.

**Dual-track `phase` on Incident** — maritime incidents carry the IAMSAR `uncertainty/alert/distress` phase alongside the operational status. A SAR coordinator sees the IAMSAR phase; a fire chief sees `active/coordinating`. Same incident, right vocabulary per role.

---

## 10. Open Questions

| Question | Impact |
|---|---|
| Commercial structure with BF | Branding, IP, go-to-market |
| JIVC GenAI day outcome (March 26) | Procurement pathways, direct relationships |
| GrIT / FOXTROT program scope | Potential fit with COP/CAD roadmap |
| JIVC existing IdP (NATO PKI?) | Zitadel federation requirements |
| Defensienet integration constraints | Adapter architecture for classified networks |
| Norwegian MoD angle | Allied market expansion, NATO interoperability |

---

*Validated through design brainstorm session 2026-03-07. Supersedes the earlier Connexio-Ops design doc (2026-02-23) which was written for a narrower Kustwacht-only scope.*
