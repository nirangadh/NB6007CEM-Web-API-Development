# NB6007CEM — Session 5 Student Study Notes
## Composite Resources and Representations
**Day 3 (AM) — Session 5 | VehicleComposite**

---

## Session overview

Sessions 3 and 4 gave you a working API with nine read-only routes and URI paths that satisfy the REST API Design Guidelines naming rules. Every JSON response your generator produced, however, was provisional: the field names, the nesting structure, and the presence or absence of embedded objects were all generator choices, not design decisions.

Session 5 closes that gap. By the end of the session every field name, every nested object, and every array in your API's responses is a design decision you can justify by reference to the guidelines. Session 6 this afternoon is when you implement those decisions in code.

---

## Session learning outcomes

By the end of this session you should be able to:

- Distinguish a resource from a representation and justify the choice of `application/json` for this API
- Design the vehicle composite resource that embeds the most recent location ping
- Design the `/last-position` processing function resource as a derived computation
- Establish authoritative JSON response shapes for all nine routes built in Sessions 3 and 4

---

## Section A — Resources and Their Representations (§6)

### What a resource is, and what a representation is

The REST API Design Guidelines separate two things that are easy to conflate: the resource and its representation.

A **resource** is an abstract, stable thing — the Province, the Vehicle, the LocationPing. It exists independently of any particular format or encoding. Its identity is fixed by its URI.

A **representation** is a rendering of that resource's information content in a specific format. The same Province resource can be rendered as `application/json` or `application/xml`. The province identifier and name do not change. Only the encoding changes.

The key principle: **the URI stays the same; the format changes**. A representation is not a different resource. It is a different view of the same resource.

### The police report analogy

A police report on the same incident can be rendered in Sinhala or in English. The case number, the time of the incident, and the offence section are identical in both documents. Only the language of rendering differs. A court clerk may request the English version; a Grama Niladhari office may require Sinhala. Same report, same facts — two representations.

*Limit: a police report language is usually fixed at the time of issue. An HTTP server negotiates format dynamically on each individual request.*

### Two generator mistakes to watch for

**Mistake 1 — Two routes for one resource.** A generator may produce `GET /vehicles` returning JSON and a separate `GET /vehicles.xml` returning XML. These are not two representations of one resource. They are two separate resources at different URIs. This violates the principle that one resource has one URI (§5.1). The format belongs in the `Accept` request header, not in the path.

**Mistake 2 — Wrong Content-Type header.** A generator using `res.send()` instead of `res.json()` in Express will produce a `Content-Type: text/plain` response header. Every JSON client will receive what looks like malformed data. The fix is simple: use `res.json()` on every route and verify the `Content-Type: application/json` header in Postman before the end of Session 6.

### The representation decision for this API

**`application/json` for all responses.** Three reasons support this choice explicitly:

1. JSON is native to Node.js and JavaScript — no parser is needed and objects are handled directly.
2. The tuk-tuk domain has no attribute vs element ambiguity, unlike XML.
3. `application/json` is the dominant media type in modern REST API practice and is supported by every client library.

### Content negotiation — for awareness (§10.1)

A client signals its preferred media type via the `Accept` request header:

```
Accept: application/json;q=0.9, application/xml;q=0.6
```

If the server does not support the requested type, it responds with `406 Not Acceptable`. This API supports one media type, so negotiation is trivial — but the `Accept` header and the `406` status code are assessable concepts that will appear at viva.

### Coursework connection

**Assessment dimension: API design.** The representation decision is documented in your report's architecture section. You must state the media type chosen and give at least one reason for the choice. The `res.json()` check is a Session 6 implementation task. WSO2 §6.

---

## Section B — Composite Resources (§4.3)

### What composite resources are

Groups of different entity types that are typically retrieved or deleted as a whole become composite resources (§4.3). The defining criterion is access pattern: if the parts of a composite are never useful without each other, retrieving them separately wastes round trips.

In the tuk-tuk system, a police dispatcher viewing a vehicle on a map always needs two things simultaneously: the vehicle's registration details and its last known position. Designing two separate calls — `GET /vehicles/{id}` for the details and `GET /vehicles/{id}/last-position` for the location — makes every map refresh twice as expensive as it needs to be. The vehicle composite solves this.

### The registration certificate analogy

A tuk-tuk registration certificate bundles vehicle details, the registered owner, and the fitness test result into one document. An officer at a checkpoint never asks for the fitness certificate without also needing the vehicle registration — they are only useful together. The composite resource works the same way: one request, one complete picture.

*Limit: a registration certificate is a static point-in-time snapshot. This composite resource returns live data updated with every new location ping.*

### Authoritative shape — vehicle composite

`GET /vehicles/{vehicle-id}` returns the following shape. This is now binding.

```json
{
  "vehicle_id": "WP-1234",
  "reg_number": "WP CAA-1234",
  "device_id":  "DEV-001",
  "station_id": "ST-05",
  "last_ping": {
    "ping_id":   "PNG-98765",
    "timestamp": "2025-01-15T10:30:00Z",
    "lat":        6.9271,
    "lng":        79.8612,
    "speed":      25
  }
}
```

The `last_ping` nested object is the embedded most recent `LocationPing`. It is not optional.

We use `GET` for this route. Session 7 explains exactly why `GET` is the correct method here (safe and idempotent, §7.1).

### Two generator mistakes for composite resources

**Mistake 1 — Flat location fields (the S1 modelling mistake at the representation level).** A generator may add `last_lat`, `last_lng`, and `last_speed` as flat scalar fields directly on the vehicle object. This collapses the `LocationPing` entity back into three anonymous numbers. What is lost: the `ping_id`, the `timestamp`, and the identity of the ping as a time-series record. The correct shape embeds the full `last_ping` object.

**Mistake 2 — Embedding the full history.** A generator may embed `"pings": [...]` — the entire location history — inside the vehicle composite. The history does not belong in the composite. It belongs at `GET /vehicles/{id}/pings` with pagination (covered in Session 9). The composite shows the most recent state only: one embedded ping, not a growing array. Sending 500 pings per vehicle on every `GET /vehicles/{id}` call is both a performance and a design failure.

### What to try with the demo (left panel)

Open `NB6007CEM_S5_VehicleComposite_Demo_RepAndURI.html`. Select the Vehicle Composite resource in the left panel and toggle between JSON and XML. Notice that the URI above both representations does not change. The resource is the same; only the rendering format changes. This is the core distinction this section teaches.

### Coursework connection

**Assessment dimension: API design and architecture.** The composite design decision — that `GET /vehicles/{id}` returns vehicle attributes plus `last_ping` — must be documented in the report. You must justify why the nested `last_ping` object is present and why the full history is not. WSO2 §4.3. The Session 6 implementation task is to update the route handler to produce this exact shape.

---

## Section C — Processing Function Resources (§4.5)

### What processing function resources are

Processing function resources provide access to derived computations — results that are not directly stored in the data model (§4.5). The computation takes existing data and derives a result on demand.

In the tuk-tuk system, a police officer occasionally needs only the last known position of a specific vehicle, without any of the vehicle's registration metadata. This is a computation derived from `LocationPing`: find the most recent ping for this `vehicle_id`. The result is not stored as a separate field anywhere — it is always derived on request.

### The stand supervisor analogy

When a police dispatcher asks the tuk-tuk stand supervisor "Where is WP-1234 right now?", the supervisor checks the last radio call log and reads out the location. There is no pre-stored current-location field in their ledger — the most recent contact entry is the answer, derived from the log. The `/last-position` resource is that same computation: ask for it, receive the result derived from the most recent ping.

*Limit: a human supervisor might misremember the last call or might not have heard it. The API always returns the exact most recent ping in the database.*

### Naming rule — noun, not verb (§5.1)

Processing function resources that represent derived reads are named as nouns (§5.1). The URI segment is `last-position`, not `getLastKnownPosition` or `calculatePosition`. Verbs in URI paths are only correct for controller resources (§4.4), which are the one resource type where §5.1 explicitly permits a verb.

### Authoritative shape — last-position

`GET /vehicles/{vehicle-id}/last-position` returns the following shape:

```json
{
  "vehicle_id": "WP-1234",
  "timestamp": "2025-01-15T10:30:00Z",
  "lat":   6.9271,
  "lng":   79.8612,
  "speed": 25
}
```

This shape contains position data only — no vehicle registration metadata. If you also need vehicle attributes, call `GET /vehicles/{vehicle-id}` (the composite).

We use `GET` for this route. Session 7 explains exactly why.

### Two generator mistakes for processing function resources

**Mistake 1 — Verb in the path.** A generator naming the route `GET /getLastKnownPosition` or `GET /vehicles/:id/calculatePosition` violates §5.1. Processing function resources that are derived reads are named as nouns. Correct: `GET /vehicles/{vehicle-id}/last-position`.

**Mistake 2 — Duplication.** A generator may make `last-position` a flat field on the vehicle composite AND a separate route simultaneously. The design decision is explicit: the composite embeds `last_ping` for access patterns that need vehicle context; `/last-position` is a lightweight processing function for position-only reads. Document which access pattern each route serves in your report.

### What to try with the demo (right panel)

Open `NB6007CEM_S5_VehicleComposite_Demo_RepAndURI.html`. In the right panel, type a vehicle identifier such as `WP-1234` into the input field. Watch the full scoped URI `/api/v1/vehicles/WP-1234/pings` assemble in real time as you type. This shows how URI templates work: the base path is fixed, and a variable (the vehicle ID) is substituted into the path.

### Coursework connection

**Assessment dimension: API design.** The `/last-position` route is new — it does not exist in your S4 build. Adding it in Session 6 is a measurable increment. Justify the noun naming choice against §5.1 in your report and document the design decision (composite vs dedicated resource) for the position data. WSO2 §4.5.

---

## Section D — Controller Resources (§4.4, for completeness)

Controller resources are used when multiple resources must be updated atomically in a single API call to maintain data consistency (§4.4). They are the one resource type where a verb URI segment is correct according to §5.1.

This API has no controller resources at this stage. IoT tracking devices only write location pings — a single-resource write each time. Police users only read. No multi-resource atomic updates arise in the core scope.

An example for illustration only (not built): if the system needed to transfer a vehicle from one station to another, updating both the vehicle's `station_id` and creating an audit record in a single atomic operation, that would be a controller resource: `POST /transfer-vehicle`.

Knowing this type exists allows you to justify its absence in your report architecture section.

---

## Section E — All Shapes Are Now Binding

From Session 5 onward, every field name and nesting structure in your API's JSON responses is a design decision. The shapes below are authoritative. Session 6 implements them.

### Atomic resource shapes

**Province:**
```json
{ "province_id": "PV-01", "name": "Western Province" }
```

**District:**
```json
{ "district_id": "DT-03", "name": "Colombo", "province_id": "PV-01" }
```

**PoliceStation:**
```json
{ "station_id": "ST-05", "name": "Colombo Fort", "district_id": "DT-03" }
```

### Collection shape rule

A collection endpoint returns an array of the corresponding atomic or composite objects. No metadata yet (count, next, previous belong to Session 9):

```json
[
  { "province_id": "PV-01", "name": "Western Province" },
  { "province_id": "PV-02", "name": "Central Province" }
]
```

### Vehicle composite shape

See Section B above. The `last_ping` nested object is mandatory.

### LocationPing shape (from `GET /vehicles/{id}/pings`)

```json
{
  "ping_id":   "PNG-98765",
  "vehicle_id": "WP-1234",
  "timestamp": "2025-01-15T10:30:00Z",
  "lat":        6.9271,
  "lng":        79.8612,
  "speed":      25
}
```

### Last-position processing function shape

See Section C above. Position data only — no vehicle metadata.

### Field naming decision — snake_case

**Decision: snake_case for every field in every resource.** This is consistent with the `seed.json` structure committed in Session 2. The decision must be documented in your report architecture section.

**Generator mistake to watch for:** camelCase field names (`vehicleId`, `regNumber`, `stationId`) appearing on some routes but not others. Field naming must be consistent across every route in the API.

### Avoid — the envelope wrapper

A generator may wrap every response in an envelope object:
```json
{ "data": [...], "status": "success" }
```

The REST API Design Guidelines §6 does not specify this wrapper. It adds complexity without value at this stage. Correct shapes are flat arrays and flat objects, exactly as defined above. A wrapper may be introduced in Session 11 if the error schema requires it. Until then, remove it.

---

## Session review — what changes in Session 6

Before Session 6 ends, your API must:

1. Return `application/json` Content-Type on every route (verify in Postman)
2. Respond to `GET /vehicles/{id}` with the composite shape including a nested `last_ping` object
3. Respond to the new route `GET /vehicles/{id}/last-position` with the position-only shape
4. Use snake_case field names consistently across all routes
5. Return flat arrays and flat objects — no envelope wrappers
6. Be redeployed on Render with all changes live

---

## Key terms

| Term | Definition |
|---|---|
| Resource | An abstract, stable entity identified by a URI. It is not tied to any particular format. |
| Representation | One rendering of a resource's information in a specific media type (e.g. `application/json`). |
| Media type | A standardised string identifying a data format, used in `Content-Type` and `Accept` headers (e.g. `application/json`, `application/xml`). |
| Content negotiation | The HTTP mechanism by which a client and server agree on the format to use for a response (`Accept` header, `406 Not Acceptable`). |
| Composite resource | A resource that bundles different entity types accessed together as a whole. |
| Processing function resource | A resource that exposes a derived computation whose result is not directly stored in the data model. |
| Controller resource | A resource that performs multi-resource atomic updates; named as a verb (the only type where verbs are correct). |
| snake_case | A naming convention where words are separated by underscores: `vehicle_id`, `reg_number`. |
| Envelope wrapper | An anti-pattern in which all API responses are wrapped in a container object (`{ "data": ..., "status": "success" }`). Avoid at this stage. |
| Flat object / flat array | A JSON object or array whose fields are direct keys, not wrapped in a container. The correct shape for §6. |
