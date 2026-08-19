# Blueprint Authoring — Manifest and Seed Data

A blueprint is the install unit of an OctoMesh application: a versioned, declarative
bundle of Construction Kit (CK) model dependencies, runtime seed data, and optional
migration scripts. One `InstallBlueprint` provisions any tenant with the complete
application — data model, dataflow pipelines, domain seed entities, and web-app
workloads.

Authoritative concept documentation:
<https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/blueprints>

## Directory Layout

```
MyApp/1.0.0/
├── blueprint.yaml            # manifest
├── seed-data/
│   └── entities.yaml         # one runtime-model file with ALL seed entities
└── migrations/               # optional, for non-additive version updates
    └── from-1.0.0.yaml
```

Keep the blueprint as the single source of truth for the application. Do not maintain
parallel standalone `ImportRt` files for the same entities — they drift.

## The Manifest (`blueprint.yaml`)

```yaml
$schema: https://schemas.meshmakers.cloud/blueprint-meta.schema.json
blueprintId: MyApp-1.0.0                  # Name-Major.Minor.Patch
description: |
  What the blueprint installs, its prerequisites, and the post-install
  rollout commands. Shown on the catalog card in Refinery Studio.

ckModelDependencies:
  - MyApp-[1.0.0,2.0.0)                   # auto-imported from CK catalogs at install

# blueprintDependencies:                  # other blueprints, resolved transitively
#   - BaseEntities-[1.0,)

seedDataPath: seed-data/entities.yaml

# migrations:                             # optional, for non-additive updates
#   - fromVersion: "1.0.0"
#     scriptPath: "migrations/from-1.0.0.yaml"

# requires:                               # optional environment gate (root blueprint only)
#   octo.environment:
#     - dev
#     - test
```

| Key | Meaning |
|---|---|
| `blueprintId` | Unique id with version. The folder name in the catalog is the name part. |
| `description` | Free text; write the install story and prerequisites here. |
| `ckModelDependencies` | CK models with version ranges, auto-imported from the configured CK catalogs on apply. Consumers never run a manual `ImportCk`. |
| `blueprintDependencies` | Other blueprints with version ranges; resolved transitively and applied in topological order first. |
| `seedDataPath` | Path to the seed file, relative to the blueprint root. |
| `migrations` | Migration scripts keyed by source version (see the docs page above). |
| `requires` | Gate on blueprint variables such as `octo.environment`. A mismatch makes the install a **successful no-op** (`WasSkipped: true`) — no error is raised. |

Version ranges use the CK range syntax: `[1.0,)` means ≥ 1.0; `[1.0.0,2.0.0)` means
≥ 1.0.0 and < 2.0.0; `[1.5.0]` means exactly 1.5.0. For `ckModelDependencies` the
engine installs the range's **lower bound** on a fresh tenant, so keep the floor at
the first model version the seed actually works with.

Do not name a blueprint with the `System.` prefix — that prefix marks service-managed
blueprints that are applied automatically and hidden in Studio.

## Seed Data (`seed-data/entities.yaml`)

The seed is a runtime-model YAML file: one `dependencies:` list and one single
`entities:` sequence. Keep every entity list item at the same indentation — it is one
sequence, not one per section.

```yaml
$schema: https://schemas.meshmakers.cloud/runtime-model.schema.json
dependencies:
- System.Communication-[3.1,4.0)          # needed when seeding DataFlow/Pipeline/Application
- MyApp-[1.0.0,2.0.0)                     # your own model, for domain seed entities
entities:
  - rtId: '0abc00000000000000000001'
    ckTypeId: System.Communication/DataFlow-1
    attributes:
      - id: System/Name-1
        value: "My App API"
  # ... pipelines, applications, domain entities — all in this one sequence
```

Seed data is applied with an **upsert** strategy keyed on `rtId`: existing entities
are updated, new ones inserted. See
[examples/seed-entities-skeleton.yaml](../examples/seed-entities-skeleton.yaml) for a
complete commented skeleton.

### ID Formats — Three Different Contexts

| Context | Format | Example |
|---|---|---|
| Seed YAML (`ckTypeId`, attribute `id`, `roleId`) | versioned `Model/Element-Version` | `MyApp/Ticket-1`, `System/Name-1` |
| Pipeline YAML embedded in the seed (`ckTypeId` in nodes) | unversioned `Model/Type` | `MyApp/Ticket` |
| GraphQL delete mutations | unversioned `Model/Type` | `MyApp/Ticket` |

### Choose Stable rtIds

Hand-assign every seed entity a fixed 24-character hex `rtId` with a recognizable
prefix (e.g. `0abc…01`, `0abc…02`). This makes re-applies idempotent (upsert by rtId),
entities easy to identify on any tenant, and cross-references inside the seed stable.
Never let rtIds be generated for seed entities.

### Associations — Wiring the Application Together

Associations are declared on the child entity:

```yaml
associations:
  - roleId: System/ParentChild-1
    targetRtId: '0abc00000000000000000001'          # the DataFlow above
    targetCkTypeId: System.Communication/DataFlow-1
  - roleId: System.Communication/Executes-1
    targetRtId: '670000000000000000000002'           # Mesh Adapter (platform-seeded)
    targetCkTypeId: System.Communication/Adapter-1
```

Standard wiring for an app blueprint:

| Entity | Association | Target |
|---|---|---|
| Pipeline | `System/ParentChild-1` | your DataFlow |
| Pipeline | `System.Communication/Executes-1` | the adapter that executes it |
| Application | `System.Communication/Manages-1` | the deployment Pool |
| Application | `System.Communication/HelmRepository-1` | a HelmRepositoryConfiguration |

On every communication-enabled tenant the service-managed `System.Communication`
blueprint seeds fixed, well-known rtIds your seed can associate to directly:

| Entity | rtId |
|---|---|
| Pool | `670000000000000000000001` |
| Mesh Adapter | `670000000000000000000002` |
| Helm repository configuration (dev channel) | `670000000000000000000003` |

Which HelmRepositoryConfiguration entities exist beyond the Pool and Mesh Adapter
depends on the environment. Inspect the tenant (Studio → entities of type
`HelmRepositoryConfiguration`) before hard-wiring an rtId — or seed your own
repository configuration pointing at the Helm repository that hosts your chart
(attributes `RepositoryUrl`, `Channel`; optional `Username`/`Password` for private
repositories). Reference:
<https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/libraries/System/Communication-3/Types>

### Embedded Pipeline YAML

Pipeline definitions live **inline** in the seed as block scalars in the
`System.Communication/PipelineDefinition-1` attribute — no separate files:

```yaml
- id: System.Communication/PipelineDefinition-1
  value: |
    triggers:
      - type: FromHttpRequest@1
        method: Get
        path: /tickets
    transformations:
      - type: ToPipelineDataEvent@1
        targetPipelineRtId: 0abc00000000000000000003   # another seed pipeline!
        awaitResult: true
        ...
```

The embedded YAML can cross-reference other seed pipelines by rtId (e.g.
`targetPipelineRtId` on `ToPipelineDataEvent@1`). These references are byte-for-byte
strings inside the block scalar — **if rtIds are ever regenerated, update BOTH the
outer entity `rtId` values AND every embedded reference** (`sed` over the seed file
works cleanly). A missed embedded reference fails only at runtime, not at validation.

Pipeline entities also carry `System/Enabled-1: true` and
`System.Communication/DeploymentState-1: 0` (Undeployed — deployment is a separate
post-install step, see [install-and-update.md](install-and-update.md)).

### Ownership, Locking, and User-Mutable Seeds

The engine stamps every seed entity on apply: `rtBlueprintSource` (owning blueprint),
`rtBlueprintLocked` (true = blueprint-managed), `rtBlueprintAppliedAt`. You do not
write these — with one exception:

Seed entities that users are meant to edit after install (settings singletons,
default categories, sample records) must ship **unlocked**:

```yaml
attributes:
  - id: System/RtBlueprintLocked-1
    value: false
```

Blueprint updates (Merge/Full modes) update locked entities and **skip** unlocked
ones (raising a resolvable conflict) — so seeded-but-user-mutable data survives
updates instead of being reset. Two caveats: the update that first applies the
unlock still rewrites the entity once, and a force re-apply
(`InstallBlueprint -f`) is a full re-seed that ignores locks — the deliberate
factory reset.

### Runtime-State Attributes

Attributes flagged `isRuntimeState: true` in their CK model are owned by the running
system: on upsert the tenant's current value is preserved and the seed value only
applies on first insert. `System.Communication` flags `Hostname`, `IngressEnabled`,
`ChartVersion`, `ValuesYaml`, and `Values` on workloads this way, so operator-set
deployment values survive re-applies. Apply the same flag to your own models'
operational attributes (status enums, counters, watermarks) — never ship mutable
operational state in seed data without it.

### Variables

| Syntax | Resolved | Examples |
|---|---|---|
| `${name}` | at blueprint apply | `${octo.tenantId}`, `${octo.environment}` |
| `{{domain.NAME}}` | at workload deploy, by the communication controller | `{{domain.default}}` |

The standard `octo.*` variable set (verified against the platform source, 2026-08-19):
`octo.tenantId`, `octo.systemTenantId`, `octo.isSystemTenant` (`"true"`/`"false"`),
`octo.environment` (one of `dev` / `test` / `staging` / `production`),
`octo.environmentMode` (the matching `System/EnvironmentModes` enum name —
Development/Testing/Staging/Production; an unknown environment falls back to
Development with a logged warning), `octo.version` (the platform version, normalized
to 3-segment SemVer), and `octo.scheme` + `octo.domain` for composing per-cluster
public URLs (`${octo.scheme}://<slug>.${octo.domain}`).

`${…}` is substituted in string attribute values and `rtWellKnownName`. An unknown
`${placeholder}` logs a warning and stays verbatim in the database. `{{domain.*}}`
placeholders are left untouched at apply time and resolved per cluster at deploy —
this is what makes one seed's `Hostname` work on every cluster (see
[app-workload-and-helm.md](app-workload-and-helm.md)).

## Validate Before Publishing

```bash
octo-bpm -c validate -p ./MyApp/1.0.0
```

Validation checks the manifest against the schema and reports the parsed blueprint
id, CK dependencies, and seed path. It does **not** validate the pipeline YAML inside
`PipelineDefinition-1` block scalars — test pipelines against a live tenant before
freezing them into the seed.

## Related References

- [install-and-update.md](install-and-update.md) — publish, install, rollout, update, uninstall
- [app-workload-and-helm.md](app-workload-and-helm.md) — the Application entity and chart contract
- [examples/blueprint.yaml](../examples/blueprint.yaml) — complete minimal manifest
- [examples/seed-entities-skeleton.yaml](../examples/seed-entities-skeleton.yaml) — commented seed skeleton
