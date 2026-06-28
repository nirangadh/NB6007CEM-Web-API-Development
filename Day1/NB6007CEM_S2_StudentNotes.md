# NB6007CEM – Web API Development
## Day 1 (PM) – Session 2: Seed Data and Deployment
### Student Notes

---

## Where We Left Off (Slide 2)

By the end of Session 1 you agreed a domain model:

**Province → District → PoliceStation → Vehicle → LocationPing + User**

Key decisions locked in:

- `device_id` is an **attribute of Vehicle**, not a separate entity. A tuk-tuk has exactly one GPS device installed; there is no reason to manage devices independently.
- `LocationPing` is a **time-series table**, not a field on Vehicle. Each ping is a separate row with its own timestamp, latitude, and longitude.
- The WSO2 nine-step design pipeline was introduced. **Step 1 (data model) is complete.** The remaining eight steps are covered across Days 2–7.

**Boundary for today:** You will not write any endpoints, choose a media type, or design any URLs today. The only goal is to make the data real and get something running on a public host.

---

## Today's Three Outputs (Slide 3)

You must produce all three before the end of this session.

| Output | What it is | Done when |
|--------|-----------|-----------|
| `seed.json` | One JSON file with all five entity arrays | Committed to your GitHub repo |
| Express app | A single-file Node.js server with one route | `GET /` returns the expected JSON locally |
| Public URL | Your app deployed on Render | The URL returns `200 OK` in a browser |

**Run sheet:**

| Time | Activity |
|------|----------|
| 0:00 – 0:15 | Recap and orientation |
| 0:15 – 1:00 | Task 1 – Generate seed data |
| 1:00 – 1:45 | Task 2 – Hello-world Express app |
| 1:45 – 2:45 | Task 3 – Deploy to Render |
| 2:45 – 3:00 | Checkpoint and micro-viva |

---

## Task 1 – Generate Seed Data (Slides 4–5)

### What to do

1. Open your AI assistant.
2. Paste the full prompt below into it.
3. Save the output as `seed.json` in the root of your project.
4. Verify FK consistency before committing (see rules below).
5. Commit with `git add seed.json && git commit -m "S2: seed data"`.

### Prompt to use

Paste this in full:

> Generate seed data for a Sri Lanka Police Tuk-Tuk Monitoring System as a single JSON object with five keys: `provinces`, `districts`, `stations`, `vehicles`, and `pings`.
>
> Requirements:
> - 9 provinces: `id`, `name`
> - 25 districts: `id`, `name`, `province_id`
> - At least 20 stations: `id`, `name`, `district_id`
> - At least 200 vehicles: `id`, `registration_number`, `device_id`, `station_id`
> - At least 7 days of pings per vehicle: `id`, `vehicle_id`, `latitude`, `longitude`, `timestamp`
>
> CRITICAL: Every foreign key must reference an `id` that actually exists in the parent list. No orphaned records.

### FK consistency rules – what must be true

Every foreign key reference must resolve to an existing parent `id`. Specifically:

- Every `district.province_id` must match an `id` in the `provinces` array.
- Every `station.district_id` must match an `id` in the `districts` array.
- Every `vehicle.station_id` must match an `id` in the `stations` array.
- Every `ping.vehicle_id` must match an `id` in the `vehicles` array.

**Minimum scale:** 9 provinces, 25 districts, 20+ stations, 200+ vehicles, 7+ days of pings per vehicle.

**Timestamps** must be sequential and varied. If every ping has the same timestamp your data is useless for querying by time range.

### What generators get wrong

AI tools often produce flat lists where foreign keys reference IDs that do not exist. For example: `"district_id": 99` in a station record when there is no district with `id: 99`. Always scan the output before committing. If the chain is broken, paste the prompt again and ask the tool to fix the FK references explicitly.

---

## Task 2 – Express App (Slide 6)

### What to do

1. Open your AI assistant.
2. Use the fallback prompt below.
3. Save the output as `index.js` and `package.json` in your project root.
4. Run `npm install`.
5. Run `npm start`.
6. Open a browser and go to `http://localhost:3000/`.
7. Verify the response is exactly `{"status":"ok","session":"NB6007CEM S2"}`.
8. Commit with `git add . && git commit -m "S2: hello-world app"`.

### Fallback prompt (use this verbatim if generation fails)

> "Generate a minimal Express.js app with a single `GET /` route returning JSON `{status:'ok', session:'NB6007CEM S2'}`. Include `package.json` with a start script using `node`. No database, no other routes, no middleware."

### Before deploying to Render

Make sure your `app.listen` call uses the environment variable for the port:

```javascript
app.listen(process.env.PORT || 3000);
```

Render injects `PORT` dynamically. If you hardcode `3000`, the service starts locally but **fails on Render**.

---

## Task 3 – Deploy to Render (Slide 7)

### What to do

1. Push your repo to GitHub if you have not already done so.
2. Go to [render.com](https://render.com) and sign up using your GitHub account.
3. Click **New → Web Service → Connect a repository**.
4. Select your repo.
5. Set **Build command:** `npm install`
6. Set **Start command:** `node index.js`
7. Set **Instance type:** Free.
8. Click **Deploy** and watch the build logs.
9. Wait for the log to show `Listening on port` (or similar).
10. Copy your public URL and test it in a browser.
11. Share the URL with the instructor.

### Critical: port binding

```javascript
// Correct – Render sets process.env.PORT automatically
app.listen(process.env.PORT || 3000);

// Wrong – will fail on Render
app.listen(3000);
```

### Free tier note

Render free services sleep after 15 minutes of inactivity. The first request after a sleep period takes up to 30 seconds to respond. This is expected behaviour for development work and is acceptable for this module.

---

## Common Failures and What to Check (Slide 8)

| Failure | What to check |
|---------|--------------|
| Port binding fails on Render | Use `process.env.PORT`. A hardcoded `3000` will cause the service to fail on startup. |
| Start command mismatch | The start command in Render must match the actual entry file name. `index.js`, `app.js`, and `server.js` are different files. |
| `seed.json` FK errors | Manually verify: does every `district_id` in `stations` exist in the `districts` list? Regenerate if the chain is broken. |
| `git push` fails or repo not linked | The repo must be pushed to GitHub before linking it in Render. A local `git commit` alone is not visible to Render. |

---

## Day 1 Checkpoint (Slide 9)

All three criteria must be true before you leave:

1. **Public URL returns 200** with `{"status":"ok","session":"NB6007CEM S2"}`
2. **`seed.json` committed** to your repo with all five entities and a correct FK chain
3. **Repository shared** with the instructor as a GitHub collaborator

**Localhost exception:** Localhost is only accepted if Render is confirmed down. If that happens, document it in your `README.md` and resolve it before Session 3.

---

## Day 1 Micro-Viva Questions (Slide 10)

You will be asked one or more of the following. Prepare honest, concise answers — these are pass/fail checks, not essays.

1. Point to one entity in your model and explain why it is there and what it owns.
2. Why is `device_id` an attribute of `Vehicle` rather than a separate `Device` entity?
3. What is the difference between the data model and the resource model?
4. What URL is your API deployed at? Show me it returns 200.
5. Why does the seed data need FK consistency before we write a single endpoint?

---

## Reference: Expected `seed.json` shape

```json
{
  "provinces": [
    { "id": 1, "name": "Western" }
  ],
  "districts": [
    { "id": 1, "name": "Colombo", "province_id": 1 }
  ],
  "stations": [
    { "id": 1, "name": "Colombo Fort", "district_id": 1 }
  ],
  "vehicles": [
    { "id": 1, "registration_number": "WP-CAB-1234", "device_id": "DEV-001", "station_id": 1 }
  ],
  "pings": [
    { "id": 1, "vehicle_id": 1, "latitude": 6.9271, "longitude": 79.8612, "timestamp": "2025-01-01T08:00:00Z" }
  ]
}
```

The full file must have all five arrays at the scale specified in the prompt. This snippet shows the shape only.

---

*NB6007CEM – Web API Development | Day 1 (PM) – Session 2 | Author: Niranga Dharmaratna*
