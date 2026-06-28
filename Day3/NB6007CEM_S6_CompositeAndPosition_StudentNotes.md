# NB6007CEM - Web API Development
## Session 6 Study Notes: Building the Composite and Position Resources
**Day 3 (PM) - Session 6 | WSO2 §4.3, §4.4, §4.5, §6**

---

## What this session delivers

By the end of this afternoon your API has eleven routes, all returning S5-authoritative JSON shapes. That means every atomic resource returns the exact field structure you designed in S5, the vehicle single-resource route has been upgraded to a composite (with the most recent ping nested inside it), and a new processing function resource provides position-only data without vehicle metadata.

This is the implementation session for everything designed in S5. No new design principles are introduced today. The work is entirely generate, read, then justify or fix against the WSO2 guidelines and the authoritative shapes.

---

## The three-layer sequence

Work through these layers in order. Do not jump to Layer 2 before all Layer 1 routes are passing.

**Layer 1 - Fix all atomic routes (~25 min)**
Every route that came from S4 must now return the S5-authoritative shape. That means snake_case field names, no envelope wrapper object, all FK fields present, and `res.json()` used on every route.

**Layer 2 - Upgrade the vehicle composite (~60 min)**
`GET /vehicles/:vehicleId` must return the vehicle composite shape: all vehicle fields plus a `last_ping` object nested inside it. The `last_ping` is the single most recent `LocationPing` for that vehicle (found by sorting descending by timestamp and taking the first result), or `null` if no pings exist for the vehicle.

**Layer 3 - Add /last-position (~35 min)**
`GET /vehicles/:vehicleId/last-position` is a processing function resource that returns position data only - no vehicle metadata such as `reg_number`, `device_id`, or `station_id`. The route path must be `/last-position` (a noun, hyphenated, lowercase - no verb).

Why the sequence matters: if you start Layer 2 before Layer 1 is finished, your codebase will have some routes returning correct shapes and others still returning generator defaults. An inconsistent codebase fails the Day 3 checkpoint even if the composite itself is correct.

---

## S5 authoritative shapes - your reference

Keep this section open as you work. These are the exact shapes your routes must return.

**Province (atomic)**
```json
{ "province_id": "PV-01", "name": "Western Province" }
```

**District (atomic)**
```json
{ "district_id": "DT-03", "name": "Colombo", "province_id": "PV-01" }
```

**PoliceStation (atomic)**
```json
{ "station_id": "ST-05", "name": "Colombo Fort", "district_id": "DT-03" }
```

**Vehicle (atomic - also used inside the composite as base fields)**
```json
{ "vehicle_id": "WP-1234", "reg_number": "WP CAA-1234", "device_id": "DEV-001", "station_id": "ST-05" }
```

**Vehicle composite (GET /vehicles/:vehicleId)**
```json
{
  "vehicle_id": "WP-1234",
  "reg_number": "WP CAA-1234",
  "device_id": "DEV-001",
  "station_id": "ST-05",
  "last_ping": {
    "ping_id": "PNG-98765",
    "vehicle_id": "WP-1234",
    "timestamp": "2025-01-15T10:30:00Z",
    "lat": 6.9271,
    "lng": 79.8612,
    "speed": 25
  }
}
```
`last_ping` is `null` if no pings exist for this vehicle.

**LocationPing (GET /vehicles/:vehicleId/pings)**
```json
{ "ping_id": "PNG-98765", "vehicle_id": "WP-1234", "timestamp": "2025-01-15T10:30:00Z", "lat": 6.9271, "lng": 79.8612, "speed": 25 }
```

**Last-position (GET /vehicles/:vehicleId/last-position)**
```json
{ "vehicle_id": "WP-1234", "timestamp": "2025-01-15T10:30:00Z", "lat": 6.9271, "lng": 79.8612, "speed": 25 }
```

**Collections** are arrays of the corresponding atomic shape. No pagination envelope yet - that comes in S9.

---

## Layer 1 - Fix all atomic routes

### What to generate

Paste this prompt into your AI assistant:

```
Update these existing Express GET routes to return JSON matching the shapes
below exactly. Use res.json() for all responses. Do not add envelope wrappers.
Do not change route paths. Data is already loaded from seed.json.

GET /provinces -> array of { province_id, name }
GET /provinces/:provinceId -> { province_id, name }
GET /districts -> array of { district_id, name, province_id }
GET /districts/:districtId -> { district_id, name, province_id }
GET /stations -> array of { station_id, name, district_id }
GET /stations/:stationId -> { station_id, name, district_id }
GET /vehicles -> array of { vehicle_id, reg_number, device_id, station_id }
GET /vehicles/:vehicleId/pings -> array of { ping_id, vehicle_id, timestamp,
  lat, lng, speed }
```

### Four generator mistakes to catch

**1. camelCase field names**

Generators frequently produce `vehicleId`, `regNumber`, `stationId`. The S5-authoritative shapes use `vehicle_id`, `reg_number`, `station_id` throughout. Every field name must be snake_case. This is a WSO2 §6 representation decision - the data structure has been decided, and it uses snake_case.

*How to check:* Call `GET /vehicles` and look at the field names in the response body. If you see `vehicleId` instead of `vehicle_id`, the generator made this mistake.

**2. Envelope wrappers**

A common generator pattern is `{ "data": [...], "status": "success", "count": 5 }`. The WSO2 design pipeline specifies a representation that matches the data model directly. Remove the wrapper. Return the flat array or flat object directly.

*How to check:* Call any collection route. The response body should open with `[` (for a collection) or `{` followed immediately by a resource field (for an atomic). If you see `{ "data":` the generator added a wrapper.

**3. Missing FK fields**

The `province_id` field on District, the `district_id` field on PoliceStation, and the `station_id` field on Vehicle are the foreign-key fields that link the hierarchy. Generators often omit them - the response contains only the resource's own fields. These foreign-key fields must be present because they are how police users traverse the Province-District-Station-Vehicle hierarchy. Without `province_id` on a District, a police user cannot determine which province that district belongs to.

*How to check:* Call `GET /districts` and confirm that each object in the array has `district_id`, `name`, and `province_id`. If `province_id` is missing, the generator made this mistake.

**4. res.send() instead of res.json()**

`res.send(anObject)` can produce a response body that is serialised JSON, but does not set `Content-Type: application/json` reliably across all Express versions and middleware configurations. Use `res.json(anObject)` on every route. `res.json()` serialises and sets the content-type header correctly.

*How to check:* Call any route in your REST client and look at the response headers. The `Content-Type` header should read `application/json; charset=utf-8`. If it reads `text/html` or anything else, the generator used `res.send()`.

### Before moving to Layer 2

Open your REST client (Postman, Thunder Client, or curl) and call each route individually. Check: the field names are snake_case, no wrapper object wraps the response body, all FK fields are present, and the Content-Type header is `application/json`. Only when all eight routes pass this check should you move to Layer 2.

**Coursework connection:** Correctly named, correctly structured route responses are the primary evidence for the API Design dimension. Getting these right in Layer 1 is what every subsequent layer builds on. WSO2 §6 (representation specification) governs the shape of these responses.

---

## Layer 2 - Upgrade the vehicle composite (WSO2 §4.3)

### What a composite resource is

WSO2 §4.3 defines a composite resource as a group of different entity types that are "manipulated as a whole because these instances are perceived as aggregates, e.g. they are typically collectively retrieved or deleted." A composite is not just returning extra fields - it is returning a genuinely aggregated view where related data is embedded directly in the response.

For the tuk-tuk system, the vehicle composite is the practical response to a key police use case: "give me this vehicle's details and its current position in one call." Rather than making two requests (one for the vehicle, one for its latest ping), the composite embeds the most recent ping directly inside the vehicle object.

### What to generate

```
Update GET /vehicles/:vehicleId to return a vehicle composite. The response
should include all vehicle fields plus a last_ping field containing the most
recent LocationPing for this vehicle.

To find the most recent ping: filter the pings array from seed.json where
vehicle_id matches the requested vehicleId, sort by timestamp descending, and
take the first result. If no pings exist for the vehicle, last_ping should be null.

Response shape:
{
  "vehicle_id": string,
  "reg_number": string,
  "device_id": string,
  "station_id": string,
  "last_ping": {
    "ping_id": string,
    "vehicle_id": string,
    "timestamp": ISO 8601 string,
    "lat": number,
    "lng": number,
    "speed": number
  } | null
}
Do not embed the full pings array. One ping only.
```

### Three generator mistakes to catch

**Generator mistake 1 - Wrong sort direction (or no sort at all)**

Generators frequently sort ascending by timestamp (earliest ping first) or do not sort at all and rely on the order items appear in `seed.json`. If the sort is ascending, `[0]` gives you the *oldest* ping, not the most recent. The correct implementation filters pings for the vehicle, sorts by `timestamp` descending (newest first), and takes `[0]`.

Before testing, read your generated code and ask: "How does this code know which ping is most recent?" If the answer is "it relies on the order in seed.json" or "it sorts ascending", the code is wrong.

Example of what wrong generated code looks like:
```javascript
// WRONG - no sort, relies on array order
const lastPing = data.pings.find(p => p.vehicle_id === req.params.vehicleId);
```

```javascript
// WRONG - sorts ascending (oldest first)
const pings = data.pings
  .filter(p => p.vehicle_id === req.params.vehicleId)
  .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
const lastPing = pings[0];
```

```javascript
// CORRECT - sorts descending (newest first)
const pings = data.pings
  .filter(p => p.vehicle_id === req.params.vehicleId)
  .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
const lastPing = pings[0] || null;
```

**Generator mistake 2 - Flat fields instead of a nested last_ping object**

This is the same mistake that appeared in your S1 ER model and again in the S5 JSON shape critique. Generators that do not receive an explicit shape instruction will often produce flat fields on the vehicle object:

```json
{
  "vehicle_id": "WP-1234",
  "reg_number": "WP CAA-1234",
  "last_lat": 6.9271,
  "last_lng": 79.8612,
  "last_timestamp": "2025-01-15T10:30:00Z"
}
```

This is wrong. The S5-authoritative composite shape nests the full `LocationPing` object. The flat-fields pattern is a modelling error: it collapses a separate entity (LocationPing) into attributes of another entity (Vehicle), which is precisely the modelling mistake you identified and corrected in S1.

The correct shape embeds the complete `last_ping` object as defined in the `LocationPing` shape.

**Generator mistake 3 - Embedding the full pings array**

Some generators embed all of a vehicle's location history inside the composite response:

```json
{
  "vehicle_id": "WP-1234",
  "pings": [ { ... }, { ... }, { ... } ]
}
```

The composite embeds exactly one ping - the most recent one - as `last_ping`. The full history of location pings belongs to the separate collection route `GET /vehicles/:vehicleId/pings`. Embedding the full array in the composite defeats the purpose of having both routes and makes the composite response very large.

### The regression arc: three times the same mistake

The flat `last_lat`/`last_lng` pattern has appeared at three distinct points in this module: in your S1 entity-relationship model where you correctly identified that embedding lat/lng directly on Vehicle was wrong, in the S5 JSON shape critique where you saw that generators default to this flattened representation, and now in your Layer 2 generated code. Generators make this error consistently. The fact that you can identify it as wrong - and explain why it is wrong against §4.3 and the S5 shape decision - is precisely what the assessed craft requires.

**Coursework connection:** The vehicle composite is a direct test of §4.3 (composite resources). Correctly implementing the nested `last_ping` and explaining the design decision (why a composite rather than two separate requests, and why `last_ping` is one nested object rather than flat fields) is material for the API Design and Architecture dimensions of the report. WSO2 §4.3.

---

## Layer 3 - Add /last-position (WSO2 §4.5)

### What a processing function resource is

WSO2 §4.5 defines processing function resources as those that "provide access to functions that either process particular resources, or that perform certain resource independent computations." The `last_ping` computation on the composite was embedded inside the composite resource. `GET /vehicles/:vehicleId/last-position` makes that computation available as a standalone resource that returns only the positional data - no vehicle metadata.

This matters because a different class of client may only need the current position (for example, a map plotting system), not the full vehicle composite. The processing function gives them precisely what they need without the overhead of vehicle fields they will discard.

### What to generate

```
Add GET /vehicles/:vehicleId/last-position to the Express app. This route returns
only the most recent position for the vehicle - no vehicle metadata.

To find the most recent position: filter the pings array from seed.json where
vehicle_id matches, sort by timestamp descending, take the first result.
Return 404 if no pings exist for this vehicle.

Route path must be lowercase with hyphens: /last-position,
not /lastPosition or /last_position.

Response shape:
{
  "vehicle_id": string,
  "timestamp": ISO 8601 string,
  "lat": number,
  "lng": number,
  "speed": number
}
```

### Two generator mistakes to catch

**Generator mistake 1 - Verb in the route path**

Generators frequently produce paths like `/getLastPosition`, `/vehicles/:id/calculatePosition`, or `/vehicles/:id/lastPosition`. The WSO2 §5.1 naming rules you applied in S3 apply here too: route paths use lowercase, hyphen-separated nouns. Processing function resources are named as verbs in conceptual terms (they represent an action), but the URI must still follow the lowercased, hyphenated convention. The path is `/last-position`, not a camelCase verb.

Spot this in your generated code before you run it. If you see `/lastPosition` or `/getLastPosition` in the route path string, fix it before testing.

**Generator mistake 2 - Returning the full vehicle composite shape**

Some generators, having just built the composite in Layer 2, will reuse the composite shape for the `/last-position` route. The position-only shape is `{ vehicle_id, timestamp, lat, lng, speed }`. It does not include `reg_number`, `device_id`, `station_id`, or a `last_ping` wrapper. These vehicle metadata fields belong to the composite route, not to the processing function.

### Bonus critique: the helper function (not a checkpoint requirement)

If your generated code for Layer 2 and your generated code for Layer 3 both contain identical logic to find the most recent ping - filtering, sorting descending, taking `[0]` - you have duplicated logic. This means if the sort logic ever needs to change, you must change it in two places. The solution is to extract a helper function:

```javascript
function getLastPing(vehicleId) {
  const pings = data.pings
    .filter(p => p.vehicle_id === vehicleId)
    .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
  return pings[0] || null;
}
```

Both routes can then call `getLastPing(req.params.vehicleId)`. The composite uses it as the value for `last_ping`, and the `/last-position` route returns it directly (destructured to position fields only, or returned as-is if the shape already matches).

Extracting this helper is not a checkpoint requirement - the checkpoint only requires that both routes return the correct shapes. But it is the kind of code reading that the micro-viva question 4 will probe, and it is a concrete marker that you engaged with your generated output rather than just deploying it.

**Coursework connection:** `/last-position` is a processing function resource (WSO2 §4.5). Explaining why it exists as a separate resource (client needs, avoiding over-fetching), naming it correctly, and implementing the correct shape are all part of the API Design dimension. The helper function refactor is good evidence for the Architecture dimension.

---

## Fallback - consolidated prompt

Use the following prompt only if your generated code fails to compile or produces wrong shapes across multiple layers. It is not a shortcut for Layer 1. The fallback produces working code that you must then read, understand, and be able to explain at micro-viva.

```
Rewrite the Express routes file with the following behaviour. Data is loaded
from seed.json at startup:
  const data = JSON.parse(fs.readFileSync('./seed.json', 'utf8'));

Helper function to get the most recent ping for a vehicle:
  function getLastPing(vehicleId) {
    const pings = data.pings
      .filter(p => p.vehicle_id === vehicleId)
      .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
    return pings[0] || null;
  }

Routes:
  GET /provinces -> res.json(data.provinces)
  GET /provinces/:id -> res.json(data.provinces.find(
    p => p.province_id === req.params.id))
  (repeat pattern for /districts and /stations)
  GET /vehicles -> res.json(data.vehicles)
  GET /vehicles/:id -> const v = data.vehicles.find(
    v => v.vehicle_id === req.params.id);
    res.json({ ...v, last_ping: getLastPing(req.params.id) })
  GET /vehicles/:id/pings -> res.json(
    data.pings.filter(p => p.vehicle_id === req.params.id))
  GET /vehicles/:id/last-position -> res.json(getLastPing(req.params.id))

All routes use res.json(). Route paths: lowercase, hyphens, plural collections,
no verbs. No envelope wrappers.
```

After using the fallback, read through the generated code and make sure you can answer all five micro-viva questions before the checkpoint.

---

## Day 3 Checkpoint - binary pass criteria

All six criteria must pass before the session closes.

**[1]** All atomic routes return S5-authoritative snake_case shapes with no envelope wrappers and correct FK fields present.

**[2]** `GET /vehicles/:vehicleId` returns the composite shape: all vehicle fields plus `last_ping` as a nested object (the most recent `LocationPing` sorted by timestamp descending), or `null` if no pings exist for the vehicle.

**[3]** `GET /vehicles/:vehicleId/pings` returns an array of `LocationPing` shapes. (This route should not have changed from S4; this criterion confirms the shape is correct.)

**[4]** `GET /vehicles/:vehicleId/last-position` returns the position-only shape: `{ vehicle_id, timestamp, lat, lng, speed }`. The route path is `/last-position` (lowercase, hyphenated, no verb).

**[5]** All responses have `Content-Type: application/json`. Verify in your REST client response headers, not just by looking at the body.

**[6]** Changes committed to the repository. Public URL updated. The instructor can reach all eleven routes on the public deployment.

---

## Micro-viva questions (Day 3)

You have five minutes. Have your deployed API open and your code visible.

**Q1.** Show me `GET /vehicles/{one of your vehicle IDs}`. Is `last_ping` a nested object or flat fields? What would be wrong with returning `last_lat` and `last_lng` as direct fields on the vehicle object instead?

**Q2.** Show me `GET /vehicles/{same ID}/last-position`. What is different about this response compared to the composite? When would a client choose one over the other?

**Q3.** Your code finds the most recent ping for the composite. How does it know which ping is most recent? Show me that logic in the code.

**Q4.** Does your `/last-position` route contain the same logic as your composite route? If yes, is there a way to avoid duplicating it?

**Q5.** Check the response headers for any of your routes. What `Content-Type` is your API returning, and what in your code produces it?

---

## Key Terms

**Composite resource (WSO2 §4.3):** A resource that aggregates instances of multiple entity types that are typically retrieved or manipulated together as a whole. In this system: the vehicle composite embeds the most recent `LocationPing` within the vehicle response.

**Processing function resource (WSO2 §4.5):** A resource that provides access to a computation or derived value rather than a stored entity. In this system: `GET /vehicles/:vehicleId/last-position` derives and returns the current position without returning vehicle metadata.

**Representation (WSO2 §6):** The format and structure (the "shape") in which a resource's information content is exchanged. The representation decision was made in S5: JSON, `application/json`, field names in snake_case. This session makes those decisions final and authoritative across all routes.

**Snake_case:** A naming convention where words in compound identifiers are separated by underscores. Example: `vehicle_id`, `reg_number`, `station_id`. WSO2 §6 mandates consistency; generators default to camelCase (`vehicleId`) which must be corrected.

**FK field (Foreign key field):** A field in one resource's representation that holds the identifier of a related resource in another collection. Example: `province_id` on a District object links the district to its parent Province. These fields are how API consumers traverse the hierarchy.

**Envelope wrapper:** A container object added around a resource's representation, typically including metadata like status codes or counts. Example: `{ "data": [...], "status": "success" }`. WSO2 §6 does not specify envelope wrappers for standard representations; remove them.
