# NB6007CEM - Web API Development
## Session 3 Student Study Notes: Collections and URI Design
**Day 2 (AM) - Session 3**

---

## Session Overview

By the end of this session you will be able to:

- Distinguish atomic from collection resources using the §4.1 and §4.2 derivation rules
- Apply the §4.6 scoping test to decide when a collection is first-class versus nested under a parent
- Name resources following all five §5.1 naming rules simultaneously
- Produce and evaluate the complete resource map for the tuk-tuk system

**Design guidelines covered:** §4.1, §4.2, §4.6, §5.1, §5.6

---

## Where We Are: The REST API Design Pipeline

The design guidelines define a seven-step pipeline for building a RESTful API:

1. Create Data Model
2. **Derive Resources** ← **TODAY**
3. Decide on Representations
4. **Name Resources by URLs** ← **TODAY**
5. Determine HTTP Methods
6. Determine Special Behaviour
7. Consider Errors

The data model was settled in S1. Representations are chosen in S5. HTTP Methods in S7. This session covers steps 2 and 4 - the two steps that give the API its shape.

---

## Section A: From Data Model to Resources

### From Data Model to Resource Model

A data model and a resource model are not the same thing, and understanding the difference is the first thing to get right.

The **data model** specifies entity properties in an abstract, implementation-independent form. No format decisions are made at this stage - the data model deliberately keeps those choices open. This is why, in S1, you produced an entity-relationship diagram without yet deciding whether responses would be JSON or XML. That decision belongs to a later step.

The **resource model** specifies what clients interact with. It is shaped by client needs first, and by the ER diagram second. The guidelines call this the "clients win over data" principle: the ER diagram drives your implementation, but your resource model is driven by client interactions.

Two important consequences follow from this:

- New resources may be derived that have no corresponding entity type in the ER diagram. Collection resources are the main example: there is no "Provinces" entity in the data model, but the API needs a `/provinces` collection because police need to browse all provinces.
- Some ER entity types may not become independent top-level resources. LocationPing is the tuk-tuk example: it exists in the data model, but it does not become a standalone resource because no client ever needs to access it independently of its vehicle.

### The Five Resource Types (§4)

The guidelines define five kinds of resources that may be derived from a data model. Not all five are warranted in every API - you derive only those that client needs justify.

| Type | What it represents | §ref |
|---|---|---|
| Atomic | An entity exchanged as a complete whole | §4.1 |
| Collection | A grouped set of atomics of the same type | §4.2 |
| Composite | An aggregate of multiple entities manipulated as a whole | §4.3 |
| Processing Function | A computation, function, or partial update | §4.5 |
| Controller | A multi-resource atomic operation for data consistency | §4.4 |

This session covers types 1 and 2. Types 3, 4, and 5 are introduced in S5.

---

## Principle 1: Atomic Resources (§4.1)

### The Formal Definition

The guidelines state: "The most basic decision to be made for deriving resources is identifying entities of the data model that are exchanged as a whole via the API. Such entities become atomic resources."

An entity qualifies as an atomic resource when all three of the following conditions hold:

1. **Accessed in multiple client scenarios.** The entity's complete set of details is needed broadly - not just as a side-effect of accessing another resource.
2. **Does NOT require traversal of relationships to other entities.** You can retrieve, create, or delete this entity without first navigating through a relationship to something else.
3. **Exchanged as a complete unit.** The API delivers the entity whole. It is never served as a partial fragment of a larger resource.

When an entity passes all three, it maps directly to one atomic resource - a 1-to-1 derivation.

### Analogy: The NIC Card

A National Identity Card (NIC) is exchanged as a complete unit. When you request someone's NIC record, you receive the whole document - name, birthday, address, NIC number - not a fragment of it. Province, District, PoliceStation, and Vehicle in the tuk-tuk system work the same way: each is retrieved, created, or updated as a whole record, never as a partial fragment.

*Limit:* A real NIC is immutable once issued; atomic resources can be updated with PUT.

### Generator Mistake: The Standalone LocationPing Trap

A code generator, given the entity-relationship model, will often produce a standalone route for every entity it finds:

```
GET /pings/{ping-id}
```

Apply the three conditions to LocationPing:

- **Multiple client scenarios?** No. Police never request "ping 4827 in isolation". Every ping request names a specific vehicle first.
- **Independent of relationships?** No. A ping has no meaning without its vehicle. Traversal through the Vehicle entity is always required to establish context.
- **Exchanged as a whole?** No. No client ever needs an isolated ping record.

LocationPing fails all three conditions. It is not an atomic resource. The guidelines formalise this in §4.6: "if manipulating instances of a certain entity type requires the traversal of its associated relationships, the entity type is not an atomic candidate."

LocationPing belongs to a scoped collection under Vehicle. This is addressed in Section B.

**Coursework connection:** The §4.1 independence test applied to your domain entities is part of the Architecture and Data Model dimension. Your report should explain which entities became atomics and why - with explicit reference to the three conditions.

### Atomic Resources in the Tuk-Tuk System

Applying the three conditions to each entity in the domain model:

- **Province** - retrieved independently (name, code, HQ). No traversal required. Passes.
- **District** - retrieved independently (name, province reference, district seat). The province reference is a property, not a traversal requirement. Passes.
- **PoliceStation** - retrieved independently (name, district, OIC contact). Passes.
- **Vehicle** - retrieved independently (plate, driver, device_id, active status). Passes.
- **User** - retrieved independently (name, role, jurisdiction scope). Passes.

One note on User: the User collection endpoint (`/users`) is deferred to S7. The auth layer determines who may create users and what scope they carry. The atomic resource exists; the collection factory that creates them arrives with authentication.

---

## Principle 2: Collection Resources (§4.2)

### The Formal Definition

The guidelines state: "The next decision to be made is whether atomic resources of the same type are needed to be grouped into a set. Such bundles become collection resources."

Two important facts about collection resources:

1. **The collection type does NOT appear in the ER diagram.** There is no "Provinces" entity. Collections are derived from client need, not found in the data model.
2. **A collection resource is a factory for its members** (§7.3). When you POST to a collection, the API creates a new atomic member.

### The Two Derivation Triggers

Either trigger alone is enough to warrant a collection:

**Catalogue trigger:** Clients need to browse, search, or filter instances of that entity type. If police need to see a list of all vehicles, `/vehicles` exists.

**Creation trigger:** The API supports creating new instances of that entity type. If the API allows registering a new vehicle, `/vehicles` must exist as the entry point for creation - because "a collection resource is a factory for its members."

### Analogy: The Motor Traffic Department Vehicle Register

The Motor Traffic Department maintains a vehicle register - a browsable ledger of all registered vehicles. The register is the collection resource; each vehicle certificate is an atomic member. You go to the register to browse, search, and add records. The register exists independently of any individual certificate.

*Limit:* The real DMT register is owned by one authority per district. In the API, a collection resource is a single canonical set accessible to all permitted clients.

### Generator Mistakes: Two Collection Traps

**Trap 1 - Wrong singular name.** A generator frequently produces `/province`, `/vehicle`, `/station` (singular nouns) for what should be collection resources. The §5.1 rule is explicit: "Names of collections should be 'pluralized', i.e. named by the plural noun of the grouped concept." The singular is ambiguous: does `/province` mean the collection or one specific province? The correct names are `/provinces`, `/vehicles`, `/stations`, `/districts`.

**Trap 2 - Phantom collection.** A generator may invent `/ping-history` as a top-level route because it sounds reasonable. But there is no "PingHistory" entity type in the data model. No catalogue trigger exists at the top level (police never browse all pings globally). No creation trigger exists either. This collection is a phantom - it does not satisfy either derivation criterion at the root level. LocationPing will be derived as a scoped collection under Vehicle (`/vehicles/{vehicle-id}/pings`), which is addressed in Section B.

**Coursework connection:** The §4.2 derivation triggers applied to your entity types are part of the API Design dimension. Your report should justify each collection you include by naming which trigger (or both) warranted it.

### Collections in the Tuk-Tuk System

| Collection | Catalogue trigger | Creation trigger |
|---|---|---|
| `/provinces` | Police browse all provinces | Register a new province |
| `/districts` | Police browse all districts | Register a new district |
| `/stations` | Police browse all stations | Register a new station |
| `/vehicles` | Police browse all vehicles | Register a new vehicle |

LocationPing is not a first-class collection. No catalogue or creation trigger exists at the root level. Its derivation as a scoped collection is addressed next.

---

## Section B: Relationships and Scoping

### Principle 3: Interpreting Relationships (§4.6)

#### The Formal Definition

The guidelines identify "one of the basic problems of deriving a resource model from a data model" as interpreting ER relationships. Two tests govern this:

**The independence test (for atomic candidates):** "If manipulating instances of a certain entity type does not require the traversal of its associated relationships, then such an entity type is a candidate for an atomic resource."

**The scoping test (for collection candidates):** "Collection resources may be subject to an interpretation of its associated relationship types, i.e. collections may only make sense as children of other resources" - these are called scoped collections.

From these two tests, two categories of collection emerge:

- **First-class (unscoped):** The complete set across all parents is meaningful to clients. The collection stands at the root level.
- **Scoped:** The collection only makes sense in the context of a specific parent instance. It is nested in the URL path under that parent.

#### Analogy: Electricity Meter Readings

When you request electricity meter readings, you request readings for meter number 4512 - not "all readings from all meters in the Western Province." The readings only exist, and only make sense, under that specific meter. `/vehicles/{vehicle-id}/pings` works the same way: the pings are always scoped to one vehicle.

*Limit:* A meter belongs to one utility; a tuk-tuk can cross district boundaries. The scoping analogy holds at the API path level, not at the physical ownership level.

---

> **The scoping decision is a client need, not a data model relationship.**
>
> The ER diagram shows that Vehicle has many LocationPings. But the API is not the ER diagram. The question is: do clients ever need all pings globally? When the answer is no, the collection is scoped - and the URL path makes that constraint permanent and visible to every caller.

---

### Scoped vs Unscoped: The Tuk-Tuk Decision

| First-class (globally meaningful) | Scoped (under a parent) |
|---|---|
| `/provinces` | `/vehicles/{vehicle-id}/pings` |
| `/districts` | |
| `/stations` | |
| `/vehicles` | |

**Evidence for first-class:** Police DO query provinces, districts, stations, and vehicles at the top level. "Show all vehicles in the system" is a legitimate police need.

**Evidence for scoped:** Police NEVER request "all pings in Sri Lanka." Every ping request names a specific vehicle first. The ER relationship (Vehicle 1-* LocationPing) becomes a scoped URI path, not a top-level collection.

### Generator Mistake: The Global /pings Trap

A code generator, seeing LocationPings in the data model, will frequently produce a top-level collection:

```
GET /pings?vehicleId=WP-CAB-1234
```

Three reasons §5.6 rejects this:

1. **Client-need test fails.** "The set of all pings globally is never a meaningful resource." The first-class criterion requires that the complete global set be of interest to clients. It is not.

2. **§5.6 requires path-level scoping when a collection is relationship-dependent.** When a collection only makes sense under a specific parent instance, the parent must appear in the path - not as a query parameter.

3. **`?vehicleId=` is a §5.7 query-string parameter for filtering an existing collection.** You cannot use a query string to scope a collection that does not exist at the root. Filtering and scoping are different concerns entirely.

The correct design is `/vehicles/{vehicle-id}/pings`. The vehicle-id in the path makes the parent relationship explicit and permanent for every caller.

Note: Query strings for pagination and filtering are covered in S9 (§10.2-10.4). They address different problems and do not substitute for path-level scoping.

**Coursework connection:** The §4.6 scoping decision for LocationPing is one of the most important design choices in the tuk-tuk system. Your report should defend it explicitly, citing the client-need test and the §5.6 path-scoping requirement.

---

## Section C: Naming the Resources

### Principle 4: URI Naming Rules (§5.1)

The guidelines state: "Proper naming of resources is key for an API to be easily understandable by clients."

Five rules must all hold simultaneously. Violating any one is sufficient to make a URI wrong.

#### Rule 1 - Nouns, not verbs

Atomic, collection, and composite resources are "things" - they are named as nouns. Processing function and controller resource types are "actions" - those use verbs (covered in S5). In S3, every resource we have derived is a noun.

The reasoning: HTTP already has verbs (GET, POST, PUT, DELETE). The URI names the thing being acted on. Putting a verb in the URI conflates the what with the how.

*Generator mistake:* `/getVehicleList` is a method call, not a resource name. The resource is `/vehicles`.

#### Rule 2 - Lowercase only

Case sensitivity rules in URI elements cause ambiguity between clients and servers - different systems interpret case differently. Lowercase removes all ambiguity. A client that types `/Vehicles` and one that types `/vehicles` should reach the same resource; with mixed case, they may not.

*Generator mistake:* `/Vehicle_Location` - both the uppercase V and the underscore violate this rule (and Rule 3).

#### Rule 3 - Hyphens, not underscores or camelCase

Underscores disappear under hyperlinks - when a URI is rendered as a clickable link, an underscore character can be invisible because it sits below the underline decoration. A reader cannot tell whether the separator is there or not.

camelCase is a programming language convention (Java, JavaScript) carried across into URI design where it does not belong. A URI is not a variable name.

Hyphens are the only accepted word separator in URI paths.

*Generator mistakes:* `/vehicleLocation` (camelCase), `/Vehicle_Location` (underscore).

#### Rule 4 - Plural for collections, singular for atomic members

The plural names the set; the singular (via a template variable) names one member.

- `/vehicles` - the collection of all vehicles
- `/vehicles/{vehicle-id}` - one specific vehicle

The pattern is consistent: collection is always plural, member identifier follows in curly brackets.

*Generator mistake:* `/vehicle` for a collection - singular is ambiguous between "the collection" and "one member".

#### Rule 5 - Forward slash for hierarchy

Parent resources immediately precede their children in the path. No trailing slash. The slash is not decorative - it signals hierarchy. `/vehicles/{vehicle-id}/pings` reads: "the pings that belong to this vehicle."

### Analogy: Sri Lanka Post Address Format

The Sri Lanka Post only delivers if you follow the standard format: No. 10, Galle Road, Colombo 3. Invent your own format - `10_GalleRd_CMB3` or `GETNoTenGalleRoadColomboCityThree` - and delivery fails. A URI is a postal address for a resource: one standard, enforced consistently.

*Limit:* Postal addresses are read by human sorters who can adapt to slight variations. URIs are parsed by machines that are strictly case-sensitive and structure-dependent. The post office worker adapts; the HTTP parser cannot.

### Generator Mistakes: §5.1 Violations in the Wild

Every path in this table was produced by a generator for the tuk-tuk system:

| Generator produces | §5.1 rule broken | Correct version |
|---|---|---|
| `/getVehicleList` | Verb-as-noun; camelCase (Rules 1 + 3) | `/vehicles` |
| `/Vehicle_Location` | Uppercase; underscore (Rules 2 + 3) | `/vehicle-locations` |
| `/vehicleLocation` | camelCase (Rule 3) | `/vehicle-locations` |
| `/locationPings` | camelCase + unscoped parent missing (Rules 3 + §5.6) | `/vehicles/{vehicle-id}/pings` |
| `/vehicles/id/pings` | Literal "id" is not a template variable (§5.6) | `/vehicles/{vehicle-id}/pings` |

---

> **A URI is a name, not a sentence. The API speaks in nouns, not instructions.**
>
> When a generator produces `/getVehicleLocation`, it is writing a method call, not naming a resource. The resource already exists; it needs a name clients can find and remember. `/vehicles/{vehicle-id}` is that name. The HTTP method (GET, POST, etc.) handles the verb - so the URI never needs one.

---

**Coursework connection:** Every path in your submitted API will be evaluated against all five §5.1 rules. This is part of the API Design dimension. You should be able to name the specific rule broken by any path a generator gives you, and explain in your own words why that rule exists.

### Principle 5: URI Templates and Scoped Collections (§5.6)

#### The Formal Definition

The guidelines define a URI template as: "an element which contains strings in curly brackets. These strings are variables that must be substituted by values when such a URI template is used by a client." (See also RFC 6570.)

The pattern for identifying one atomic member within a collection:

```
/{collection-name}/{member-id}
```

The member-id variable immediately follows the collection name. It must be substituted with a real identifier on every API call. A template without substitution is meaningless.

When a collection is scoped under a parent, the parent member's template is extended:

```
/{collection}/{member-id}/{scoped-collection}
```

Applied to the tuk-tuk system:

```
/vehicles/{vehicle-id}/pings
```

This reads: "the pings that belong to the vehicle identified by vehicle-id."

#### Template vs Literal

The difference matters:

| Template | Literal (wrong) |
|---|---|
| `/vehicles/{vehicle-id}` | `/vehicles/vehicle-id` |

`{vehicle-id}` is a variable that is substituted per call: `/vehicles/WP-CAB-1234`, `/vehicles/SP-TUK-0021`. The curly brackets signal "fill this in."

`vehicle-id` without curly brackets is a literal string that the router tries to match exactly. No route in your system will have a registered vehicle with the identifier `vehicle-id`. Every call will return 404.

#### Variable Naming Convention

Name variables using the entity-type name followed by `-id`, in lowercase-hyphen form: `vehicle-id`, `province-id`, `station-id`. This makes it immediately clear which entity the variable identifies.

#### The Depth Rule

Only nest as deep as the client relationship requires. `/vehicles/{vehicle-id}/pings` is justified because pings are always scoped to one vehicle. There is no requirement to nest further (for example, `/vehicles/{vehicle-id}/pings/{ping-id}` is not needed - no client ever needs to address an individual ping by its own identifier).

### Analogy: The Police Stop-and-Check Form

A police stop-and-check form has a printed blank: "Vehicle Reg. No.: ___." You fill in WP CAB 1234 to mean exactly that vehicle. `/vehicles/{vehicle-id}` is the same blank - a parameterised slot that resolves to one specific resource. Leave it empty, and the form simultaneously describes every vehicle and no vehicle.

*Limit:* A police form blank is filled in once and submitted. A URI template variable is substituted fresh on every API call - the slot resets each time.

---

## The Tuk-Tuk Resource Map - Session 3

Everything below was derived in this session. HTTP methods, JSON representations, and authentication are not included - those belong to later sessions.

### Unscoped Collections and Atomic Members

```
/provinces                      collection (catalogue + creation)
/provinces/{province-id}        atomic member

/districts                      collection (catalogue + creation)
/districts/{district-id}        atomic member

/stations                       collection (catalogue + creation)
/stations/{station-id}          atomic member

/vehicles                       collection (catalogue + creation)
/vehicles/{vehicle-id}          atomic member
```

### Scoped Collection

```
/vehicles/{vehicle-id}/pings    scoped collection (LocationPings under one vehicle)
```

### Not Yet Derived

| Capability | Session |
|---|---|
| HTTP methods (GET, POST, PUT, DELETE) | S7 |
| JSON representation structure | S5 |
| Composite resources, last-known location | S5 |
| /users collection | S7 |

---

## Activity: URI Naming Linter

Open the URI Linter demo on your device. Try the following paths and read the violations the linter flags:

- `/getVehicleLocation`
- `/Vehicle_Location`
- `/vehicleLocation`

For each violation: **name the §5.1 rule it breaks in your own words**, not just the linter label. Then type a path from your own domain model and repeat the exercise.

Find one path that passes clean (no violations flagged).

**What to try next:** Type variations of the same resource name - for example, `/VehicleLocation`, `/vehicle_location`, and `/vehicle-location` - and compare the results. Only one of these three passes.

**Pairing note:** Work in pairs. Both members must be able to explain each violation by rule name at the S4 micro-viva.

---

## Discussion Question

User is listed as an atomic resource - but `/users` is not on the resource map. Which §4 rule explains the gap? Which session introduces the `/users` collection, and why does the authentication layer determine the answer?

Think through your answer before S4. You may be asked at the micro-viva.

*Hint: consider the creation trigger and ask who has authority to create users in a police monitoring system.*

---

## Key Terms

**Atomic resource (§4.1):** A resource derived from one entity type of the data model. Exchanged as a complete unit. Identified by the three conditions: multiple scenarios, independence of relationships, complete unit.

**Collection resource (§4.2):** A resource grouping atomics of the same type. Not found in the ER diagram - derived from client need. Warranted by catalogue or creation trigger (or both). A factory for its members (§7.3).

**Composite resource (§4.3):** A resource representing an aggregate of multiple entities manipulated as a whole. Not covered in S3 - introduced in S5.

**Derivation trigger:** A client need that justifies the existence of a collection. Two triggers exist: catalogue (browsing or filtering instances) and creation (creating new instances).

**Factory principle (§7.3):** "A collection resource is a factory for its members." POST to the collection creates a new atomic member.

**First-class collection:** A collection whose global set is meaningful to clients. Stands at the root level of the API.

**Scoped collection:** A collection that only makes sense under a specific parent instance. Nested in the URL path under the parent's member template.

**URI template (§5.6, RFC 6570):** A URI pattern containing curly-bracket variables that must be substituted by values on each API call. Pattern: `/{collection-name}/{member-id}`.

**Independence test (§4.6):** The test for whether an entity qualifies as an atomic resource. If manipulating it requires traversal of relationships to other entities, it does not qualify.

**Scoping test (§4.6):** The test for whether a collection must be scoped under a parent. If the collection only makes sense in the context of a specific parent instance, it is a scoped collection.

**WSO2 §5.1 naming rules:** The five simultaneous constraints on resource names: nouns not verbs; lowercase only; hyphens as word separators; plural for collections, singular for members; forward slash for hierarchy.
