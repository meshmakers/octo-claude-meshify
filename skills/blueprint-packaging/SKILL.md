---
name: blueprint-packaging
description: Package an OctoMesh application as an installable blueprint - the versioned bundle of CK model dependencies plus seed data (DataFlow, pipelines embedded as YAML, Application workload, domain seed entities) - and ship the web app as an operator-deployed Helm release. Covers blueprint.yaml manifests, seed entity authoring with stable rtIds and association wiring, octo-bpm validate/publish, InstallBlueprint and update semantics, the web-app Helm chart contract, the Dockerfile/proxy image pattern, DeployWorkload and kubectl-free troubleshooting. Trigger on - OctoMesh blueprint, blueprint.yaml, seed data, seed entities.yaml, InstallBlueprint, UpdateBlueprint, UninstallBlueprint, octo-bpm, blueprint catalog, RefreshBlueprintCatalogs, Application entity, workload, DeployWorkload, UndeployWorkload, Helm chart for an OctoMesh app, chart contract, ingress hostname, publicUri, app image Dockerfile, package my app as a blueprint, install the app on a tenant, RtBlueprintLocked.
---

# Blueprint Packaging — the Install Unit of an OctoMesh App

## Why blueprints

A blueprint is how an OctoMesh application ships: one versioned bundle of CK model dependencies + seed data (the DataFlow with embedded pipeline YAML, the Application workload entity, domain seeds). One `InstallBlueprint` provisions any tenant; the blueprint is the **single source of truth** — do not maintain parallel standalone ImportRt files.

## Authoring checklist

- `blueprint.yaml` manifest: id, version, `ckModelDependencies` (version floors), `seedDataPath`, optional `requires` gate. Verified key set: `references/blueprint-authoring.md`; start from `examples/blueprint.yaml`.
- Seed data is **one** `entities:` sequence: DataFlow → pipelines (definition embedded as a block scalar) → Application → domain seeds, wired with `ParentChild`/`Executes` associations and associations to the tenant's `System.Communication` seeds (Pool `670…001`, Mesh Adapter `670…002`, Helm repo `670…003`). Skeleton with all the wiring: `examples/seed-entities-skeleton.yaml`.
- **Choose stable rtIds** (24 hex chars) in an own prefix range. Cross-references are byte-strings inside embedded pipeline YAML (`targetPipelineRtId`) — regenerating rtIds means updating both the outer entity ids and every embedded reference.
- Seeded entities the user may edit need `System/RtBlueprintLocked-1: false` — blueprint updates rewrite locked entities and skip unlocked ones.
- Validate early, always: `octo-bpm -c validate -p <blueprint dir>`.

## Install, update, and their sharp edges

```bash
octo-bpm -c publish -p <blueprint dir> --catalog <catalog>
octo-cli -c InstallBlueprint -b <Name>-<version>
octo-cli -c DeployDataFlow -id <dataflow rtId>
octo-cli -c DeployWorkload -id <application rtId>
```

- `InstallBlueprint` is **idempotent per version** — a same-version re-install skips the seed. `-f` (ReApply) is a full re-seed that ignores locks and does **not** prune stale association edges; clean re-seed = delete app entities, then `-f`.
- Catalog changes need `RefreshBlueprintCatalogs` before the new version is visible.
- Full semantics (update diff behavior, uninstall, requires-gate skip): `references/install-and-update.md`.

## The app workload

The Application entity references a Helm chart + image; the Communication Operator deploys it. The chart contract (env `PORT` + `UPSTREAM_URL`, probes on `GET /`, `image.tag` defaults to `appVersion` — **never pin image tags in blueprints**), the SPA + thin-proxy image pattern, ingress `Hostname` templating (`${octo.tenantId}`, `{{domain.default}}`), and triage via `LastDeploymentError` without kubectl: `references/app-workload-and-helm.md`.

**Changing an Application's chart requires `UndeployWorkload` + fresh `DeployWorkload`** — a plain redeploy collides on helm resource names.

## Additional resources

### Reference files

- **`references/blueprint-authoring.md`** — manifest keys, seed structure, embedding pipelines, rtId strategy, locking
- **`references/install-and-update.md`** — publish/install/update/uninstall semantics and pitfalls
- **`references/app-workload-and-helm.md`** — Application entity, chart contract, ingress/hostname, deployment troubleshooting

### Examples

- **`examples/blueprint.yaml`** — minimal realistic manifest
- **`examples/seed-entities-skeleton.yaml`** — commented seed with full association wiring (parses as valid YAML incl. the embedded pipeline)
- **`examples/Dockerfile`** — multi-stage SPA + proxy image
- **`examples/chart-contract.md`** — what a compatible web-app chart must fulfill
