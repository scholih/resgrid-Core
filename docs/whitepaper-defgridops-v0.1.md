# DefGridOps: Operational Intelligence for Emergency Services and Defence

**Version:** 0.1 — Initial Draft
**Date:** 2026-03-07
**Author:** Relogiks
**Classification:** Public

---

## Executive Summary

Modern defence and emergency services organizations possess large, mature systems of record — ERP platforms managing personnel, equipment, and logistics — and sophisticated communications infrastructure. What they lack is the layer in between: a **system of action** that answers the operational question under pressure: *what do we do right now, with what we have, given what is happening?*

DefGridOps is that system. It is an AI-native operational platform combining Computer-Aided Dispatch (CAD), Common Operational Picture (COP), and Knowledge Management (KM) in a single, standards-compliant platform. It does not replace existing systems. It sits between them — connecting data from systems of record to real operational workflows in real time.

The platform is designed for multi-agency operations. It speaks ICS, IAMSAR, NATO C2, and AIS natively. It is air-gappable by default. It surfaces relevant procedures at incident creation, learns from every closed case, and integrates with whatever identity, logistics, and communications infrastructure is already in place.

---

## 1. The Problem

### 1.1 The Gap Between Systems of Record and Systems of Action

Large defence and public safety organizations typically operate several mature enterprise systems: SAP or equivalents for HR and logistics, radio and TETRA networks for communications, and national databases for assets and personnel. These systems answer the question *what do we have?*

What they cannot answer — at least not in the thirty seconds an operational coordinator has — is: *what do we do with it right now?*

When a vessel reports distress in the North Sea, a SAR Mission Coordinator does not need the SAP screen. They need to know which patrol vessels are within range, which are already tasked, who the on-scene coordinator should be under IAMSAR protocol, what the relevant SOP says for this vessel type and location, and which other agencies need to be notified. That answer currently lives across four systems, three radio channels, and the coordinator's experience.

This gap is not unique to maritime SAR. It exists across every domain where time-critical, multi-agency decisions are made under uncertainty: border surveillance, fire and EMS dispatch, military COP, critical infrastructure protection.

### 1.2 The Knowledge Problem

Alongside the coordination gap sits a knowledge gap. Operational knowledge — standard operating procedures, doctrine, lessons learned, equipment manuals, after-action reports — is held in document repositories that are difficult to query under pressure. Search returns documents, not answers. Onboarding new operators takes months. When an experienced operator leaves, their pattern recognition leaves with them.

Generative AI has changed what is possible here. The question is no longer whether AI can surface relevant knowledge in natural language in real time. It can. The question is whether it can do so reliably, in Dutch, with citation, without hallucinating, within a secure and air-gapped environment.

### 1.3 The Fragmentation Problem

The Dutch maritime safety space alone involves Kustwacht, KNRM, Rijkswaterstaat, Marechaussee, and the Navy — each with their own systems, their own terminology, and their own command authority. Coordination during a major incident requires a unified operational picture that none of these organizations currently has by default. The same fragmentation exists across fire, EMS, border, and military domains.

---

## 2. The Solution: DefGridOps Platform

### 2.1 Core Concept

DefGridOps is structured as a **Core** platform with three functional modules:

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
│   Geo primitives · Event bus · API gateway           │
│   Identity federation · Audit log · Real-time (WS)  │
├─────────────────────────────────────────────────────┤
│               INTEGRATION LAYER                      │
│   SAP · AIS · GPS/AVL · NL-Alert · Radio · IdP      │
└─────────────────────────────────────────────────────┘
```

Each module is independently deployable. An organization can start with KM alone, add CAD, then COP. Each addition builds on — and enriches — the one before. The modules are not sold separately; they are phases of a single platform adoption.

### 2.2 Design Principles

**Opinionated, not configurable.** The platform has a fixed, well-researched domain model derived from ICS, IAMSAR, NATO C2, and APCO standards. There are no custom fields, no drag-and-drop form builders, no no-code configurability. Every data field exists because it supports a specific operational need. This keeps the codebase small, the system fast, and the AI layer effective.

**Spider-in-the-web, not replacement.** The platform never replaces HR, logistics, identity, or communications systems. It integrates with them via lightweight adapters and makes their data operationally useful under pressure. Adding a new integration is an adapter problem, not a platform problem.

**AI as first-class capability.** Knowledge retrieval, SOP surfacing, run log ingestion, and pattern intelligence are core platform behaviors — not features added to a traditional CAD. The GenAI layer improves with every incident through a structured knowledge flywheel.

**Air-gappable by default.** All AI inference runs locally via Ollama. No data is sent to external APIs. The entire platform deploys from a single `docker-compose up`. This is not a compromise — it is the only viable architecture for classified or sensitive operational environments.

**Multi-agency first.** Unified command, cross-agency resource tasking, and shared operational picture are not afterthoughts in the data model. They are first-class entities derived from ICS Unified Command and IAMSAR SMC/OSC structures.

---

## 3. The Three Owned Domains

### 3.1 Incidents — Computer-Aided Dispatch

The incident is the atomic unit of the platform. Every decision, resource movement, communication, and outcome is anchored to an incident.

**Lifecycle (fixed and auditable):**

```
new → active → coordinating → resolved → closed
```

Every state transition is timestamped and immutable. The timeline is append-only. No history is ever deleted or modified.

**Incident model highlights:**
- Type from a curated taxonomy per deployment: maritime SAR, border event, fire, EMS, shadow fleet, infrastructure threat
- Severity P1–P4, globally consistent
- EIDO-compatible globally unique identifier (NENA-STA-021)
- Dual-track phase: maritime incidents carry `uncertainty / alert / distress` (IAMSAR) alongside the operational status — the right vocabulary per role
- Full multi-agency command structure: lead agency, participating agencies, Unified Command

**Dispatch** links a resource to an incident with its own status lifecycle:

```
dispatched → en_route → on_scene → returning
```

Resources are consumed from integration (SAP, HR systems). The platform tracks their operational state. Master records stay in the systems that own them.

**Knowledge integration at dispatch time:** when an incident is created, relevant SOPs surface automatically — "vessel distress in SRR North", "hazmat maritime", "concurrent incidents protocol". The operator does not search. The platform delivers.

### 3.2 Operational Picture — Common Operational Picture (COP)

A persistent, real-time map of what is happening and where.

- Active incidents by type, severity, and phase
- All resource positions — from AIS (vessels), GPS/AVL (vehicles/aircraft), or manual update
- Incoming alert feed (NL-Alert, P2000, AIS distress, manual reports)
- Zone awareness: North Sea sectors, SRRs, border zones, Areas of Operations
- NATO APP-6D/MIL-STD-2525D symbology for all resources — correct military symbology without manual configuration
- AI recommendation panel: dismissable, non-blocking, always optional

**Graceful degradation:** the COP never blocks on a data source. If AIS is unavailable, last known position is shown with a staleness indicator. The operational picture is always alive.

**Multi-agency view:** each participating agency sees the full picture relevant to their role. A Kustwacht SMC sees all vessels and assigned assets. A Mariniers liaison sees their unit in the correct APP-6D symbol. Unified command roles are visible to all participants.

### 3.3 Knowledge and GenAI — the KM layer

**Ingest pipeline:**

```
Source → OCR/parsing (Docling)
       → chunking (structure-aware)
       → embedding (bge-m3, 1024-dim, multilingual)
       → pgvector (semantic search)
       → entity extraction
       → Neo4j (GraphRAG — entity relationships)
```

Supports any document source: uploaded files, API pull, internal system crawl.

**Query pipeline:**

```
Operator question (Dutch, English, or mixed)
  → embedding
  → hybrid retrieval: BM25 keyword + semantic vector (pgvector)
  → graph traversal: related entities, referenced procedures (Neo4j)
  → Qwen 2.5 reasoning
  → cited, persona-aware answer in the operator's language
```

**The knowledge flywheel:**

```
Incident created  →  relevant SOPs surfaced automatically
Incident closed   →  run log ingested into knowledge base
                  →  next similar incident is answered better
```

Every closed case is a new document. Every operation makes the next one faster and more informed. The system gets smarter with every deployment month.

**Persona-awareness:** the same document corpus returns different answers for a coordinator, a specialist, and a commanding officer. The query layer understands context. A "status of vessel HNLMS Holland" query from a command role gets a strategic summary; the same query from a maintenance role gets technical availability data.

---

## 4. Integration Architecture — Spider-in-the-Web

Each external system is connected via an independent adapter service. Adapters are small, purpose-built translators. They speak one external protocol and publish to the platform's internal event bus (NATS JetStream).

```
External sources                 Adapters              Internal model
────────────────                 ────────              ──────────────
SAP HR          ──── pull ────►  personnel-adapter  ►  Personnel
SAP Equipment   ──── pull ────►  asset-adapter      ►  Resource
AIS feed        ── stream ────►  ais-adapter        ►  Position update
GPS/AVL         ── stream ────►  avl-adapter        ►  Position update
NL-Alert/P2000  ── stream ────►  alert-adapter      ►  Incoming alert
Radio/C2        ── stream ────►  comms-adapter      ►  Communication log
Internal docs   ──── push ────►  ingest-adapter     ►  Knowledge base
```

**Architecture rules:**
- Adapters are optional at runtime. The platform starts and operates without any of them.
- Adapters only publish to NATS. They never call internal platform packages.
- Adding a new integration is writing a new adapter — it does not touch the core.
- Adapters are enabled per deployment via configuration. A Kustwacht deployment enables AIS and SAP. A police deployment enables their CAD feed and HR system.

This architecture means the platform can go live with a useful subset of integrations on day one, and grow connectivity without platform changes.

---

## 5. Standards Compliance

Standards compliance is not a checkbox exercise. It is the mechanism by which this platform achieves interoperability across agencies, nations, and command levels. The domain model was designed outside-in from the following standards:

| Standard | Body | What it governs | How implemented |
|---|---|---|---|
| **EIDO** (NENA-STA-021) | NENA | CAD-to-CAD incident data exchange | Incident ID format; EIDO JSON export endpoint |
| **APCO Common Incident Types** | APCO | Incident type taxonomy | Base type codes; maritime and military extensions |
| **APCO Common Status Codes** | APCO | Resource status vocabulary | Resource and assignment status enum |
| **ICS / NIMS** | FEMA | Incident command structure | Command entity; Unified Command model |
| **IAMSAR** | IMO/ICAO | Maritime SAR coordination | Incident phases (uncertainty/alert/distress); SMC/OSC command roles |
| **APP-6D / MIL-STD-2525D** | NATO/DoD | Military map symbology | `Resource.app6_sidc` field; COP symbol rendering |
| **ITU-R M.1371 (AIS)** | ITU | Vessel position and identity | AIS adapter; `Resource.ais_mmsi`; `Position.ais_fields` |
| **JC3IEDM / MIM** | NATO MIP | C2 information exchange model | Entity naming and relationship structure |
| **NIEM** | US DoJ/DHS | Information exchange foundation | EIDO compliance; interoperability schemas |

**Standards map across domains:**

| Concept | ICS/CAD | IAMSAR/Maritime | NATO/Military | Platform |
|---|---|---|---|---|
| Something happening | Incident | SAR Case | Action / Event | **Incident** |
| Who is in charge | Incident Commander | SMC | Commanding Officer | **Command** |
| On-scene lead | Ops Section Chief | OSC | Task Group Commander | **Command member** |
| Responding organizations | Agencies | MRCC / assisting units | Organisations | **Agency** |
| Who/what responds | Resource | SAR facility | Materiel / Person | **Resource** |
| Work assigned | Assignment | Tasking | Task | **Assignment** |
| Where things are | AVL / map | AIS / GMDSS | Operational picture | **Position** |
| Geographic context | Division / Zone | Search area / SRR | Area of Operations | **Zone** |

The same platform entity carries the right vocabulary to the right role. Terminology is not a configuration problem — it is a rendering problem, solved by persona and role context in the query and display layers.

---

## 6. Technical Architecture

### 6.1 Technology Stack

The platform is built for operational environments: low overhead, high reliability, full local deployment, AI-navigable codebase.

**Runtime:**

| Layer | Technology | Rationale |
|---|---|---|
| API + adapters | Go 1.22+ | Static binaries, minimal footprint, explicit concurrency model; readable by AI tooling |
| AI service | Python 3.12 + FastAPI | Best ML ecosystem — LlamaIndex, GraphRAG, Ollama client |
| Web UI | HTMX + Go templates | Server-rendered; no JS framework complexity; works in low-bandwidth environments |

**Data:**

| Store | Technology | Owns |
|---|---|---|
| Primary | PostgreSQL 16 + pgvector | Incidents, resources, assignments, document chunks, embeddings (HNSW index) |
| Graph | Neo4j Community | Entity relationships; GraphRAG traversal |
| Cache / presence | Redis 7 | Sessions; real-time cursor state |
| Message bus | NATS JetStream | All internal events; adapter-to-platform communication |

**AI inference (all local — no external API calls):**

| Model | Purpose |
|---|---|
| Qwen 2.5 7B via Ollama | Reasoning; Dutch-language answers; default deployment |
| bge-m3 via Ollama | Multilingual embeddings (NL/EN/FR/DE) — 1024-dimensional |
| Qwen 2.5 72B | Deep analysis; optional when GPU is available |

**Supporting services:**

| Concern | Technology |
|---|---|
| Identity | Zitadel (OSS, self-hosted) — federates with existing IdPs via OIDC/SAML |
| DB queries | sqlc — type-safe, generated from SQL; no ORM |
| DB migrations | golang-migrate |
| Observability | Grafana + Prometheus + Loki + OpenTelemetry + Langfuse (all local) |

### 6.2 Project Structure

Go monorepo with clean internal module boundaries. Modules are logical separations within a single deployable — not microservices.

```
defgridops/
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
│   └── transport/    ← HTTP handlers, WebSocket, routing
│
├── services/
│   └── defgrid-ai/   ← Python FastAPI (RAG, GraphRAG, Ollama)
│
└── infra/
    ├── postgres/     ← init SQL, migrations
    ├── nats/         ← JetStream configuration
    └── docker/       ← Dockerfiles, docker-compose
```

**Module boundary rules:**
- `cad` and `cop` never import each other. Cross-module communication is via NATS events.
- Both consume from `core` (geo primitives, identity context, event types).
- `km` is triggered by `cad` events: incident created surfaces SOPs; incident closed ingests run log.
- `store` is the only package that touches PostgreSQL.
- Adapters only publish to NATS. They never import internal packages.

### 6.3 Development Paradigm

The platform is developed exclusively through AI-assisted (vibe) coding. Go's explicitness — no magic frameworks, no reflection-heavy patterns, minimal abstractions — keeps every file navigable by both human operators and AI coding tools. The language choice is deliberately conservative: the codebase must be readable and modifiable by an AI agent with zero prior context.

---

## 7. Build Sequence

Each phase delivers a usable, demonstrable product. The sequence is designed to show value early and build trust with operational users before the full platform is complete.

### Phase 0 — Walking skeleton

Full stack running, nothing functional yet. Every service starts, connects, and stays healthy. Proves the architecture and delivery pipeline.

- Go API with `/health` and NATS connectivity
- Docker Compose: PostgreSQL+pgvector, Neo4j, Redis, NATS, Zitadel
- defgrid-ai FastAPI with `/health` and Ollama connectivity
- All module packages present, empty, but wired to build

### Phase 1 — Knowledge foundation

Start with KM, not CAD. Fastest path to demonstrable operational value.

- Document ingest pipeline: upload → OCR → chunk → embed → pgvector + Neo4j
- Hybrid search: BM25 keyword + semantic vector retrieval
- Dutch-language cited answers via Qwen/Ollama
- Persona-aware responses (coordinator vs. specialist vs. commander)

*This phase alone is a better Connexio. It is a credible standalone product and the entry point for any Kustwacht or JIVC conversation.*

### Phase 2 — Incident core (CAD)

- Incident lifecycle: create → dispatch → resolve → close
- Resource assignments and run log
- SAP adapter: read personnel and equipment
- KM integration: SOPs on incident create; run log ingested on incident close

*Phase 1 + Phase 2 is a CAD with embedded AI. Most emergency services organizations have no equivalent.*

### Phase 3 — Operational picture (COP)

- Real-time map with resource positions via WebSocket
- AIS adapter: live vessel positions from AIS feed
- Active incidents overlaid on map with zone awareness
- APP-6D symbol rendering for military resources
- Multi-agency view with role-based visibility

*Phase 1 + 2 + 3 is the full platform. Demonstrable to Kustwacht, JIVC, and NATO-aligned customers.*

### Phase 4 — Intelligence layer

- Dispatch recommendations on new incidents (nearest resource + capability match + availability)
- Pattern analysis across incident history
- Proactive zone alerts (anomaly detection, risk scoring)
- Cross-incident entity graph traversal (same vessel appears in multiple incidents)

---

## 8. Market and Procurement Fit

### 8.1 Target Segments

**Emergency services (dual-use entry point):** Coast guard, KNRM, fire, EMS, border police. Classic CAD buyers. Procurement via standard public tender. Reference customer value: demonstrates the platform in a real operational environment.

**Dutch defence:** JIVC, COMMIT, Navy, Marechaussee, Kustwacht (defence-classified operations). Procurement via SDIR (Strategic Defence Innovation Research) — the Dutch MoD's innovation procurement framework for SMEs and startups, structured as phased funding: paid feasibility study (Phase 3) then paid prototype (Phase 4). The "Data & AI" challenge announced at Purple Nectar 2026 is a direct fit.

**Allied nations:** Maritime nations with SAR / coast guard operations and NATO interoperability requirements. Norway, Belgium, Denmark as near-term targets. APP-6D compliance and IAMSAR implementation make the platform immediately credible to allied buyers without customization.

### 8.2 Strategic Positioning

DefGridOps is not competing with Motorola PremierOne, Hexagon I/CAD, or the large CAD incumbents in the core emergency dispatch market. The competitive frame is different:

- Incumbents are configured, not opinionated — they require months of professional services to deploy.
- Incumbents are not AI-native — AI is bolted on after the fact.
- Incumbents are not multi-agency first — unified command is an add-on or a separate product.
- Incumbents are not air-gappable — cloud dependency is a blocker for defence.

DefGridOps wins on: time-to-value (single `docker-compose up`), AI-native knowledge layer (live from day one), multi-agency model (ICS/IAMSAR/NATO built in), and security posture (fully local inference, no external dependencies).

### 8.3 The Flywheel Effect

Every organization that adopts the platform contributes to the knowledge commons. Incident types, SOP patterns, and after-action learning accumulate. An organization that has been on the platform for a year responds to novel incidents faster than one that has been on it for a month — and faster than any incumbent that relies solely on human expertise and static documentation.

---

## 9. Open Questions

The following questions are open and will be resolved through operational engagement, JIVC dialogue (March 26, 2026), and early adopter conversations:

| Question | Impact |
|---|---|
| Commercial structure with BF / Connexio | Branding, IP, go-to-market for Kustwacht segment |
| JIVC Leveranciersdialoog outcome (March 26) | Procurement pathways; direct JIVC/COMMIT relationships |
| SDIR Data & AI challenge timeline and scope | Primary funded procurement pathway for Dutch defence |
| GrIT / FOXTROT program alignment | Potential integration requirement for Dutch defence deployments |
| NATO PKI / Defensienet identity federation | Zitadel OIDC/SAML configuration requirements |
| Norwegian MoD angle | First allied market; NATO interoperability validation |

---

## Appendix A — Domain Model Reference

See `docs/plans/2026-03-07-platform-design.md` section 9 for the full annotated domain model with multi-agency property explanations.

---

## Appendix B — Standards Documents

The following documents are held in the Relogiks knowledge corpus and inform this platform's standards compliance:

**CAD / Emergency Management:**
- NENA-STA-021: Emergency Incident Data Object (EIDO) Standard
- APCO Common Incident Types Standard
- APCO Common Status Codes 2020
- NIMS/ICS Basics (FEMA, 2017)
- ICS Unified Command Technical Assistance (FEMA)

**Maritime / IAMSAR:**
- IAMSAR Manual Vol. I — Organisation and Management
- IAMSAR Manual Vol. III — Mobile Facilities
- Kustwacht Annual Reports 2021–2024
- KNRM Annual Reports 2022–2024

**Defence / NATO:**
- Netherlands Defence Doctrine 2025
- Netherlands Defence White Paper 2024
- Defence Strategy for Industry and Innovation (D-SII) 2025–2029
- Departementaal I-plan Defensie 2025–2027
- NATO AJP-01 Allied Joint Doctrine (Ed F, 2022)
- NATO AJP-3.19 Civil-Military Cooperation
- NATO AJP-6 Communications and Information Systems (Ed B)

---

*DefGridOps is a Relogiks platform. Relogiks — advanced CAD/COP solutions.*
*Document version 0.1 — initial draft. Subject to revision.*
