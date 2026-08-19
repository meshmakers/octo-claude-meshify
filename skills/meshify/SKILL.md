---
name: meshify
description: Hub and entry point for building applications on OctoMesh or migrating ("meshifying") an existing application onto the platform. Drives the phased workflow — inventory the app, model the domain as a Construction Kit, map the API onto HTTP pipelines or the GraphQL data layer, adapt the wire contract, package as a blueprint, install on a tenant — and routes detail work to the sibling skills ck-modeling, pipeline-api, mesh-data-layer, and blueprint-packaging. Also owns the lookup map of docs.meshmakers.cloud (including the pipeline-node URL protocol) and the tenant/CLI prerequisites. Trigger on - meshify, build an app on OctoMesh, OctoMesh application, migrate to OctoMesh, move my backend to OctoMesh, port my app to the mesh, data mesh app, which OctoMesh skill, where is the OctoMesh documentation, docs.meshmakers.cloud, OctoMesh docs lookup, tenant prerequisites, EnableCommunication, octo-cli context setup, getting started with OctoMesh app development.
---

# Meshify — Build or Migrate an Application onto OctoMesh

## Purpose and scope

Guide the end-to-end work of putting an application on OctoMesh: a typed data model (Construction Kit), a backend (declarative pipelines and/or the generated GraphQL API), and an install story (blueprint + operator-deployed web app).

In scope: new OctoMesh-powered apps, and migrating existing apps of any backend technology. Out of scope: installing/operating the OctoMesh platform itself (assume a running environment and a tenant), and custom adapter development in C# (point to `https://docs.meshmakers.cloud/docs/developerGuide/solutionArchitecture/adapterDevelopment` if it comes up).

Assumed toolchain: `octo-cli`, `octo-ckc`, `octo-bpm`, `docker`, `helm`, plus access to an OctoMesh environment. Verify with `references/tenant-prerequisites.md` before starting hands-on work.

## Choose the target architecture first

| | A — Full pipeline backend | B — Mesh as data layer |
|---|---|---|
| Backend | Replaced by declarative HTTP pipelines on the tenant's Mesh Adapter (no service code) | Kept; reads/writes OctoMesh via the per-tenant GraphQL API |
| App shipping | SPA + thin proxy, operator-deployed Helm release, installed via blueprint | Existing hosting stays; blueprint optional (CK model + seeds only) |
| Best when | CRUD-ish API, moderate latency needs, install-per-tenant product story | Complex/transactional backend logic, incremental migration |

Architecture B is a valid first step toward A. Full selection criteria, constraints (no atomic multi-entity transactions in pipelines, no built-in auth on adapter HTTP routes), and the phased details: `references/migration-playbook.md`.

## The workflow

For a migration, run the phases of `references/migration-playbook.md` in order; for a green-field app, skip the inventory and start at domain modeling:

1. **Inventory** the existing app: entities, API surface (method + route + payload), auth, hosting.
2. **Model the domain** as a CK model → route to skill `ck-modeling`.
3. **Map the API**: architecture A — each endpoint becomes an HTTP pipeline → skill `pipeline-api`; architecture B — map data access onto GraphQL operations → skill `mesh-data-layer`.
4. **Adapt the client**: isolate the wire contract (enum name-write/int-read, date normalization, null semantics) in one API module.
5. **Package**: blueprint (CK dependencies + seed data + Application workload) and Helm chart → skill `blueprint-packaging`.
6. **Install and verify** on a tenant: `InstallBlueprint` → `DeployDataFlow` → `DeployWorkload` → smoke-test.

Keep the hub role while routing: after a sibling skill finishes its phase, return here to drive the next phase.

## Documentation lookup (never document nodes from memory)

`https://docs.meshmakers.cloud` is the single documentation authority. `references/docs-site-map.md` maps every section an app developer needs (CK guide, dataFlows, octo-cli command reference, blueprints, dataAccess/GraphQL, identity) plus the URL construction rules and their verified edge cases.

Pipeline node docs follow a fixed pattern — five categories (`trigger`, `extract`, `transformation`, `load`, `control`):

```
https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/<category>/<camelCaseNodeName>_<version>
```

The node set evolves with the platform. Before configuring an unfamiliar node, WebFetch its doc page; discover the current node list via the nodes overview page (`.../dataFlows/nodes/intro`) rather than any baked-in catalog.

## Routing table

| Task smells like | Skill |
|---|---|
| Domain modeling, CK YAML, model versioning, migrations, octo-ckc | `ck-modeling` |
| Pipeline YAML, HTTP endpoints, nodes, DataContext, deploy/debug loop | `pipeline-api` |
| GraphQL queries/mutations from an existing backend, API clients, tokens | `mesh-data-layer` |
| blueprint.yaml, seed data, octo-bpm, Helm chart, Application workload, install | `blueprint-packaging` |

## Additional resources

### Reference files

- **`references/migration-playbook.md`** — the phased methodology with selection criteria, per-phase routing, a worked localStorage-app example, and the architecture-A design constraints to disclose up front
- **`references/tenant-prerequisites.md`** — one-time machine setup (octo-cli contexts, login), tenant communication readiness (EnableCommunication seeds Pool `670…001`, Mesh Adapter `670…002`, Helm repo `670…003`; DeployPool; adapter Online), verification commands, and a platform glossary
- **`references/docs-site-map.md`** — the docs.meshmakers.cloud lookup map: URL rules, section table, node lookup protocol, live-verified URLs
