# Migration Playbook: Meshifying an Existing Application

Turn an existing application — a localStorage-only SPA, a small CRUD service, a
script glued to a spreadsheet — into an application backed by OctoMesh. This
playbook is the phased methodology; detail work is delegated to sibling
skills at each step.

Read [tenant-prerequisites.md](tenant-prerequisites.md) first if the target
tenant has not been prepared yet — every phase below assumes an OctoMesh
environment and an authenticated `octo-cli` context already exist.

## Phase 0 — Qualify: what OctoMesh gives you, and which shape to build

Before touching code, decide what part of OctoMesh the app should sit on top
of, and which of two target architectures fits.

### What OctoMesh gives an application

- **A typed data mesh.** The domain model is authored once as a Construction
  Kit (CK) — versioned entity types, attributes, enums and associations —
  instead of hand-rolled database tables and DTOs.
- **A generated GraphQL API.** Importing a CK model auto-generates a
  per-tenant GraphQL schema with create/read/update/delete operations for
  every type (naming pattern `[TypeName]Input` / `[TypeName]InputUpdate`),
  served at `https://<assetRepoHost>/tenants/<tenantId>/graphql` with a
  playground at `.../graphql/playground`
  ([Data Access overview](https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/intro),
  [Runtime Model API reference](https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/runtime)).
  No separate API story is needed for a new CK type — importing it is
  enough.
- **A declarative pipeline backend.** Business logic and HTTP endpoints can
  be authored as pipeline YAML executed by a tenant's Mesh Adapter — no
  service code, no deployment of a custom API server.
- **A blueprint install story.** A blueprint packages a CK model dependency,
  seed runtime data (pipelines, domain records) and an optional app workload
  into one unit; a single `InstallBlueprint` provisions all of it on any
  tenant.
- **Operator-managed app hosting.** A containerized web app can ship as a
  `System.Communication/Application` entity; the Communication Operator
  rolls it out as a Helm release on the target Kubernetes cluster, the same
  mechanism used for Adapters.

### Two target architectures

**Architecture A — Full pipeline backend.** The existing backend is
retired. Every API endpoint becomes an HTTP pipeline
(`FromHttpRequest@1`) on the tenant's Mesh Adapter. The frontend ships as a
static SPA plus a thin Node (or similar) proxy that forwards `/api/*` calls
to the adapter — the proxy exists only because the adapter serves JSON, not
HTML, so something has to serve the SPA's static files. There is no other
service code.

**Architecture B — Mesh as data layer.** The existing backend stays. Only
its data access changes: instead of a hand-rolled database, the backend (or
the frontend directly, if it has no backend of its own) reads and writes
through the generated GraphQL API. All existing business logic,
authorization and orchestration code keeps running exactly where it runs
today.

### Selecting an architecture

| Criterion | Favors A (full pipeline backend) | Favors B (mesh as data layer) |
|---|---|---|
| API complexity | Endpoints are close to CRUD, plus a few rules expressible as pipeline nodes (lookups, computed fields, simple server-side aggregation) | Endpoints encode substantial custom logic, multi-system orchestration, or algorithms better expressed in a general-purpose language |
| Latency / control | The adapter's own request/response cycle is acceptable | Fine-grained control over caching, batching, streaming, or long-running work is required |
| Transactionality | Operations are single-entity, or the app can tolerate a check-then-update race — pipelines have **no atomic multi-entity transaction primitive** | Operations must commit atomically across multiple entities in one request |
| Auth | Acceptable behind a thin proxy or gateway that adds auth in front of the pipelines — adapter HTTP routes carry **no built-in authentication** | Request-level authorization the existing backend already implements needs to keep working unchanged |
| Appetite for rewrite | Ready to re-express the API surface as pipeline YAML and stop maintaining a custom backend | Wants to keep the current backend and business logic largely as-is |

**B is a valid first step toward A.** A team not ready to retire its backend
can start by pointing its existing data-access layer at the generated
GraphQL API instead of a custom database, then migrate individual endpoints
to pipelines over time as confidence grows. The two architectures are not
mutually exclusive at any point during a migration — an app can run parts of
its API as pipelines and parts through its own backend against the mesh
indefinitely.

## Phase 1 — Inventory the existing app

Before modeling anything, write down what exists today. This inventory
drives every later phase.

- **Entities / data model.** List every distinct "thing" the app persists
  (tasks, orders, projects, …) and, for each, its fields and their types.
- **API surface.** One row per endpoint:

  | Method | Route | Request payload | Response | Notes |
  |---|---|---|---|---|

- **Auth model.** How are callers authenticated today (session cookie, JWT,
  none)? Is authorization per-endpoint, per-record, or absent?
- **Hosting.** How is the app deployed today (container, static host, PaaS)?
  What environment variables or secrets does it need at runtime?

## Phase 2 — Model the domain as a CK model

Turn the entity inventory into CK types, attributes, enums and associations.
A small app maps to a small model — a family organizer's entire domain
fit in one CK model with six types and four enums (see the worked example
below). Route detailed authoring work to the domain-modeling skill:

```
Skill("ck-modeling", "model <entities from the inventory> as a CK model")
```

## Phase 3 — Map the API

How each existing endpoint gets served depends on the architecture chosen in
Phase 0.

**Architecture A:** each endpoint from the Phase 1 table becomes one HTTP
pipeline on the Mesh Adapter. Route this to the pipeline-API skill, endpoint
by endpoint or as a batch:

```
Skill("pipeline-api", "build an HTTP pipeline for <method> <route> from the inventory")
```

**Architecture B:** each endpoint's data access is re-expressed as a GraphQL
query or mutation against the generated schema (`runtime` field,
`[TypeName]Input` / `[TypeName]InputUpdate` types), leaving the endpoint's
business logic and auth check where they are today. Route this to the
mesh-data-layer skill:

```
Skill("mesh-data-layer", "map <route>'s data access onto GraphQL operations")
```

## Phase 4 — Adapt the frontend/backend: the wire-contract layer

Whichever architecture is chosen, the shape of data crossing the wire
changes slightly from what the original app expected — enum representation,
date formatting, and null/empty handling are the recurring differences.
Concentrate every such mapping in **one** API module so the rest of the
codebase keeps using the app's original types.

Concrete mapping concerns to check for, drawn from a completed
architecture-A migration:

- **Enum representation.** CK enum values may need to be **written** as
  their declared names (e.g. `"Recurring"`) and are **read** back as integer
  keys — handle both directions in the one module, not scattered through
  call sites.
- **Date/time normalization.** A string that looks like a date can come back
  reformatted by the platform's JSON handling; normalize on read rather than
  assuming byte-for-byte round-tripping.
- **Empty-collection writes.** Writing an empty array to clear a collection
  attribute can be indistinguishable from "no value" and null the attribute
  instead — if that bites, model a sentinel (a `count`/marker field) rather
  than relying on an empty array surviving the round trip.
- **Error shape (architecture A only).** `FromHttpRequest@1` pipelines
  cannot set HTTP status codes — errors typically come back as HTTP 200 with
  an `{ "error": ... }` payload. Client code must check for that field
  explicitly rather than relying on the HTTP status.

## Phase 5 — Package as a blueprint + chart

Once the CK model and the API mapping work end to end against a scratch
setup, package everything as one installable blueprint: the CK model
dependency, the seed data (pipelines for architecture A, or just domain
records for architecture B), and — if the app ships a UI — the Application
entity plus its Helm chart. Route this to the blueprint-packaging skill:

```
Skill("blueprint-packaging", "package <app> as a blueprint")
```

## Phase 6 — Install and verify on a tenant

```
octo-cli -c InstallBlueprint -b <BlueprintId>-<Version>
octo-cli -c DeployDataFlow -id <dataFlowRtId>             # architecture A: rolls out the pipelines
octo-cli -c DeployWorkload -id <applicationRtId>          # if the app ships a UI workload
```

Verify the install output reports success and the expected CK model loaded,
then exercise the app end to end: the full endpoint/query matrix for
architecture A, or a real read/write cycle through the GraphQL API for
architecture B. See [tenant-prerequisites.md](tenant-prerequisites.md) for
the `GetAdapters` / `GetPools` / `GetDataFlowStatus` commands used to confirm
the tenant side is healthy before and after install.

## Worked example: a localStorage app going full-pipeline-backend

A family organizer app — tasks with a points/sticker game, a shopping list,
pinboard notes, a weekly calendar — started as a React SPA storing
everything in the browser's `localStorage`. It was migrated to architecture
A end to end:

| Phase | Outcome |
|---|---|
| Starting point | React SPA, all state in `localStorage`, no backend at all |
| 1. Inventory | Tasks, shopping list, pinboard notes, weekly calendar, household/points — all client-only state in one state-management store |
| 2. CK model | One CK model: 6 types, 4 enums |
| 3. API mapping | 16 HTTP pipelines on the Mesh Adapter: task CRUD with server-side point booking, shopping-list operations, notes, calendar-day upserts, a household state-aggregation endpoint |
| 4. Wire contract | One API module: enum names written / integer keys read, date-string normalization on read, an empty-array write sentinel for one collection attribute, error payloads read from `{ "error": ... }` |
| 5. Packaging | One blueprint: CK model dependency + DataFlow/pipelines + domain seed data (a household singleton, default calendar categories) + the SPA as an Application workload |
| 6. Install + verify | `InstallBlueprint`, then `DeployDataFlow` + `DeployWorkload`; the app became reachable at the operator-managed ingress host for the tenant |

The state-management store's public interface did not change during the
migration — only its actions started firing an API call alongside the
optimistic local update, and a boot-time load replaced the `localStorage`
read. Existing UI components needed no changes.

## Design constraints to disclose up front (architecture A)

Set expectations with whoever owns the migration before committing to
architecture A for a given endpoint:

- Pipeline check-then-update is not atomic — concurrent requests to the same
  entity can race.
- Adapter HTTP routes carry no built-in authentication — put a proxy or
  gateway in front if the endpoint needs one.
- Responses are always `application/json` — an endpoint that needs to serve
  HTML belongs on the app workload, not the adapter.

These are the same constraints noted in Phase 0's selection criteria; they
resurface here because a per-endpoint decision (some endpoints on
architecture A, others kept on B) is common and each endpoint should be
checked against them individually.
