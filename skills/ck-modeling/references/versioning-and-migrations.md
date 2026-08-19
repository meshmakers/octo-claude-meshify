# Versioning and Migrations

CK model versions follow SemVer and, once published, are **immutable**. Any schema change — a new attribute, a renamed type, a widened value range — requires a new version. Re-importing the exact same version into a tenant is a no-op: nothing changes, because nothing in the model changed. This is why iterating on an unreleased model locally (before it has ever been published) is handled differently from evolving a released one — see "Iterating before the first release" below.

## The version-bump gate

`octo-ckc ValidateVersion` classifies the diff between your declared version and the last published baseline, and enforces that the bump is large enough:

```bash
octo-ckc -c ValidateVersion -p './ConstructionKit'
```

| Change | Required bump |
|---|---|
| New type, attribute, or enum member | MINOR |
| Flagging an existing attribute `isRuntimeState: true` | MINOR |
| Removing or renaming a type/attribute, changing an attribute's `valueType` | MAJOR |
| Description/metadata-only changes | PATCH |
| Changing a type's `displayNameRule` / `displayDescriptionRule` | PATCH |

If the declared version doesn't satisfy the required bump, `ValidateVersion` fails with a non-zero exit code (gate on `exit code != 0`, not `== 1`). Useful flags:

| Flag | Description |
|---|---|
| `-p` / `--path` | Root path(s) of the CK model directory. Multiple paths validate in dependency order. |
| `-cn` / `--catalogName` | Restrict baseline retrieval to one named catalog instead of querying all readable catalogs. |
| `-rf` / `--refresh` | Force a catalog cache refresh before determining the baseline. |
| `-o` / `--output` | Write the validation report as Markdown to a file. |
| `-cl` / `--changelog` | Write/update the version's section in `CHANGELOG.md` next to `ckModel.yaml` after a successful validation. |
| `-rmm` / `--requireMigrationForMajor` | Escalate a missing migration script for a required major bump from a warning to an error. |

A **dependency bump cascades**: if `Basic` publishes `1.3.0` and your model still declares (and was compiled against) `Basic-[1.2,2.0)`, the compiler froze the resolved dependency in at `1.2.x`. Your model must be recompiled — which re-resolves the range to `1.3.0` — and republished with its own version bump, because the resolved dependency set changed. `ValidateVersion` enforces this.

## Iterating before the first release

While a model has never been published to a shared catalog, there is no baseline to bump against and no consumer depending on the exact version. The practical loop is: edit YAML → `octo-ckc -c Compile` → `octo-cli -c ImportCk -f <compiled-file> -w` into your dev tenant → inspect the result → repeat. Re-importing the **same** version number after an edit is a no-op (the tenant already holds that version, unchanged), so a real edit needs either:

- a version bump on every iteration (clean, matches how the platform is meant to be used), or
- deleting and recreating the dev tenant to pick up an edited-but-not-rebumped model.

Once the model is published and something else (a blueprint, another model, a running tenant) depends on a specific version, treat every subsequent change as a normal version bump — see `catalogs-and-publishing.md` for the publish step.

## Migrations

A migration script transforms existing runtime entity data during a CK model version upgrade — renaming a type, merging two types into one, defaulting a newly-required attribute, and similar structural changes. Migrations are optional: only add one when a version bump actually needs to move *existing tenant data*, not just the schema.

### File layout

```
ConstructionKit/
├── ckModel.yaml
├── types/
└── migrations/
    ├── migration-meta.yaml
    ├── 1.0.0-to-2.0.0.yaml
    └── 2.0.0-to-2.1.0.yaml
```

Migration files placed in this folder ship with the model automatically when it is compiled and published — no extra packaging step is needed.

### migration-meta.yaml

```yaml
$schema: https://schemas.meshmakers.cloud/ck-migration-meta.schema.json
ckModelId: MyModel-2.1.0
migrations:
  - fromVersion: "1.0.0"
    toVersion: "2.0.0"
    scriptPath: 1.0.0-to-2.0.0.yaml
    description: "Rename Widget type to Component"
    breaking: true
  - fromVersion: "2.0.0"
    toVersion: "2.1.0"
    scriptPath: 2.0.0-to-2.1.0.yaml
    description: "Add default status attribute to all Components"
```

Only define an entry for a version step that needs a **data** transformation — the engine auto-bridges any gap where the tenant's existing data is already compatible with the new schema (see below), so there is no need for empty placeholder entries.

### Migration script

```yaml
$schema: https://schemas.meshmakers.cloud/ck-migration.schema.json
sourceVersion: "1.0.0"
targetVersion: "2.0.0"
description: |
  Renames the Widget CK type to Component.

preConditions:
  - type: EntityExists
    target:
      ckTypeId: MyModel/Widget

steps:
  - stepId: rename-widget-to-component
    description: "Change CkTypeId from Widget to Component"
    action: Transform
    target:
      ckTypeId: MyModel/Widget
    transform:
      type: ChangeCkType
      newCkTypeId: MyModel/Component
    onConflict: Skip
    continueOnError: false

postValidations:
  - validationId: no-widgets-remain
    description: "Ensure no entities with legacy Widget type exist"
    type: NoEntitiesOfType
    target:
      ckTypeId: MyModel/Widget
    severity: Warning
```

**Pre-conditions** (skip the step if not met): `EntityExists`, `EntityNotExists`, `CkModelVersionInstalled`, `AttributeEquals`.

**Actions**: `Transform` (change type / rename / set / copy / delete an attribute value), `Update` (set attribute values), `Delete` (remove matching entities), `Add` (insert new entities).

**Transform types** (used with `action: Transform`): `ChangeCkType`, `SetValue`, `RenameAttribute`, `CopyAttribute`, `DeleteAttribute`, `MapValue`.

**Target selection**: `ckTypeId`, `rtId`, `rtWellKnownName`, `filter` (attribute/operator/value, combinable with `and`; operators `Eq`, `Ne`, `Exists`, `NotExists`, `Contains`, `StartsWith`), `blueprintSourceOnly` (restrict to blueprint-created entities).

**Conflict behavior**: `Fail` (default), `Skip`, `Overwrite`.

**Post-validations** (`EntityExists`, `NoEntitiesOfType`, `EntityCount`) each carry a `severity` of `Error` or `Warning` — warnings log but don't fail the migration.

### Version-gap resolution

The engine resolves the migration path between a tenant's current version and the new model version with a multi-layer strategy: an exact step if one exists, a multi-hop chain of steps otherwise, and two auto-bridge cases at the ends of the chain — no manual bridge entries required:

- **Start gap**: the tenant is older than the earliest defined migration entry point → a no-op bridge (the tenant's data is already compatible with the start of the chain).
- **End gap**: the defined chain doesn't reach the exact new version → all defined steps run, and the remaining bump is treated as schema-only.

Example — a tenant at `2.2.0` receiving model version `3.1.2`, with migrations defined starting at `3.0.1`:

```
2.2.0 → 3.0.1  (auto-bridge, no-op)
3.0.1 → 3.0.2  (migration script executed)
3.0.2 → 3.0.3  (migration script executed)
3.0.3 → 3.1.0  (migration script executed)
3.1.0 → 3.1.1  (migration script executed)
3.1.1 → 3.1.2  (auto-bridge, schema-only)
```

### Options and best practices

Programmatic migration execution accepts `DryRun` (simulate without writing), `CreateBackup` (snapshot the tenant first), and `ContinueOnError`.

- Make each step idempotent, or guard it with `preConditions`.
- Use `continueOnError: true` for steps that only partially apply across tenants (e.g. merging several legacy subtypes where not every tenant has every subtype).
- Prefer `severity: Warning` post-validations to verify without blocking.
- Keep the migration chain linear — each `fromVersion` should appear at most once.

## Upgrading an already-provisioned tenant

Publish the rebuilt/republished model to the catalog first, *then* upgrade the dependency chain on the tenant — not the other way around. Running the tenant-side upgrade after only a dependency was published, while a dependent model still pins the old dependency version, leaves the dependent unresolvable: its pins no longer resolve against what the tenant now holds, so it flips to a failed/unresolved model state and its types disappear from the tenant's GraphQL schema until the dependent is republished and the tenant re-synced.

Pre-flight the upgrade before running it:

```bash
octo-cli -c CheckUpgrade -cn "PublicGitHubCatalog" -m "MyModel-2.0.2"
```

```
Model:              MyModel
Installed Version:  2.0.0
Target Version:     2.0.2
Upgrade Needed:     True
Migration Path:     True
Breaking Changes:   False
```

`CheckUpgrade` takes `-cn`/`--catalogName` (required) and `-m`/`--modelId` (required, e.g. `MyModel-2.0.2`). See `catalogs-and-publishing.md` for the full catalog-import lifecycle (`LibraryStatus`, `ImportFromCatalog`, `FixAll`, `RefreshCatalogs`).

## See also

- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/versioningRules>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/migrations>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/blueprints>
- octo-cli command reference: [CheckUpgrade](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/CheckUpgrade), [ImportCk](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/ImportCk)
