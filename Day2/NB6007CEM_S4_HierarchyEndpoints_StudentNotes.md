# NB6007CEM - Web API Development
## Session 4 Student Study Notes: Hierarchy Endpoints
**Day 2 (PM) - Build Session**
*BSc (Hons) Computer Science with Software Engineering - Coventry University, delivered at NIBM*
*Niranga Dharmaratna - niranga@nibm.lk*

---

## What This Session Delivers

Session 3 produced a complete, validated route map for the Tuk-Tuk Monitoring System. Session 4 turns that map into running code. By the end of this session your public API returns real seed data from nine endpoints. That is the Day 2 checkpoint and the foundation everything from S5 onwards is built on.

---

## Before You Generate: The Representation Boundary

This is the most important thing to read before touching your AI tool.

Your generator will produce JSON response bodies as soon as you ask for Express routes. That is expected and fine. But the field names, nesting depth, and array structure in those response bodies are choices the generator made - not choices you have made. We have not formally decided our representation yet.

**Session 5 is where the representation decision happens.** Until then, treat response shapes as provisional.

Why does this matter at viva? Field names you submit in your final API are design decisions you must be able to justify. "The AI generated them" is not a justification. S5 gives you the framework to make and defend those choices. For now: wire the routes, get the data flowing, and stay aware that the shapes you see today are not yet locked in.

---

## The Nine Routes

The complete set of endpoints wired in this session:

| URI | Resource type |
|---|---|
| `GET /provinces` | Collection |
| `GET /provinces/:province-id` | Atomic member |
| `GET /districts` | Collection |
| `GET /districts/:district-id` | Atomic member |
| `GET /stations` | Collection |
| `GET /stations/:station-id` | Atomic member |
| `GET /vehicles` | Collection |
| `GET /vehicles/:vehicle-id` | Atomic member |
| `GET /vehicles/:vehicle-id/pings` | Scoped collection |

Scope today: GET only. No authentication. No status codes beyond 200 and 404. Full CRUD arrives in S7-S8.

---

## Step 1: Generate Your Routes

Copy this prompt exactly into your AI tool. Read the full output before running `node`.

```
Add Express.js GET routes for the following resources.
Serve data from seed.json loaded into memory at startup.
Route paths must follow REST API Design §5.1 naming rules:
  lowercase, hyphens as separators, plural nouns for
  collections, singular nouns for members, nouns not verbs.
Do not add a database. Do not add authentication.
Routes to add:
  GET /provinces
  GET /provinces/:provinceId
  GET /districts
  GET /districts/:districtId
  GET /stations
  GET /stations/:stationId
  GET /vehicles
  GET /vehicles/:vehicleId
  GET /vehicles/:vehicleId/pings
```

Keep the S3 URI-naming linter open in a browser tab while you work. Run every generated path through it before testing locally.

---

## Step 2: Critique Before You Run

Read the generated code against the §5.1 checklist before starting `node`. The S3 linter is your tool.

### The §5.1 Naming Checklist

**Lowercase only.** Path segments use lowercase characters only. `/Vehicles` is a violation. `/vehicles` is correct.

**Hyphens, not underscores or camelCase.** `/police_stations` and `/policeStations` are violations. `/police-stations` is correct.

**Plural nouns for collections.** `/province` is a violation. `/provinces` is correct. The plural signals this is a collection resource, not an individual member.

**Singular nouns for members.** `/vehicles/:vehicleId` is correct. A path parameter identifies one member of the collection. The collection path stays plural.

**Nouns, not verbs.** `GET /getVehicles` is a violation. `GET /vehicles` is correct. The HTTP method `GET` is already the verb; the path segment is a noun.

---

## The Pings Scoping Rule - §4.6 and §5.6

A global `GET /pings` route is the most common generator mistake on this session. You need to understand why it is wrong and how to fix it.

### What the generator produces (wrong)

```javascript
// Added uninstructed:
app.get('/pings', (req, res) => {
  res.json(data.pings);
});
```

This returns every ping in the system regardless of which vehicle it belongs to. It violates §5.6: the collection of all pings in all vehicles is not of interest - only pings scoped to a specific vehicle are. A police officer looking at the system wants to know where tuk-tuk WP-1234 has been, not the entire movement history of all 200+ vehicles in one response.

### The correct scoped form

```javascript
app.get('/vehicles/:vehicleId/pings', (req, res) => {
  const p = data.pings.filter(
    x => x.vehicle_id === req.params.vehicleId
  );
  res.json(p);
});
```

This returns only pings for the vehicle identified by `:vehicleId`. The §4.6 principle here is about interpreting relationships in the data model: the Vehicle-to-LocationPing relationship means that pings only make sense as children of a specific vehicle. There is no use case for all pings globally.

**Fix: remove `GET /pings` entirely.** It is a design error, not a bonus endpoint. Micro-viva Q2 asks you to explain why.

---

## Seed Loading: Load Once at Startup

If your generator reads `seed.json` inside a route handler, move it to startup.

### Anti-pattern (reads per request)

```javascript
// File read on every request - wrong
app.get('/vehicles', (req, res) => {
  const d = JSON.parse(
    fs.readFileSync('./seed.json', 'utf8')
  );
  res.json(d.vehicles);
});
```

### Correct pattern (load once at startup)

```javascript
// Read once when the server starts
const data = JSON.parse(
  fs.readFileSync('./seed.json', 'utf8')
);

app.get('/vehicles', (req, res) => {
  res.json(data.vehicles);
});
```

The anti-pattern reads the file from disk on every incoming request. With seed data this technically works but wastes I/O on every call. The correct pattern reads once at startup and holds the parsed object in memory. All route handlers then use the already-parsed `data` variable.

**Critique question:** search your generated file for `fs.readFileSync`. How many times does it appear, and where? One call at the top of the file before `app.listen()` is correct. One call inside each route handler is the anti-pattern.

---

## What to Look for in Generated Code

Run this checklist against your generated file before deploying:

**Verbs in route names.** `app.get('/getVehicles')` or `/fetchDistricts`. §5.1: path segments are nouns. `GET` is already the verb. Fix: rename to `/vehicles` and `/districts`.

**Database boilerplate inserted.** `mongoose.connect()`, knex, pg, or Sequelize models added despite the prompt instruction. Fix: delete the database block entirely. The prompt specified `seed.json` only.

**Global GET /pings route.** A bare `/pings` route returning all pings in the system. §5.6: pings are scoped to a vehicle. Fix: remove it. The only correct form is `/vehicles/:vehicleId/pings`.

**seed.json read inside route handlers.** `fs.readFileSync` inside each `app.get()` callback. Fix: move the read to the top of the file before `app.listen()`. One call, one variable, reused by all routes.

---

## Fallback Prompt

If generated routes do not serve seed data after one fix attempt, use this prompt verbatim:

```
Generate an Express.js file that:
1. Loads seed.json once at startup using fs.readFileSync.
2. Adds these GET routes, all returning JSON from seed data:
   GET /provinces              - return all provinces
   GET /provinces/:provinceId  - return one province by id
   GET /districts              - return all districts
   GET /districts/:districtId  - return one district by id
   GET /stations               - return all stations
   GET /stations/:stationId    - return one station by id
   GET /vehicles               - return all vehicles
   GET /vehicles/:vehicleId    - return one vehicle by id
   GET /vehicles/:vehicleId/pings  - return all pings where
     vehicle_id matches vehicleId
3. All route paths use lowercase and hyphens only.
4. No database, no auth, no middleware except express.json().
```

If this also fails: obtain the reference build file from the instructor and critique it against the §5.1 checklist and the pings scoping rule. Critiquing working code demonstrates the assessed skill just as effectively as generating it.

---

## Day 2 Checkpoint - Binary Pass

All five must pass before the micro-viva begins. If any fail, fix and redeploy first.

**1. All 9 routes live and returning data from seed.json.**
`GET /provinces`, `/provinces/:id`, `/districts`, `/districts/:id`, `/stations`, `/stations/:id`, `/vehicles`, `/vehicles/:id`, `/vehicles/:id/pings` - all returning 200 with real seed data.

**2. All route paths pass §5.1.**
Confirmed with the linter or manual check: lowercase, hyphens, plural collections, no verbs. No `/getVehicles`. No bare `/pings`.

**3. `/vehicles/:id/pings` returns scoped pings only.**
The response contains pings for that vehicle only, not all pings globally. Test with two different vehicle IDs and verify the results are different.

**4. seed.json loaded once at startup.**
`fs.readFileSync` appears once at the top of the file before `app.listen()`. Not inside any route handler.

**5. Committed and deployed to public URL.**
Changes committed to the repository. The instructor can hit all 9 endpoints at your public URL. A localhost-only submission is only accepted if the hosting platform is down; it must be resolved at the start of S5.

---

## Micro-Viva Questions - Day 2

These questions are visible in advance. Answer at your deployed API, not from memory.

**Q1.** Show me `GET /vehicles` returning data from your seed. How many vehicles does it return?

**Q2.** Show me `GET /vehicles/{one-of-your-vehicle-ids}/pings`. Why is this a scoped collection and not `GET /pings`?

**Q3.** Your generator probably produced at least one §5.1 violation. What was it? Which naming rule did it break?

**Q4.** Where in your code does `seed.json` get loaded? Why does it matter whether it loads once or per request?

**Q5.** What is your public URL? Show me any two endpoints returning 200 with real seed data.

---

## Pace Extension Work

If you finish all 9 routes and the checkpoint criteria before the end of the build period, add query-string filtering to `/vehicles`:

```
GET /vehicles?province-id=1
```

Filter vehicles by their province. This is **preview work only** - it is explicitly out of scope for assessment until S9. Label it as such in your repository with a comment. Do not submit it as part of your S4 checkpoint. The purpose is to stay engaged and get a head start on the filtering concept.

---

## What Comes Next: S5

Session 5 resolves the open question from today. The field names and shapes in your response bodies are generator choices. S5 introduces the formal representation decision: why JSON rather than XML, and how the JSON structure should be designed according to §6 of the design guidelines. After S5 you will be able to go back to your seed-serving routes and justify or fix the shapes they return.

---

## Coursework Connection

The routes you wire today directly build the **Architecture and data model** and **Deployment and operation** assessment dimensions. Nine live endpoints at a public URL is the deployment baseline. The `seed.json` loading pattern and the scoped `/pings` route are the first two things an assessor or viva examiner will check: they are simple, binary, and visible immediately.

The representation shapes are **not yet** part of your design — that comes in S5 via the **API design** dimension. Do not treat today's generator output as finished design work.

WSO2 sections in scope today: §4.1, §4.2, §4.6, §5.1, §5.6.

---

## Key Terms

**Scoped collection.** A collection resource whose members only make sense in the context of a parent resource. `/vehicles/:vehicleId/pings` is a scoped collection: pings only exist meaningfully within a specific vehicle, not as a global set.

**Atomic resource.** A single identifiable entity from the data model. `/vehicles/:vehicleId` identifies one vehicle. The atomic resource is the member; the collection resource groups them.

**URI template.** A URI pattern containing variables in curly brackets, e.g. `/vehicles/{vehicle-id}`. In Express, written as `/vehicles/:vehicleId`. The variable is substituted by the client with a specific value.

**Seed data.** Pre-populated static data loaded from `seed.json` at server startup. Used throughout the block so every student has a consistent, realistic dataset to serve from their API without needing a database.

**Representation boundary.** The deliberate distinction between the routes (decided today) and the response shapes (decided in S5). Routes are URI design. Representation is what the body contains and how it is structured.
