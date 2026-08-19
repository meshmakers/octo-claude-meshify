# Blueprint Lifecycle — Publish, Install, Roll Out, Update, Uninstall

How a finished blueprint gets onto a tenant and stays current. Manifest and seed
authoring: [blueprint-authoring.md](blueprint-authoring.md).

Command reference pages (per command):
`https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/<Command>`
and `.../communication-services/<Command>`.

## Tenant Prerequisites

A blueprint that seeds DataFlows, Pipelines, or Applications needs the
communication stack active on the target tenant:

```bash
octo-cli -c EnableCommunication                     # enables the stack; the service-managed
                                                    # System.Communication blueprint seeds the
                                                    # Pool (670…001) and Mesh Adapter (670…002)
octo-cli -c DeployPool -id 670000000000000000000001 # operator creates the pool resources
octo-cli -c DeployWorkload -id 670000000000000000000002   # roll out the Mesh Adapter
octo-cli -c GetAdapters -j                          # verify: Mesh Adapter is Online
```

Check adapter state before installing — pipelines deploy onto the adapter, and an
offline adapter turns every later step into a confusing failure. The full tenant-readiness
checklist (CLI contexts, login, pool deployment, verification commands) is owned by the
meshify skill's `references/tenant-prerequisites.md`; the commands above are the minimum
for blueprint installs.

The CK models named in `ckModelDependencies` must be resolvable from a CK catalog
the environment's asset repository is configured to read (`octo-cli -c ListCatalogs`
shows the sources; `octo-cli -c RefreshCatalogs` refreshes their caches). Publishing
CK models is covered by the CK modeling skill.

## Publish to a Blueprint Catalog (`octo-bpm`)

`octo-bpm` is the blueprint authoring tool. Commands (invoked as
`octo-bpm -c <command>`): `new`, `validate`, `pack`, `publish`, `unpublish`, `list`,
`get`, `catalogs`, `config`, `version`.

```bash
octo-bpm -c validate -p ./MyApp/1.0.0
octo-bpm -c publish  -p ./MyApp/1.0.0 --catalog LocalFileSystemBlueprintCatalog -f
```

- `publish` validates first, then writes the blueprint into the target catalog.
  Default catalog is `LocalFileSystemBlueprintCatalog`; `-f/--force` replaces an
  already-published version.
- The local file-system catalog lives on the machine that runs the **asset
  repository service** (layout `blueprints/v1/<Name>/<version>/…` under the
  configured root, default `~/.octo/local-blueprint-catalog`). It is therefore only
  usable when you control that host — self-hosted clusters and local development.
- Managed environments read a fixed, operator-configured set of blueprint catalogs.
  There is currently **no channel for publishing an own blueprint into a managed
  environment's catalog** — configurable per-environment catalogs are planned but not
  yet available. Until then, own blueprints install only on environments whose catalog
  configuration is under one's own (or the operator's) control: self-hosted clusters
  via the local file-system catalog, or a GitHub-Pages catalog the operator adds.

After publishing, make the environment see it:

```bash
octo-cli -c RefreshBlueprintCatalogs        # forced refresh, no service restart needed
octo-cli -c ListBlueprints                  # blueprint must appear here before install
```

## Install

```bash
octo-cli -c InstallBlueprint -b MyApp-1.0.0
```

What happens on apply (see
<https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/blueprints>):
dependency closure is resolved and topo-sorted, CK models are auto-imported, seed
data is upserted by rtId, entities are stamped with ownership attributes, and a
history entry is recorded.

Confirm the output: `Success: true`, `ApplicationMode: Initial`, your CK model listed
under the loaded models, no warnings. Two silent-looking outcomes to know:

- **Same-version re-install is a no-op.** The version is recorded as applied, so a
  plain re-install skips the seed import — even if you deleted the entities in the
  meantime. Pass `-f` to force.
- **`requires:` gate mismatch is a successful no-op** (`WasSkipped: true`). Check the
  manifest's `requires:` against the environment.

### Force Re-Apply (`-f`) Semantics

`InstallBlueprint -b <id> -f` re-applies the seed via upsert at the same version —
the recovery / factory-reset path. Know exactly what it does:

- **Full re-seed ignoring locks**: every seed entity is reset to seed values,
  including entities shipped with `RtBlueprintLocked-1: false`. Only attributes
  flagged `isRuntimeState: true` in their CK model keep the tenant's live value.
- **Association edges are not pruned**: force re-apply upserts attributes but does
  NOT remove association edges that are no longer in the seed. Changing an
  association in the seed and re-applying leaves the old edge behind. For a clean
  re-seed, delete the app's entities first (GraphQL
  `runtime.runtimeEntities.delete`, **unversioned** ckTypeIds), then
  `InstallBlueprint -f`.

## Post-Install Rollout

Installing only **seeds** entities. Trigger the rollout explicitly afterwards:

```bash
octo-cli -c DeployDataFlow -id <dataflowRtId>       # pushes all pipelines to their adapters
octo-cli -c DeployWorkload -id <applicationRtId>    # operator helm-installs the web app
```

- Use `DeployDataFlow` (not per-pipeline `DeployPipeline`) as the standard rollout.
- Verify with `octo-cli -c GetDataFlowStatus -id <dataflowRtId> -j` and
  `octo-cli -c GetPipelineStatus -id <pipelineRtId> -j`.
- Application workload details and triage: [app-workload-and-helm.md](app-workload-and-helm.md).

Put these exact commands (with the seed's fixed rtIds) into the blueprint's
`description:` — they are the consumer's runbook.

## Update to a New Version

```bash
octo-cli -c PreviewBlueprintUpdate -tv MyApp-1.1.0 [-m Safe|Merge|Full|Migration]
octo-cli -c UpdateBlueprint        -tv MyApp-1.1.0 [-m <mode>] [-dr]
```

| Mode | Behaviour |
|---|---|
| `Safe` | Add new entities only; existing entities untouched. |
| `Merge` (default) | Add new + update **locked** entities. Unlocked entities raise `UserModified` conflicts (default resolution: skip). |
| `Full` | Merge, plus delete entities no longer in the seed. Unlocked → conflict. |
| `Migration` | Run the migration script from the installed to the target version. Required for non-additive changes. |

Key consequences for seed design:

- Locked entities (the default) are **updated** on Merge/Full — the blueprint stays
  authoritative for them.
- Unlocked entities (`RtBlueprintLocked-1: false`) are **skipped** — user edits
  survive updates.
- Runtime-state attributes are preserved either way.

`-dr` simulates without persisting. A pre-update backup is created by default.
After an update that changed pipeline YAML or workload configuration, re-run the
rollout commands (`DeployDataFlow` / `DeployWorkload`) — updating the seed does not
redeploy anything by itself.

## Inspect State

```bash
octo-cli -c ListBlueprintInstallations      # what is installed on the tenant (+ IsDependency)
octo-cli -c GetBlueprintHistory             # append-only apply log with entity counts
```

Both are also visible in Refinery Studio under **Repository → Blueprints**
(<https://docs.meshmakers.cloud/docs/userGuide/studio/repository/blueprints>).

## Uninstall

Order matters when the seed contains deployed workloads:

```bash
octo-cli -c UndeployWorkload -id <applicationRtId> -y   # helm-uninstall FIRST
octo-cli -c UndeployDataFlow -id <dataflowRtId>         # remove pipelines from adapters
octo-cli -c UninstallBlueprint -n MyApp [-c] [-y]       # NAME only, no version
```

`UninstallBlueprint` erases the blueprint-owned entities. `-c/--cascade` also removes
dependent blueprints and orphaned dependencies. Uninstalling without undeploying
first strands the Helm release and the adapter routes.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Blueprint missing from `ListBlueprints` after publish | Catalog cache stale. Run `RefreshBlueprintCatalogs` (forced refresh), then list again. |
| Install reports `WasSkipped: true` | `requires:` gate mismatch — the environment is not in the allowed list. Successful no-op by design. |
| Install fails with a missing CK dependency | The model in `ckModelDependencies` is not in any configured CK catalog (or cache stale — `RefreshCatalogs`). |
| Re-install after seed edit changes nothing | Same-version installs no-op. Use `-f`, or bump the blueprint version. |
| Association change did not take effect after `-f` | Force re-apply does not prune old edges. Delete the entities, then re-apply. |
| Seeded user data reset after maintenance | Someone ran `InstallBlueprint -f` (ignores locks). Use `UpdateBlueprint` for regular upgrades; reserve `-f` for factory reset. |
| Endpoints dead right after install | Rollout step skipped — run `DeployDataFlow`, check `GetDataFlowStatus`, confirm the adapter is Online (`GetAdapters -j`). |
