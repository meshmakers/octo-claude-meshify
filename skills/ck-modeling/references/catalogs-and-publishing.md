# CK Catalogs and Publishing

Two different tools touch "catalogs," for two different purposes, and it's worth keeping them apart:

- **`octo-ckc` (the compiler)**, run on your machine, resolves the version ranges in your `ckModel.yaml` `dependencies:` list against catalogs *while compiling*, and can publish your compiled model to a catalog for others to consume.
- **The Asset Repository service**, reached through `octo-cli` or the UI, browses catalogs and imports already-compiled models *into a tenant*.

## Catalog sources

| Catalog | Description | Default |
|---|---|---|
| **EmbeddedResource** | Models embedded in the service binaries themselves (the platform's own `System` model ships this way). | Always active |
| **PublicGitHub** | Released CK libraries published by meshmakers (including `System`), served from the public GitHub Pages catalog at <https://meshmakers.github.io/> (verified in the tool's defaults). | Enabled |
| **PrivateGitHub** | Pre-release channel. In `octo-ckc` it defaults to the meshmakers pre-release catalog (enabled out of the box) and can be repointed at an own private GitHub repository (`Config -go/-gr/-gb/-gt`). | Disabled in production |
| **LocalFileSystem** | A local folder catalog, for development. Default root `~/.octo/local-catalog` on both `octo-ckc` and the tenant-facing service. | Disabled in production |

The "Default" column above is the **service-side** default (what a tenant's Asset Repository service can browse/import from out of the box). `octo-ckc`, running locally on your machine, has its own separate catalog configuration and enables its local file-system catalog by default (`LocalFileSystemCatalog` is the tool's default target for `Get`/`Publish`), configured with:

```bash
octo-ckc -c Config -lcp '<path>' -lce true   # local file-system catalog path + on/off
octo-ckc -c Config -go '<owner>' -gr '<repo>' -gb '<branch>' -ge true   # your own private GitHub catalog
octo-ckc -c GetCatalogs                       # list configured catalogs
```

## Clean-install behavior (verified 2026-08-19)

A fresh `octo-ckc` install — no `~/.octo-ckc/settings.json`, no caches, no credentials — resolves public dependencies out of the box: compiling a model that declares `System-[2.0,3.0)` on a clean machine downloaded and pinned `System` with zero configuration. Two behaviors to know:

- The version range resolves to the **highest version across all enabled catalogs**; the model is then fetched from the first catalog holding that version, checked in the order EmbeddedResource → LocalFileSystem → PrivateGitHub → PublicGitHub.
- Because the default-enabled PrivateGitHub catalog is the **pre-release** channel, a clean install can pin a dependency version that is not yet in the released (public) catalog. To build against released libraries only, disable it persistently: `octo-ckc -c Config -ge false` (re-enable with `-ge true`).

## System-managed models

Models named `System` or starting with `System.` (e.g. `System.Identity`, `System.Communication`) are managed by the OctoMesh services themselves: imported automatically at tenant creation/service startup, shown as "Service-Managed" in Library Overview, not updatable through Library Management, and excluded from `FixAll`. Your own models declare a dependency on `System` but never redefine or attempt to import it yourself.

## Version compatibility

Importing a library into a tenant checks **transitive compatibility** against every system-managed model the tenant has installed: walking the full dependency chain, for each dependency on a system-managed model the **major version must match**. `System-2.0.7` installed + a library requiring `System-2.0.3` → compatible (same major, installed is higher). `System-2.0.7` installed + a library requiring `System-3.0.0` → incompatible, and the import is refused.

## How dependency ranges resolve

Your source `ckModel.yaml` declares dependencies as ranges (`System-[2.0,3.0)`). `octo-ckc Compile` resolves each range against the configured catalogs, picks the **highest published version** that satisfies it, and freezes that exact version into the compiled output as a pin — this is why a dependency's later release doesn't retroactively affect an already-compiled model; the dependent must be recompiled and republished (see `versioning-and-migrations.md` for the version-bump gate this triggers).

Importing a compiled library into a tenant then resolves its full dependency tree: fetch the target model, walk its (already-exact) dependency list, recurse into sub-dependencies, deduplicate by model name keeping the highest required version, exclude system-managed models, and topologically order the result — dependencies import before dependents, each as its own background job.

## Developer loop

```bash
# 1. Scaffold (once)
octo-ckc -c New -p './ConstructionKit'

# 2. Compile — validates schema conformity and resolves dependencies
octo-ckc -c Compile -p './ConstructionKit' -o './out'

# 3. Import into your dev tenant to iterate
octo-cli -c ImportCk -f './out/<modelfile>.yaml' -w

# 4. Edit YAML, go back to step 2 — re-importing the SAME version is a no-op,
#    so bump the version (or delete/recreate the dev tenant) between edits
#    while the model has never been published; see versioning-and-migrations.md

# 5. Before publishing, gate the version bump
octo-ckc -c ValidateVersion -p './ConstructionKit'

# 6. Publish the compiled model to a catalog other consumers pull from
octo-ckc -c Publish -f './out/<modelfile>.yaml' -c 'LocalFileSystemCatalog'
```

`Publish` accepts `-f`/`--file` (required, the compiled model file from step 2), `-c`/`--catalog` (optional, catalog name, `LocalFileSystemCatalog` by default), and `-r`/`--replace` (optional, overwrite an existing entry for that exact version — normally publishing the same version twice without `-r` is rejected, consistent with versions being immutable). `Get -c <catalog> -n <modelId>` and `Find -id <modelId-range>` let you inspect what a catalog already holds before publishing.

For a self-hosted setup, `LocalFileSystemCatalog` (a shared folder or shared filesystem path both your build step and your tenant's Asset Repository service can read) is the natural target for models you author yourself; blueprint packaging then references the published `modelId-version` as a `ckModelDependencies` floor — see the blueprint-packaging skill.

## Tenant-side lifecycle commands

All of the following are `octo-cli` commands against the Asset Repository service; catalog import/refresh requires the **Data Model Management** authorization scope (`octo_api.data_model_management` or `octo_api`), included by default in the `TenantOwners` group. Read-only browsing (`ListCatalogs`, `ListCatalogModels`, `CheckDependencies`, `CheckUpgrade`, `LibraryStatus`) needs only `octo_api.read_only` or `octo_api`.

| Command | Purpose |
|---|---|
| `ListCatalogs` | List catalog sources the tenant service knows about. |
| `ListCatalogModels -cn <catalog> -q <term>` | List/search models in catalogs. |
| `CheckDependencies -cn <catalog> -m <modelId>` | Show the resolved dependency tree for a catalog model before importing it. |
| `CheckUpgrade -cn <catalog> -m <modelId>` | Pre-flight an upgrade (see `versioning-and-migrations.md`). |
| `LibraryStatus [-na] [-io]` | Installed models vs. catalog availability, with a needed-action column. |
| `ImportFromCatalog -cn <catalog> -m <modelId> [-w]` | Import a model and its full dependency tree into the active tenant. |
| `RefreshCatalogs [-cn <catalog>]` | Refresh catalog caches (all, or one named catalog). |
| `FixAll [-w] [-y]` | Import every model in the tenant that needs an update/fix — skips system-managed models. |
| `UpdateSystemCkModel -tid <tenantId>` | Update a tenant's `System` model to the latest service-shipped version. |

Example — install a public library and its dependencies, waiting for completion:

```bash
octo-cli -c ImportFromCatalog -cn "PublicGitHubCatalog" -m "Industry.Energy-2.0.0" -w
```

`CheckDependencies` output shows exactly what will be touched before you commit to it:

```
Industry.Energy v2.0.0  [INSTALL]
  Industry.Basic v2.1.0  [INSTALL]
    Basic v2.0.2  [NONE] (installed: v2.0.2)
    System v2.0.7  [NONE] (service-managed: v2.0.7)

Models to import (2):
  Industry.Basic-2.1.0
  Industry.Energy-2.0.0
```

## REST API (for programmatic access)

| Method | Endpoint | Scope |
|---|---|---|
| `GET` | `system/v1/ckmodelcatalog` | List all models from all catalogs |
| `GET` | `system/v1/ckmodelcatalog/search?q={term}` | Search models |
| `GET` | `system/v1/ckmodelcatalog/catalogs` | List catalog sources |
| `POST` | `system/v1/ckmodelcatalog/refresh` | Refresh all catalog caches |
| `GET` | `{tenantId}/v1/models/LibraryStatus` | Merged view: installed + catalog |
| `POST` | `{tenantId}/v1/models/ResolveDependenciesBatch` | Resolve dependency trees |
| `POST` | `{tenantId}/v1/models/ImportFromCatalogBatch` | Batch import with jobs |
| `POST` | `{tenantId}/v1/models/CheckUpgrade` | Pre-flight migration check |
| `POST` | `{tenantId}/v1/models/ImportFromCatalog` | Single model import |

## See also

- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/libraryManagement>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/createConstructionKitLibrary>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/versioningRules>
- octo-cli command reference: [ImportCk](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/ImportCk), [ImportFromCatalog](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/ImportFromCatalog), [ListCatalogs](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/ListCatalogs), [ListCatalogModels](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/ListCatalogModels), [CheckDependencies](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/CheckDependencies), [CheckUpgrade](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/CheckUpgrade), [LibraryStatus](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/LibraryStatus), [RefreshCatalogs](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/RefreshCatalogs), [FixAll](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/FixAll), [UpdateSystemCkModel](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/UpdateSystemCkModel)
