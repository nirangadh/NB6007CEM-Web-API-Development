# NB6007CEM Web API Development
## Session 7 Study Notes: HTTP Methods, Headers and Status Codes
**Day 4 (AM) | WSO2 Design Guidelines §7, §8, §9, §12 (intro)**

---

## Two Deferred Threads, Now Delivered

Since Session 3 every route in the tuk-tuk API has used GET, and each slide said "Session 7 explains exactly why." Since Session 1 the design principle "the device writes, the police read" has been stated but not enforced. Both of those commitments are closed in this session.

---

## The Design Pipeline Position

The WSO2 design pipeline runs: **Data Model → Derive Resources → Decide Representations → Name by URIs → Determine HTTP Methods → Determine Special Behaviour → Consider Errors → Security → Maturity.** Session 7 covers steps 5 and 6 together because an HTTP method specification is incomplete without the status codes and headers that accompany it. Specifying a route means specifying the full response contract.

---

## §7 HTTP Methods and Their Properties

### Safe and Idempotent

Two properties govern HTTP methods:

**Safe** means the request produces no side effects on the server. A safe request can be cached, prefetched, and retried by any intermediate node (browser, proxy, load balancer) without risk.

**Idempotent** means calling the request once produces the same server state as calling it five times. If a request fails mid-flight, the client can simply retry.

> **ANALOGY:** A lift-call button is like GET or PUT: press it five times and the lift still comes once. A coin-slot turnstile is like POST: each press is a separate charge with a separate effect. Safe and idempotent methods can be retried by any intermediate node without consequence; POST cannot.
>
> *Limit:* a lift button tracks whether the lift is already coming; an HTTP server does not de-duplicate retries automatically. The responsibility to avoid unnecessary retries belongs to the client.

### The Four Methods

| Method | Safe? | Idempotent? | Key rule |
|---|---|---|---|
| GET | Yes | Yes | All read routes. A police app retrying a read cannot create a duplicate ping. |
| PUT | No | Yes | Replace the complete resource. Five calls produce the same final state as one. |
| DELETE | No | Yes* | Remove the resource. *First call returns 200; subsequent calls return 404. |
| POST | No | No | Create a new resource or trigger an action. Each call may create a new resource. |

### Why GET is Always Correct for Read Routes

GET is the only safe method, meaning it cannot produce side effects. A police monitoring app may retry a failed request automatically. If that route used POST instead of GET, a transient network failure followed by an automatic retry could silently create a duplicate ping record. Using GET makes it structurally impossible for a read route to have write side effects.

### Generator Mistakes: HTTP Methods

**Mistake 1: POST for update operations.**
POST is not idempotent. If a generator uses POST to update a vehicle record (`POST /vehicles/WP-1234` with a full body), calling it five times in a poor network may create five records. PUT is the correct method for replacement: calling `PUT /vehicles/WP-1234` five times with the same body produces the same vehicle record every time. WSO2 §7.2.

**Mistake 2: GET with a request body.**
GET is safe. Its semantics do not include a body. A generator that produces `GET /vehicles` with a JSON body for filtering is violating the safe contract and will behave inconsistently across HTTP clients and proxies. Filter criteria belong in the query string (§10.2, covered in Session 9) or in a POST request to a processing function resource (§4.5, already introduced in Session 5).

**Coursework connection:** Method choice is part of the API Design dimension. A justified route table naming the method and the reason it is correct scores above one that simply lists routes.

---

## §9 Status Codes: The Protocol Contract

Status codes are not error messages. They are the server's binding statement about what happened and what the client should do next. The client uses the status code to decide whether to retry, redirect, display an error, or follow a Location header.

### 2xx Success Codes

**200 OK** - The request succeeded. The response body contains the result. Correct for GET, and for POST requests that trigger an action without creating a new resource.

**201 Created** - A new resource was created. The `Location` header contains the URI of the newly created resource. The response should also include `ETag` and `Last-Modified`. This status code is mandatory for any POST that creates a resource. This is what generators almost always get wrong.

**202 Accepted** - The request has been accepted for asynchronous processing. Success is not guaranteed and the client must poll for the outcome. Covered fully in Session 11.

> **ANALOGY:** When you register a new SIM card, the counter does not just say "done." They hand you the number you have been assigned - that is where your new identity now lives. A 201 response must work the same way: the Location header tells the client exactly where the created resource can be found.
>
> *Limit:* a SIM number is assigned once and never changes. A LocationPing URI is permanent but is one entry in a time-series; the resource it points to is the specific ping, not the vehicle.

**The core rule:** a created resource must tell you where it now lives. A client that receives 200 from a POST does not know whether a resource was created, where it is, or whether the request did anything at all. The status code is not decoration.

### 4xx Client Error Codes

| Code | Name | Meaning | Generator mistake to avoid |
|---|---|---|---|
| 400 | Bad Request | Malformed body, missing fields, out-of-range values | Returning 200 with an error message in the body |
| 401 | Unauthorized | Credentials missing or rejected; response must include WWW-Authenticate | Using 401 and 403 interchangeably |
| 403 | Forbidden | Credentials valid but resource off-limits; client should not retry | - |
| 404 | Not Found | Resource does not exist | Returning 200 with null or an empty object |
| 406 | Not Acceptable | Accept header requests a format the server cannot return | - |
| 415 | Unsupported Media Type | Request Content-Type not supported | - |
| 412 | Precondition Failed | Conditional request failed (If-Match). Covered in Sessions 9-11. | - |

5xx codes denote server errors and are generic. They do not need to be documented per endpoint per WSO2 §9.

### 401 vs 403 - They Are Not the Same

**401 Unauthorized** means "Who are you? I do not recognise these credentials." The response must include a `WWW-Authenticate` header. The client is expected to retry with correct credentials.

**403 Forbidden** means "I know exactly who you are. You are not allowed here." No `WWW-Authenticate` header is needed. The client should not retry - permission is denied regardless of credentials.

**Generator mistake:** Using 401 and 403 interchangeably breaks the client protocol. A client that receives 401 expects to retry. A client that receives 403 knows not to bother.

Tuk-tuk examples:
- X-API-Key header missing from a device push → 401 (device can retry once it has the correct key)
- Station officer requests another district's vehicle list → 403 (identity confirmed, jurisdiction denied)

---

## §8 Headers: The Full Wire Picture

Headers carry the non-functional properties of an HTTP request or response. Understanding the full set of headers required by §7.3 and §9 is what separates a well-specified route from a skeleton.

### Request Headers

**Accept** - Media types the client can handle. Our API accepts and returns `application/json` only. A mismatch triggers 406.
```
Accept: application/json
```

**Authorization** - Credentials for authentication. The format depends on the scheme. For police read routes: `Authorization: Basic {base64(username:password)}`. For device write routes: a custom header `X-API-Key: dev-WP-1234-secret` is used in this implementation. Bearer token format belongs to Session 13 (OAuth).

**Content-Type** - Format of the request body for PUT and POST. Must be `application/json`. A mismatch triggers 415.
```
Content-Type: application/json
```

Conditional request headers (`If-Match`, `If-None-Match`, `If-Modified-Since`) are used for caching and concurrency control. They arrive in Sessions 9-11.

### Response Headers

**Content-Type** - Format of the response body. Express sets this automatically when you call `res.json()`.
```
Content-Type: application/json
```

**Location** - URI of the newly created resource. Mandatory in all 201 responses.
```
Location: /vehicles/WP-1234/pings/PNG-99999
```

**ETag** - A fingerprint of the resource at the moment of the response. Returned with 200 and 201 responses. Clients send it back in subsequent conditional requests. What clients do with ETag - `If-None-Match`, conditional caching, 304 Not Modified - is covered in Session 9.
```
ETag: "a3f9c2d1"
```

**Last-Modified** - Timestamp of the most recent modification. Companion to ETag. Returned with 200 and 201.
```
Last-Modified: Wed, 15 Jan 2025 10:30:00 GMT
```

**WWW-Authenticate** - Authentication scheme required. Mandatory in every 401 response.
```
WWW-Authenticate: Basic realm="tuk-tuk-api"
```

### The POST /vehicles/:id/pings Full Wire Picture

This is the most important route in the system. The contrast between what a generator produces and what §7.3 and §9 require shows what a correct HTTP implementation looks like.

**What generators typically produce:**
```
POST /vehicles/WP-1234/pings HTTP/1.1
Content-Type: application/json

{ "lat": 6.9271, "lng": 79.8612 }

HTTP/1.1 200 OK
Content-Type: application/json

{ "success": true }
```
The client learns nothing: no URI for the created resource, no fingerprint, no timestamp.

**What §7.3 and §9 require:**
```
POST /vehicles/WP-1234/pings HTTP/1.1
X-API-Key: dev-WP-1234-secret
Content-Type: application/json

{ "lat": 6.9271, "lng": 79.8612, "speed": 28 }

HTTP/1.1 201 Created
Location: /vehicles/WP-1234/pings/PNG-99999
ETag: "a3f9c2d1"
Last-Modified: Wed, 15 Jan 2025 10:30:00 GMT
Content-Type: application/json

{
  "ping_id": "PNG-99999",
  "vehicle_id": "WP-1234",
  "timestamp": "2025-01-15T10:30:00Z",
  "lat": 6.9271,
  "lng": 79.8612,
  "speed": 28
}
```

The difference is not stylistic. The status code, Location header, and response body are each specified by a different section of the guidelines.

---

## §12 Intro: First Authentication - Enforcing the Write-Read Line

### Why Two Schemes

The tuk-tuk system has two fundamentally different client types with different security requirements, and the authentication design reflects that distinction.

**Write path - IoT devices:**
Devices are embedded hardware, not human users. They cannot maintain a session or respond to login prompts. A per-device API key is practical: each key is bound to one vehicleId, the server validates that the key in the header matches the vehicle in the path, and a compromised key can be revoked individually.

```
X-API-Key: dev-WP-1234-secret
```

A device presenting WP-1234's key attempting to push a ping for WP-5678 receives 401. The key does not match the path.

**Read path - police users:**
Police users are humans using a software client. HTTP Basic auth requires only a username and password and is supported universally.

```
Authorization: Basic {base64(username:password)}
```

Missing or wrong credentials return `401` with `WWW-Authenticate: Basic realm="tuk-tuk-api"`. The client can then prompt the user and retry.

**Admin path - POST/PUT/DELETE /vehicles:**
Uses Basic auth. All authenticated users have equal admin access in Sessions 7-8. Role-based scoping (station officers vs HQ) is deferred to Session 13.

### Basic Auth Over HTTP is Unsafe

Base64 encoding is not encryption. The string `Zm9vOmJhcg==` decodes immediately to `foo:bar`. Basic auth over plain HTTP sends credentials in cleartext - every intermediate node between client and server can read the Authorization header.

HTTPS is what makes any HTTP authentication scheme safe. For this API on Render, HTTPS is provided automatically. The principle applies everywhere: if your server is HTTP-only, Basic auth is not acceptable for any non-trivial system.

> **Analogy:** Speaking your password aloud in a corridor - you reach the right person, but everyone nearby also hears it. HTTPS is a private room.

The Session 13 upgrade path moves from Basic auth to OAuth bearer tokens, which reduces exposure even if HTTPS is temporarily unavailable and allows fine-grained scope control.

---

## Demo: HTTP Status and Header Inspector

**File:** `NB6007CEM_S7_CrudStatusCodes_Demo_StatusInspector.html`

Open the file in any browser - no internet connection required.

**What to try:**
Pick each scenario from the left panel in turn. For each scenario, the right panel shows two columns: what a first-pass code generator typically returns (left), and what §7.3 and §9 require (right).

Start with **"Device pushes a ping (success)"** - this is the anchor scenario that shows the full 200 vs 201 contrast with all response headers.

Then try **"GET without Authorization header"** and note that 401 requires a `WWW-Authenticate` header while 403 does not.

**What it proves:** Status codes and headers are a specified contract, not an implementation detail. Any route that a generator produces must be checked against this standard before it is considered correct.

---

## Complete Route and Status Map (Session 8 Targets)

| Route | Method | Success | Key Errors | Auth |
|---|---|---|---|---|
| GET /provinces | GET | 200 | - | Basic |
| GET /provinces/:id | GET | 200, 404 | - | Basic |
| GET /districts | GET | 200 | - | Basic |
| GET /districts/:id | GET | 200, 404 | - | Basic |
| GET /stations | GET | 200 | - | Basic |
| GET /stations/:id | GET | 200, 404 | - | Basic |
| GET /vehicles | GET | 200 | - | Basic |
| GET /vehicles/:id | GET | 200, 404 | - | Basic |
| GET /vehicles/:id/pings | GET | 200, 404 | - | Basic |
| GET /vehicles/:id/last-position | GET | 200, 404 | - | Basic |
| POST /vehicles/:id/pings | POST | 201 + Location | 400, 401, 404 | API-key |
| POST /vehicles | POST | 201 + Location | 400, 401 | Basic |
| PUT /vehicles/:id | PUT | 200 | 400, 401, 404 | Basic |
| DELETE /vehicles/:id | DELETE | 200 | 401, 404 | Basic |

**Domain rules:**
- `LocationPing` is append-only. There are no PUT or DELETE routes for pings. GPS history is legal evidence; deletion is a compliance violation, not a design decision.
- `Province`, `District`, `PoliceStation` are read-only in this system. No write routes exist for these entities.

---

## Session 8 Checkpoint (Binary Pass)

All five criteria must pass before the micro-viva:

1. `POST /vehicles/:id/pings` returns **201 Created**, not 200
2. The 201 response includes a **Location** header pointing to the created ping URI
3. `GET /vehicles/:id` returns **404** (not 200 with null) when the vehicle does not exist
4. A GET request without an Authorization header returns **401** with a `WWW-Authenticate` header
5. A POST with a missing or wrong `X-API-Key` returns **401**

Deployment must be live on Render. Localhost is not accepted. Commit and redeploy before the micro-viva.

---

## Coursework Connection

**API Design (§7, §8, §9):** Every route in the deployed API must use the correct method, return the correct status code, and include the required headers. The report justification section should explain, for at least the key routes, why the method was chosen and what the response contract guarantees.

**Security (§12 intro):** The write-read authentication split must be explained in the report with the rationale for each scheme. The viva may ask you to explain the difference between Basic auth and API-key auth and why each was chosen for its path.

**Deployment and Operation:** The API must be publicly reachable on Render with every route returning the correct status codes.

---

## Key Terms Glossary

**Safe (HTTP method property)** - A request with no side effects on the server. Can be cached, prefetched, and retried freely. GET is the only safe method in this API.

**Idempotent (HTTP method property)** - A request where calling it once or many times with the same input produces the same server state. GET, PUT, and DELETE are idempotent; POST is not.

**201 Created** - HTTP status code indicating a new resource was successfully created. Must be accompanied by a `Location` header.

**Location header** - A response header containing the URI of a newly created resource. Mandatory with 201 responses.

**ETag (Entity Tag)** - A fingerprint of a resource's current state, returned in GET and 201 responses. Used in conditional requests for caching and concurrency control.

**WWW-Authenticate** - A response header that must accompany every 401 response, indicating the authentication scheme the client should use.

**HTTP Basic Authentication** - An authentication scheme where credentials (username:password) are Base64-encoded and sent in the Authorization header. Requires HTTPS to be safe.

**API-key authentication** - A scheme where a pre-shared secret key identifies a client (in this system, a specific device/vehicle). Sent as a custom header `X-API-Key`.

**Bearer token** - An OAuth access token carried in the Authorization header. Introduced in Session 13 as the upgrade from Basic auth.

**Content negotiation** - The process by which a client and server agree on the format of a response (via Accept and Content-Type headers).
