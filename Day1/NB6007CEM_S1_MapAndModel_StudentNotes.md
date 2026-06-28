# NB6007CEM — Web API Development
## Session 1 Study Notes: The Map, the Model and How Systems Talk

**Module:** NB6007CEM — Web API Development (Level 6, 20 credits)  
**Programme:** BSc (Hons) Computer Science with Software Engineering — Coventry University / NIBM  
**Session:** S1 (Day 1 AM)  
**Lecturer:** Niranga Dharmaratna | niranga@nibm.lk

---

## Session Learning Outcomes

By the end of Session 1 you should be able to:

1. **Describe** the Sri Lanka Police Tuk-Tuk Monitoring System: its IoT architecture, geographic hierarchy, and the two distinct API consumer roles.
2. **Explain** how two systems communicate: client, server, a shared language, and an agreed protocol — applied to the tuk-tuk system.
3. **Apply** ER notation to model the domain and explain why the data model precedes every other design decision (REST API Design §3).

---

## Section A — The Business Case

### The Problem

Sri Lanka Police need to monitor thousands of civilian tuk-tuks as they move across provinces, districts, and police stations. Each tuk-tuk carries an IoT GPS device that transmits the vehicle's position automatically at regular intervals — the driver does not need to act. Police dispatchers, supervisors, and field officers query that data in real time and historically.

The API sits in the middle of this system:

- **IoT devices write to it.** They push location pings autonomously.
- **Police applications read from it.** They query location history and last-known positions.

This is not a system where police officers report their own locations. The data producer (the device) and the data consumer (the police application) are completely separate parties.

**Analogy:** A tuk-tuk meter logs every fare automatically — no driver taps a button. A central licensing database is the API; transport inspectors query it later, never the vehicle directly. Limit: a meter records fare amounts, not GPS coordinates. The producer-consumer pattern is identical; the data type is not.

### The Scale

The system is seeded at the following scale for coursework:

| Element | Seed Scale | Role |
|---|---|---|
| Provinces | 9 | Top-level jurisdiction scope |
| Districts | 25 | Mid-level read scoping |
| Police Stations | 20+ | Station-level access boundary |
| Vehicles (Tuk-Tuks) | 200+ | IoT data producers |
| Location Pings | 1 week+ | Append-only time-series history |

This scale drives data generation in Session 2. Seed data generation is one of the first tasks you will complete.

**Coursework connection:** the architecture and data model dimension is assessed from Day 1. The domain framing you establish today is what you justify in your report's architecture section.

---

## Section B — How Two Systems Talk

### Client and Server

Every networked system divides into two roles: one asks, one answers. The **client** makes a request. The **server** processes it and returns a response. The client does not need to know how the server works internally — it only needs to know how to ask correctly.

One server can serve many clients simultaneously, and one client can talk to many servers.

In the tuk-tuk system:
- IoT GPS devices are **write-clients** — they send location pings.
- Police applications are **read-clients** — they query position data.
- The REST API we are building is the **server**.

**Analogy:** At a trishaw stand, a passenger says "Pettah" — they see no driver roster, no assignment process; they get one answer. The internal routing stays completely hidden. Limit: a dispatcher remembers regular customers; an HTTP server does not — each request is treated as the first. That statelessness is intentional.

### A Shared Language

Client and server must agree on the shape of what is exchanged: a data format both sides can read and write. This format is:
- Separate from the **protocol** (how the data travels across the network).
- Separate from the **data model** (what the data conceptually represents).

Examples of formats: JSON, XML, plain text. We have not chosen one yet. That decision belongs to **Session 5**, and making it before understanding the data model is the mistake we are deliberately avoiding.

> REST API Design principle: specify the data model first, choose the representation format later.

### A Protocol

A protocol is a complete, agreed set of rules so that a conversation between two systems works reliably. It defines how to open the conversation, how to frame a request, how to frame a reply, how to signal failure, and how to close.

Without an agreed protocol, the server cannot tell where one message ends and the next begins. Protocols exist at every network layer. The web uses **HTTP** at the application layer.

**Analogy:** A bus pass check works because both sides agreed on the steps in advance — you hold it out, they nod or question. No one improvises on the spot. Limit: a conductor can bend the rules for a child without a pass; HTTP cannot — every rule is machine-enforced with no flexibility or mercy.

### A Glimpse of HTTP

HTTP (HyperText Transfer Protocol) is the application-level protocol of the entire web. Every API call in this module travels over HTTP, or its encrypted version HTTPS.

HTTP defines:
- **Verbs for asking:** GET (read), POST (create), PUT (replace), DELETE (remove).
- **Status codes for answering:** 200 OK, 201 Created, 404 Not Found, 401 Unauthorised...
- **One critical property:** HTTP is stateless. The server treats every request as entirely independent.

Verbs and status codes are taught fully in **Session 7 (Day 4 AM)**. Today, HTTP is simply our agreed protocol.

---

## Section C — The Tuk-Tuk System

### Two Client Types, Two Roles

The system has two distinct client types, and the distinction shapes every security decision:

**Tracking Devices (Write-clients):**
- IoT GPS units fitted to each civilian tuk-tuk.
- Authenticate as their specific vehicle.
- POST location pings only — fully autonomous, no human involvement.
- Can only write data for their own vehicle; never for another's.

**Police Users (Read-clients):**
- HQ, provincial, district, and station officers.
- GET location data and movement history — read only.
- Scoped by jurisdiction: a station officer reads only their station's vehicles.
- Never POST data. The read path and write path never cross.

### The Write-Read Split

> **The device writes. The police read. The API must enforce that line.**

Every authentication decision, every permission check, every scope in this system flows from this one fact. The write path and the read path are architecturally separate by design — different authentication mechanisms, different permissions, different data flows.

This separation is not just a policy choice; it must be enforced by the API itself. If a police user could accidentally or intentionally write a ping, the system's data integrity would collapse immediately.

---

## Section D — The Data Model

### Why the Data Model Comes First

Step 1 of REST API design (REST API Design §3): model the data **before** writing a single URL or choosing a format.

The data model is **implementation-independent** — it describes what exists in the domain, not how it is stored or transferred. It is the foundation from which resources, URIs, representations, and HTTP methods are all derived. Getting the model wrong means everything built above it is also wrong. Changing the model late is expensive. Changing a URL later is cheap. **Sequence matters.**

**Analogy:** An architect commits to a floor plan before choosing the tiles — moving walls on paper costs nothing; after construction it costs everything. The data model is that floor plan; URLs, fields, and status codes are the interior finish. Limit: unlike a floor plan, a live public API cannot be quietly revised — every client breaks.

**Boundary for this session:** no URLs, no JSON, no XML. We are modelling, not designing endpoints. URIs come in Session 3. The format choice is Session 5.

### ER Notation — A Quick Refresher

An **entity** is a thing that exists independently and has attributes — for example, `Vehicle` with a registration number and a `device_id`. A **relationship** connects two entities — for example, a Vehicle is registered at a PoliceStation. **Cardinality** tells you how many: one station has many vehicles (1-to-many); one ping belongs to one vehicle (many-to-1).

We are modelling the domain, not writing SQL tables. No primary-key syntax, no foreign-key constraints here.

**Analogy:** Grama Niladhari knows every household in its area; the DS office covers multiple GN areas; the district covers multiple DS offices; the province covers multiple districts. This IS our Province - District - Station hierarchy. You use this structure daily. Limit: GN carries legal authority and jurisdiction; our model carries read-scope only.

### The Domain Model

```
Province  1—*  District  1—*  PoliceStation  1—*  Vehicle  1—*  LocationPing

User (Role: HQ / Provincial / District / Station)
  └── read access scoped to their jurisdiction level
```

**Entity attribute summary:**

| Entity | Key Attributes |
|---|---|
| Province | province_id, name |
| District | district_id, name, province_id |
| PoliceStation | station_id, name, district_id |
| Vehicle | vehicle_id, reg_number, **device_id**, station_id |
| LocationPing | ping_id, vehicle_id, timestamp, lat, lng, speed |
| User | user_id, role (HQ / Provincial / District / Station) |

### device_id as a Vehicle Attribute

`device_id` is an attribute of `Vehicle` — **not** a separate `Device` entity.

Every tuk-tuk has exactly one GPS device; the device exists only to serve the vehicle it is fitted to. A separate `Device` entity would add a join, a concept, and a table with no analytical value at this scope.

This is a **deliberate design decision**, not a shortcut. At the viva, you will be asked to explain it:

> "If the system later needed to track device replacements, calibration history, or battery status, a separate Device entity would earn its place. For this system's scope and requirements, it does not. This is a valid engineering choice — simplicity when justified is correct design."

### LocationPing as Its Own Entity

`LocationPing` is not a field on `Vehicle` — it is its own entity. A vehicle accumulates hundreds of pings per day, each with a timestamp, latitude, longitude, and speed. This is a **time-series**: append-only, never updated, continuously growing. It cannot be represented as a single field on `Vehicle`.

Keeping `LocationPing` as a separate entity is what later makes filtering by time window, pagination, and historical queries arise naturally from the design. This is the model driving Session 9's content.

**Analogy:** An electricity meter appends a new reading each billing cycle — it never overwrites a single "current reading" field. The utility company queries the history of readings, not a single value. Limit: a meter is read monthly; a GPS ping arrives every 30 seconds. Same append-only pattern; very different data volume.

### Generator Mistakes — What to Watch For

When you ask a code generator to produce an ER model for this system, expect these mistakes:

1. **Invents a separate Device entity** as the primary data source, obscuring the Vehicle relationship entirely.
2. **Bakes format names into the model**: `json_payload`, `response_data` — conflating representation with structure.
3. **Adds `last_known_lat` and `last_known_lng` as fields on `Vehicle`** — collapsing the time-series into a single value.
4. **Uses SQL-style CamelCase field names** — pre-empting implementation decisions the data model must not make.
5. **Makes the police `User` entity the data producer** — conflating who reads the API with the device that writes to it.

Every one of these appears in a typical first-pass generated ER. Your job at the viva is to spot each mistake and explain exactly which design principle it violates.

**Coursework connection:** the architecture and data model assessment dimension is directly developed by this session. The entity decisions you make today — and the justifications you can give for them — form the architecture section of your report.

---

## Section E — The Design Map

### From Data Model to Resources: A Preview

REST API Design §4 derives five kinds of resource from the data model. You do not need to master this today — full derivation happens in Session 3. But know the five kinds exist:

1. **Atomic** — a single instance (e.g. one Vehicle, one Province).
2. **Collection** — a set of atomics (e.g. all Vehicles in a district).
3. **Composite** — a bundle retrieved as one (Vehicle + its last ping).
4. **Processing Function** — a derived computation (last known position).
5. **Controller** — a multi-resource operation (rare in this system).

**Demo — Resource Type Classifier:** Open the demo. Drop an entity from the domain model and immediately see which of the five kinds it resolves to, and the rule that decided it. Try: Province, LocationPing, and the "last known position" concept.

### Clients Win Over Data

The resource model is not a direct copy of the data model (REST API Design §4 principle). The domain model drives implementation; the resource model is driven by how clients actually need to access data.

Police do not want "all location pings ever." They want "pings for WP-1234 in the last 24 hours, Western Province." When the domain model and a client access pattern are in tension, the client access pattern wins.

**Analogy:** A library catalogue holds every book, but readers ask "all books by this author at Colombo branch, published after 2019" — never "show me all books." The catalogue is the data model; the search interface is the resource model, shaped entirely by what readers need. Limit: our API scopes reads to jurisdiction — not every combination a reader might want is valid.

### The WSO2 Design Spine — Your Map

REST API Design defines a sequential design process. This map is the structure for the entire 8-day block:

| Step | What | Session |
|---|---|---|
| 1 | **Create Data Model** | S1 — Today |
| 2 | Derive Resources | S3 |
| 3 | Decide on Representations | S5 |
| 4 | Name Resources by URIs | S3–S4 |
| 5 | Determine HTTP Methods | S7 |
| 6 | Determine Special Behaviour | S9–S11 |
| 7 | Consider Errors | S11 |
| 8 | Security | S13 |
| 9 | Maturity Evaluation | S15 |

The complete map is visible from Day 1. You always know where you are in the design process and what comes next. Each step maps to assessment dimensions — keep pace and you accumulate evidence continuously rather than deferring everything to the final day.

---

## Activity — ER Model the Case

**Duration:** 25 minutes (individual, then share)

1. Open draw.io (diagrams.net), paper, or any diagram tool.
2. Draw the 5 entities: Province, District, PoliceStation, Vehicle, LocationPing.
3. Add at least 3 key attributes per entity.
4. Draw the relationships and mark cardinality (1-to-many).
5. Add the User entity with its role attribute and jurisdiction link.
6. Compare with the reference model from the slides. Note every difference.
7. Be ready to explain one design decision at the micro-viva.

**Demo — Resource Type Classifier:** Before completing the activity, open the demo and try dropping each entity from your model to see which resource kind it resolves to.

---

## Key Terms

| Term | Meaning |
|---|---|
| **Client** | The party that initiates a request in a client-server system. |
| **Server** | The party that receives requests and returns responses. |
| **Protocol** | A complete, agreed set of rules governing how two systems communicate. |
| **HTTP** | HyperText Transfer Protocol — the application-level protocol of the web. |
| **Stateless** | Each HTTP request is handled independently; the server holds no session memory. |
| **Data model** | An implementation-independent description of what exists in the domain. |
| **Entity** | A thing that exists independently and has attributes (ER modelling). |
| **Cardinality** | The count of instances on each side of a relationship (e.g. 1-to-many). |
| **Time-series** | An append-only, ordered sequence of records — LocationPing is the example here. |
| **IoT device** | Internet of Things — an embedded device that communicates with a network service autonomously. |
| **Write-client** | A client that sends data to the API (IoT devices in this system). |
| **Read-client** | A client that queries data from the API (police users in this system). |
| **Write-read split** | The architectural separation of data producers from data consumers — the foundational design decision in this system. |
| **REST API Design §3** | The data model step in the REST API design process. |
| **WSO2 design spine** | The nine-step REST API design process used as the teaching framework for this module. |
