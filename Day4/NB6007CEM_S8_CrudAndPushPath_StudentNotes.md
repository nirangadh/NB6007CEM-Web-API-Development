# NB6007CEM - Web API Development
## Session 8 Student Notes: CrudAndPushPath
**Day 4 PM - Build Session**

---

## 1. What this session delivers

By the end of Day 4 your deployed API does four things it could not do at the start of the day. Listed in build order:

1. Every GET route that looks up a resource by `:id` returns 404 when the resource is not found, not 200 with a null body.
2. Devices can push location pings to your API using an API key, and your POST response carries the correct 201 status code plus the Location, ETag, and Last-Modified headers.
3. Police officers must supply Basic Auth credentials to read any data; unauthenticated GET requests receive a 401 response with the WWW-Authenticate header.
4. (If time permits) New vehicles can be registered, existing vehicles updated or removed, and pings remain permanently on the server even when the vehicle they belong to is deleted.

These four increments advance the API design, security, and architecture assessment dimensions simultaneously.

---

## 2. The four-layer run sheet

| Layer | Time | What it adds |
|-------|------|--------------|
| 1 | 25 min | Member 404 on all `:id` routes |
| 2 | 55 min | POST /vehicles/:id/pings with X-API-Key auth |
| 3 | 35 min | Basic Auth middleware on all GET routes |
| 4 | 30 min | Vehicle CRUD - POST, PUT, DELETE on /vehicles |

**Compression rule.** If you fall behind, move Layer 4 to homework before cutting anything else. Never compress Layer 2 (the push path is the anchor for all other auth work) or drop the WWW-Authenticate requirement from Layer 3.

---

## 3. REST client prerequisite

A browser address bar only sends GET requests. The moment you add an X-API-Key header or a Basic Auth Authorization header, you need a REST client that lets you set headers manually.

Check this before writing a single line of code: make a GET /vehicles call in your REST client and confirm the response arrives. If it does not, fix the client before touching the server.

Suitable clients: Postman, Insomnia, Thunder Client (VS Code extension), or `curl` in a terminal.

---

## 4. Route reference card

### Read path - Basic Auth required

All GET routes require a valid Authorization header after Layer 3.

```
GET /provinces
GET /provinces/:id
GET /districts
GET /districts/:id
GET /stations
GET /stations/:id
GET /vehicles
GET /vehicles/:id
GET /vehicles/:id/pings
GET /vehicles/:id/pings/:pingId   (new this session)
GET /vehicles/:id/last-position
```

Status codes: 200 | 401 (missing/invalid credentials) | 403 (credentials wrong) | 404 (member not found)

### Write path - X-API-Key required

```
POST /vehicles/:vehicleId/pings   201 + Location | 400 | 401 | 403 | 404
```

### Admin path - Basic Auth required (Layer 4, droppable)

```
POST /vehicles          201 + Location | 400 | 401
PUT  /vehicles/:id      200 | 400 | 401 | 404
DELETE /vehicles/:id    200 | 401 | 404
```

### Deferred (do not build today)

Conditional GET (If-None-Match / 304) is Session 9. Pagination and filtering are Session 9. The standardised §11 error schema is Session 11. OAuth and role scopes are Session 13.

---

## 5. Layer 1 - Member 404

### The rule

A collection GET always returns 200 even when the collection is empty. An empty array is a valid state, not an absent resource. A member GET returns 404 when the requested ID is not in the data store.

```javascript
// WRONG - returns 200 with undefined body when ID not found
const v = data.vehicles.find(v => v.vehicle_id === req.params.id);
res.json(v);

// CORRECT
const v = data.vehicles.find(v => v.vehicle_id === req.params.id);
if (!v) return res.status(404).json({ error: 'Vehicle not found' });
res.json(v);
```

Apply this pattern to every `:id` route: province, district, station, vehicle, ping.

### Error body boundary

Today's goal is the correct status code, not a perfect error body. `{ "error": "Vehicle not found" }` is sufficient for this session. The standardised error schema specified in the design guidelines (with `code`, `message`, and `moreInfo` fields) is covered in Session 11.

### Generator mistake to catch (WSO2 §9)

Code generators frequently return `res.json(variable)` after a `.find()` call without checking for `undefined`. This produces a 200 response with a `null` or empty body when the ID does not exist. Scan every member GET route in your generated code before moving to Layer 2.

**Coursework connection.** Correct status codes are assessed under the API design dimension (WSO2 §9). A member GET that returns 200 with a null body is a design error that loses marks regardless of whether the rest of the route works correctly.

---

## 6. Layer 2 - The push path (THE ANCHOR)

This layer is the most important build of the session. Every subsequent auth layer depends on the POST route working correctly first.

### The resource being created

A LocationPing is an append-only time-series record. When a tuk-tuk's GPS device calls `POST /vehicles/:vehicleId/pings`, it creates a new ping and the server assigns a unique `ping_id`. The server sets the timestamp at receipt time (`new Date().toISOString()`) unless the device body includes a `timestamp` field, in which case the server accepts the device-supplied value. The trade-off: server-set timestamps record time of receipt and are more trustworthy for audit purposes; device-supplied timestamps record the GPS event time but can be spoofed. Either choice is defensible - what matters is that you can explain the one you implemented.

### The generate prompt

Ask your AI to generate the following. Read the output carefully before running it.

> Generate POST /vehicles/:vehicleId/pings for an Express server.
>
> Build a deviceKeys object from the seeded vehicles: `const deviceKeys = { 'v-01': 'key_v01', 'v-02': 'key_v02', ... }`
>
> Require X-API-Key header. Return 401 if header absent. Return 403 if key does not match `deviceKeys[vehicleId]`. Return 404 if vehicleId not in vehicles array. Return 400 if body missing latitude, longitude, or speed.
>
> Server sets timestamp: `new Date().toISOString()`. Push ping to pings array. Return 201 with Location `/vehicles/:vehicleId/pings/:pingId`, ETag (quoted ping_id), and Last-Modified header.
>
> Also add GET /vehicles/:vehicleId/pings/:pingId returning 200 or 404.

### The 401 vs 403 split (WSO2 §12)

These two codes are not interchangeable.

**401 Unauthorized** means the client has not provided any credentials, or the credentials provided cannot be parsed. The client does not know whether the request would be accepted. The response must include a `WWW-Authenticate` header.

**403 Forbidden** means the client has been identified but is not permitted to perform the requested action. The API key header is present and valid, but it belongs to a different vehicle than the one in the path.

Generator mistake to catch: code generators frequently return 401 in both cases. Check whether your generated middleware distinguishes between the two scenarios.

### The 201 rule (WSO2 §7.3 and §9)

When a POST creates a resource successfully, the response code must be 201, not 200. A 200 is the correct code for a successful GET, PUT, or POST that returns a result without creating anything new.

Additionally, the response to a successful POST must include a `Location` header containing the URI of the newly created resource. For this endpoint, that URI is `/vehicles/:vehicleId/pings/:pingId`.

Verify this immediately: after your POST returns, copy the Location header value and make a GET request to that URL. If it returns 200 with the ping body, the 201 is correct. If it returns 404, the POST is lying about what it created.

### Generator mistakes to catch (WSO2 §7.3, §8, §9)

- Returns 200 instead of 201
- Location header absent from the response
- Location header present but pointing to a path that does not exist as a GET route
- ETag or Last-Modified headers missing
- 401 returned when the key belongs to a different vehicle (should be 403)
- vehicle_id taken from the request body instead of the URL path parameter

**Coursework connection.** The POST /vehicles/:id/pings route is assessed under API design (§7.3, §9, §8), security (§12), and architecture (the write-read split). Returning 200 instead of 201 and missing the Location header are both specific deductions in the rubric.

---

## 7. Layer 3 - Basic Auth on read routes (WSO2 §12.1)

### The generate prompt

Ask your AI to generate a Basic Auth middleware for Express.

> Generate an Express middleware function called `basicAuth` that:
>
> Reads the Authorization header. If absent: return 401 with `WWW-Authenticate: Basic realm="Police API"`.
>
> Decodes the Base64 credentials. If username is not "police" or password is not "nibm2024": return 403.
>
> Otherwise call next(). Apply this middleware before every GET route handler. Do not apply it to POST /vehicles/:vehicleId/pings.

### WWW-Authenticate requirement (WSO2 §12.1)

When a server returns 401 it must include a `WWW-Authenticate` header. This tells the client which authentication scheme to use and which realm (protected area) the request relates to. Without it, a standards-compliant client cannot automatically prompt for credentials.

Generator mistake to catch: generated code frequently returns 401 without including the `WWW-Authenticate` header. Inspect the response headers of your first unauthenticated GET before moving on.

### Scoping boundary

After Layer 3, a valid police officer with correct credentials can see all vehicles across all provinces and districts. That is expected for this session. Jurisdiction filtering - where a station officer sees only their own station's vehicles - is part of the OAuth and role scopes content in Session 13.

### Generator mistakes to catch (WSO2 §12.1)

- `WWW-Authenticate` header missing from 401 response
- Middleware registered in Express after the route handlers (order matters: middleware must come before the route it guards)
- Basic Auth applied globally including to the POST /vehicles/:vehicleId/pings route (which uses a different auth scheme)
- Credentials decoded but not validated (any Authorization header value passes)

**Coursework connection.** Authentication is assessed under the security dimension (WSO2 §12). A 401 response missing WWW-Authenticate is an HTTP standards violation that is specifically checked in the rubric.

---

## 8. Layer 4 - Vehicle CRUD (WSO2 §7)

> **Droppable.** If Layers 1-3 overran, move this layer to homework and complete it against the reference build. Layers 1-3 are the Day 4 checkpoint gate.

### POST /vehicles (WSO2 §7.3)

Creating a new vehicle follows the same pattern as creating a ping. The response must be 201 with a Location header pointing to the new vehicle's URI. Validate that the body contains the required fields (vehicle_id, plateNumber, vehicleType, stationId) and return 400 if any are missing.

### PUT /vehicles/:id (WSO2 §7.2)

PUT replaces the entire resource, not just the fields in the request body. If a field is missing from the PUT body it is removed from the stored resource, not preserved. This is the whole-document semantics of PUT. Generators frequently implement PUT as a partial update (merging the body into the existing document) which is incorrect - that behaviour belongs to PATCH.

### DELETE /vehicles/:id (WSO2 §7.4)

A successful DELETE returns 200. A second DELETE on the same URI returns 404, because the resource no longer exists.

### Ping immutability

When a vehicle is deleted, its pings remain in the pings array. Do not cascade-delete. Location pings are an append-only audit log: GPS records are evidence of a tuk-tuk's movement history. Deleting a vehicle removes it from active service but must not erase historical tracking data.

**Analogy.** Pings are the event log. Vehicles are the subject file. You can close a case file but you cannot redact the event log. Deletion is not erasure.

**Coursework connection.** For your report, explain why you chose to retain or cascade-delete pings on vehicle deletion and what the business consequence of each choice is. The design choice itself does not determine marks; the justification does.

**Generator mistakes to catch (WSO2 §7.2, §7.4)**

- PUT implements partial update (merge) instead of whole-document replacement
- DELETE returns 200 on second call instead of 404
- A PUT or DELETE route is generated for pings (there must be none)

---

## 9. Generator mistakes - consolidated

Before you test any layer, scan your generated code for these:

**Layers 1 and 2**
- Collection GET returning 404 on empty result (should be 200 with `[]`)
- POST /pings returning 200 instead of 201
- Location header absent or pointing to a non-existent route
- ETag or Last-Modified missing from 201 response
- 401 and 403 used interchangeably (missing header is 401; wrong key is 403)
- vehicle_id taken from the request body instead of the URL path

**Layers 3 and 4**
- `WWW-Authenticate` header absent from 401 response
- Auth middleware registered after the route handler in Express
- Basic Auth applied to the POST /pings route (wrong scheme)
- DELETE /vehicles also deletes pings (immutability violation)
- Server timestamp overwritten by a timestamp field in the device body

---

## 10. Day 4 Checkpoint

All items must be YES before you commit and redeploy:

- [ ] POST /vehicles/:id/pings returns 201 (not 200)
- [ ] Location header present and resolves to the correct ping URL
- [ ] Missing X-API-Key header returns 401; wrong key returns 403
- [ ] GET /vehicles without credentials returns 401 with WWW-Authenticate header
- [ ] GET /vehicles with Basic Auth (police:nibm2024) returns 200
- [ ] GET /vehicles/:id/pings/:pingId returns 200 or 404 correctly
- [ ] Committed and redeployed to Render - all routes responding

---

## 11. Key Terms

**201 Created.** The HTTP status code for a successful POST that creates a new resource. The response must include a Location header.

**401 Unauthorized.** The client has not provided credentials or the credentials cannot be parsed. The response must include a WWW-Authenticate header.

**403 Forbidden.** The client has been identified but is not permitted to perform the requested action. No WWW-Authenticate header is required.

**Basic Authentication.** An HTTP authentication scheme in which the client encodes username:password in Base64 and sends it in the Authorization header. Only secure over HTTPS.

**deviceKeys.** An object mapping each vehicle_id to its API key. Used to authenticate device push requests.

**ETag.** An entity tag header returned with a resource response, representing a version identifier. Clients can use it for conditional requests.

**Idempotent.** A request that produces the same server state regardless of how many times it is sent. PUT and DELETE are idempotent; POST is not.

**Last-Modified.** A response header indicating when the resource was last changed. Clients can use it for conditional GET requests.

**Location header.** A response header containing the URI of a newly created resource. Required on all 201 responses.

**Middleware.** A function in Express that runs before route handlers. Auth middleware reads the request headers, validates credentials, and calls `next()` to pass control to the route handler or returns an error response.

**Ping immutability.** The design rule that LocationPing records cannot be updated or deleted once created. They are append-only audit records.

**WWW-Authenticate.** A response header that must accompany every 401 response. It tells the client which authentication scheme to use (e.g. `Basic realm="Police API"`).

**X-API-Key.** A custom HTTP request header used to carry the device's API key for the push path.
