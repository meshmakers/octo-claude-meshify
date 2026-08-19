# octo-claude-meshify

A Claude Code plugin for **building applications on [OctoMesh](https://docs.meshmakers.cloud)** — and for migrating ("meshifying") an existing application onto the platform.

It is aimed at application developers **outside** the OctoMesh core team: you have a running OctoMesh environment and want your app's data model, API, and deployment story to live on the mesh. The plugin does not cover setting up OctoMesh itself.

## What the skills know

| Skill | What it covers |
|---|---|
| `meshify` | The hub: migration playbook (inventory your app → model the domain → map the API → package → install), the two target architectures, tenant prerequisites, and the map of docs.meshmakers.cloud |
| `ck-modeling` | Construction Kit data modeling: CK YAML authoring (types, enums, associations), versioning rules and migrations, catalogs and publishing |
| `pipeline-api` | Declarative HTTP APIs built from ETL pipelines on the Mesh Adapter: triggers, DataContext/JSONPath, entity CRUD, deployment and debugging — with live lookup of pipeline-node docs instead of baked-in node details |
| `blueprint-packaging` | Packaging the app as an installable blueprint (CK dependencies + seed data + workloads), the web-app Helm chart contract, and operator-deployed Application workloads |
| `mesh-data-layer` | Keeping your own backend and using OctoMesh as the data layer: the per-tenant GraphQL API, machine-to-machine auth, and client wire conventions |

Two target architectures are supported:

1. **Full pipeline backend** — the app's backend is replaced by declarative HTTP pipelines executed by the tenant's Mesh Adapter; the frontend ships as a SPA + thin proxy, deployed by the Communication Operator as a Helm release. No service code.
2. **Mesh as data layer** — the app keeps its backend and reads/writes OctoMesh through the generated per-tenant GraphQL API. A valid first migration step toward (1).

## Prerequisites

- A running OctoMesh environment (managed or your own cluster) and a tenant you can work on
- The OctoMesh CLI toolchain: `octo-cli`, `octo-ckc`, `octo-bpm`
- `docker` and `helm` for the app image and chart
- Documentation reference: <https://docs.meshmakers.cloud>

## Installation

```bash
# Add the marketplace, then install the plugin
/plugin marketplace add meshmakers/octo-claude-meshify
/plugin install octo-claude-meshify@octo-claude-meshify
```

Or from a local checkout:

```bash
claude --plugin-dir /path/to/octo-claude-meshify
```

## Quick start

Ask Claude, for example:

- "I have a React app with an Express backend — help me meshify it."
- "Model my inventory domain as a Construction Kit."
- "Build an HTTP endpoint that returns all open orders as JSON."
- "Package my app as a blueprint so a tenant can install it."
- "How does my existing backend authenticate against the OctoMesh GraphQL API?"

The `meshify` hub skill routes to the right sibling skill and keeps the overall migration workflow on track.
