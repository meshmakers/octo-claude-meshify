# OctoMesh Docs Site Map

Lookup map for `https://docs.meshmakers.cloud` — the public OctoMesh documentation site — for developers building or migrating an application onto OctoMesh. Use this file to construct doc URLs directly instead of guessing, and to find the right section for a given task.

## 1. URL construction rule

All page URLs live under one prefix and follow a predictable scheme (the section tables below name pages by their path segment):

```
https://docs.meshmakers.cloud/docs/<section>/<path>
```

Rules to apply when building a URL:

- **Pages have no file extension** — never append `.md` or `.html`.
- **A section landing page lives at the folder URL** — e.g. `https://docs.meshmakers.cloud/docs/userGuide/studio` (verified live, section 4); there is no `.../index` variant.
- **A few pages replace their last URL segment with a custom slug.** Confirmed case: the stream-data archives reference lives at `.../dataAccess/streamData/archives/reference` — **not** `.../streamData/archives/archives` and **not** `.../streamData/reference`. Only the last segment differs; the folder path stays intact.
- **When a constructed URL 404s**, do not keep guessing variants: use the site's built-in search or navigate the section's sidebar from a known-good page (e.g. the section landing page) to locate the real URL.

## 2. Sections an app developer needs

| Need | Pages (path segments under `/docs/`) | Public URL prefix |
|---|---|---|
| Construction Kit modeling guide | `technologyGuide/constructionKits/` (`intro.md`, `design.md`, `createConstructionKitLibrary.md`, `versioningRules.md`, `migrations.md`, `blueprints.md`, `libraryManagement.md`, `reference/`) | `/docs/technologyGuide/constructionKits/` |
| Published CK library reference (types/attributes/enums/associations per library+version) | `technologyGuide/constructionKits/libraries/<Library>-<major>/` (e.g. `System-2`, `System/Communication-3`, `Basic-2`) | `/docs/technologyGuide/constructionKits/libraries/` |
| Data flows / pipeline concepts | `technologyGuide/communication/dataFlows/intro.md`, `EdgeMeshPipelines.md` | `/docs/technologyGuide/communication/dataFlows/` |
| Pipeline node reference (all node types) | `technologyGuide/communication/dataFlows/nodes/` — see the Node Lookup Protocol in section 3 | `/docs/technologyGuide/communication/dataFlows/nodes/` |
| Pipeline triggers | `technologyGuide/communication/pipelineTriggers/intro.md` (PipelineTrigger entities, cron scheduling), trigger node docs under `nodes/trigger/`, **plus** the CLI commands `DeployTriggers` / `UndeployTriggers` under `communication-services` | `/docs/technologyGuide/communication/pipelineTriggers/`, `/docs/technologyGuide/communication/dataFlows/nodes/trigger/`, `/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/` |
| Adapters (Mesh, Modbus, SAP, Zenon, Simulation) | `technologyGuide/communication/adapters/intro.md`, `adapters/types/*.md` | `/docs/technologyGuide/communication/adapters/` |
| Formula expressions (mXparser-based) | `technologyGuide/communication/formulaExpressions.md` | `/docs/technologyGuide/communication/formulaExpressions` |
| octo-cli command reference | `technologyGuide/tools/octo-cli/intro.md`, `common-workflows.md`, `command-reference/<category>/<Command>.md` — see table below | `/docs/technologyGuide/tools/octo-cli/` |
| Blueprints (packaging CK model + seed data for install) | `userGuide/studio/repository/blueprints.md` (concept, in Refinery Studio UI terms) and the CLI commands `InstallBlueprint`, `UpdateBlueprint`, `UninstallBlueprint`, `ListBlueprints`, `ListBlueprintInstallations`, `GetBlueprintHistory`, `PreviewBlueprintUpdate`, `RefreshBlueprintCatalogs` under `command-reference/asset-repository-services/` | `/docs/userGuide/studio/repository/blueprints`, `/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/` |
| GraphQL / asset-repository query API | `technologyGuide/dataAccess/` (`constructionKit/retreive/*`, `runtime/*`, `streamData/*` — GraphQL query/mutation shapes for CK metadata, runtime entities, and stream data); DTO/type reference under `apiReference/Communication.Contracts/graphql/` and `.../datatransferobjects/` | `/docs/technologyGuide/dataAccess/`, `/docs/apiReference/Communication.Contracts/` |
| Identity / auth (clients, scopes, providers, roles) | `technologyGuide/identityService/` (`authentication.md`, `clients-and-scopes.md`, `identity-providers.md`, `groups.md`, `users-and-roles.md`, `cross-tenant-authentication.md`, `email-domain-group-rules.md`); matching CLI commands under `command-reference/identity-services/` | `/docs/technologyGuide/identityService/`, `/docs/technologyGuide/tools/octo-cli/command-reference/identity-services/` |
| Getting started | `technologyGuide/gettingStartedLocally/intro.md` (local/dev-cluster oriented); `developerGuide/gettingStarted/` (`intro.md`, `systemRequirements.md`, `developmentEnvironment.md`, `configureGit.md` — this subtree is written for engineers building the OctoMesh platform itself, not for app developers consuming it; treat it as background, not a primary entry point) | `/docs/technologyGuide/gettingStartedLocally/`, `/docs/developerGuide/gettingStarted/` |
| Glossary | `glossary/intro.md`, `techTerms.md`, `devTerms.md`, `apiTerms.md` | `/docs/glossary/` |

Other sections that exist but are secondary for app developers: `technologyGuide/achitecture/intro.md` (platform architecture overview — note the path really is spelled `achitecture`; the URL uses that spelling), `technologyGuide/tools/octo-mcp-service/` (MCP server for OctoMesh, separate from this plugin), `apiReference/Sdk.*` and `apiReference/ConstructionKit.Contracts` (generated .NET SDK type reference, useful only when writing custom adapters/nodes in C#), `userGuide/studio/` (Refinery Studio UI manual — useful for the human operating the tenant, not for CLI/API-driven app builds).

### Conceptual deep-dives (developerGuide/solutionArchitecture)

Three handwritten pages under `developerGuide/solutionArchitecture/` are genuinely useful for an app developer, distinct from the rest of `developerGuide/` (which targets engineers building the OctoMesh platform itself):

- `intro.md` — platform architecture overview: the Data Mesh paradigm as OctoMesh implements it, system components, and how they fit together (`/docs/developerGuide/solutionArchitecture/intro`).
- `dataModelConcepts.md` — how Construction Kits, Runtime entities, and associations relate conceptually, before diving into the per-library reference tables (`/docs/developerGuide/solutionArchitecture/dataModelConcepts`).
- `apiIntegration.md` — how to integrate with OctoMesh APIs: authentication, GraphQL operations, and the service client SDK, at a level above the raw DTO reference in `apiReference/` (`/docs/developerGuide/solutionArchitecture/apiIntegration`).
- `adapterDevelopment.md` — writing custom Edge/Mesh adapters when the built-in node set (section 3) is not enough (`/docs/developerGuide/solutionArchitecture/adapterDevelopment`).

### Per-integration setup articles

`technologyGuide/communication/articles/<system>/` holds narrative setup guides for specific external systems, each with an `intro.md` and usually a `setup.md`: `sap/` (incl. `rfcserver.md`), `zenon/`, `teamsBot/`, `microsoft365Email/`, `signal/`. Use these alongside the matching node docs (e.g. `nodes/extract/sapLogin_1.md`) when integrating one of these specific systems rather than building a generic HTTP/polling pipeline.

### Generated per-node C# API reference

Separate from the narrative node docs in section 3, `apiReference/Adapters/MeshNodes/<category>/` and `apiReference/Adapters/Sap/` hold auto-generated .NET type reference pages (constructor signatures, property tables) for the same nodes. Consult these only when writing custom C# pipeline logic (e.g. inside an `executeCSharp_1` node) that needs the exact CLR type shape — the narrative pages in section 3 are the primary reference for configuring a node in YAML.

## 3. Pipeline node lookup protocol

Node documentation pages live at a fixed, predictable path:

```
technologyGuide/communication/dataFlows/nodes/<category>/<camelCaseNodeName>_<version>.md
→ https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/<category>/<camelCaseNodeName>_<version>
```

`<category>` is one of five fixed values. `<camelCaseNodeName>` is the node's config `type` value in camelCase, and `<version>` is an integer (a node can have multiple doc pages across its version history, e.g. `applyChanges_1.md` and `applyChanges_2.md` both exist for the ApplyChanges node — always check for the highest available version number for current behavior).

The five categories, each with 3-4 example pages (verified live; **the node set evolves, so treat this as a sample, not a catalog**):

| Category | One-line description | Example node pages |
|---|---|---|
| `trigger/` | Starts a pipeline execution — HTTP request, polling, timer, inbound event, email, or a system event such as a watched entity change | `fromHttpRequest_1`, `fromPolling_1`, `fromWatchRtEntity_1`, `fromPipelineTriggerEvent_1` |
| `extract/` | Reads data from a source system or from the OctoMesh runtime model into the pipeline's data context | `getRtEntitiesByType_1`, `makeHttpRequest_1`, `getAssociationTargets_1`, `getFileSystemContent_1` |
| `transformation/` | Reshapes, computes, filters, or enriches data already in the pipeline's data context | `map_1`, `dataMapping_1`, `createUpdateInfo_1`, `math_1` |
| `load/` | Writes the pipeline's data out to a target — the runtime model, a time-series archive, an outbound webhook, or a downstream system | `applyChanges_2`, `saveStreamDataInArchive_1`, `toWebhook_1`, `toPipelineDataEvent_1` |
| `control/` | Governs pipeline flow — branching, iteration, and sub-flow selection | `if_1`, `foreach_1`, `for_1`, `switch_1` |

Before configuring an unfamiliar node, fetch its doc page. Do not guess a node's YAML config shape from its name or from memory of a similar node — field names, required vs. optional fields, and `targetValueWriteMode`/`targetValueKind` semantics (documented once, centrally, on the nodes overview page) vary per node and per version.

To discover the **current** full node list rather than relying on a point-in-time catalog:

- Fetch the category's `_category_.json`-driven sidebar by loading the nodes overview page, `https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/intro` (this also documents the shared `path`, `targetPath`, `targetValueWriteMode`, `targetValueKind` fields common to all nodes), then follow the sidebar under "Nodes" for the live category listings.
- Or WebFetch a category index directly, e.g. `https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/trigger/fromHttpRequest_1` and use the page's own "next/previous" or sidebar links to enumerate siblings.

Node counts at verification time (2026-08-19, for scale/sanity only — do not treat as authoritative): trigger 13, extract 24, transformation 29, load 15, control 6.

## 4. Verified live URLs (2026-08-19)

The following constructed URLs were checked live with WebFetch on 2026-08-19 and confirmed to resolve with the expected content:

| URL | Confirms |
|---|---|
| `https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/trigger/fromHttpRequest_1` | Node page pattern (`<category>/<camelCaseName>_<version>`) |
| `https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/asset-repository-services/InstallBlueprint` | octo-cli command-reference page pattern (`command-reference/<category>/<CommandName>`) |
| `https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/intro` | Plain `<section>/<file>` mapping |
| `https://docs.meshmakers.cloud/docs/userGuide/studio` | `index.md` → containing-folder URL rule |
| `https://docs.meshmakers.cloud/docs/technologyGuide/communication/formulaExpressions` | Plain `<section>/<file>` mapping |
| `https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/streamData/archives/reference` | `slug:` frontmatter override rule (replaces only the last path segment) — the naive guess `.../streamData/reference` **404s**; the correct URL keeps the `archives/` segment |

## Known gaps

- As of 2026-08-19 there are **no dedicated command-reference pages for `octo-ckc` or `octo-bpm`** under `technologyGuide/tools/`. CK library publishing/versioning is covered narratively in `technologyGuide/constructionKits/createConstructionKitLibrary` and `constructionKits/reference/repositories`, and blueprint operations via the `octo-cli` `asset-repository-services` commands listed in section 2. For the exact flags of an `octo-ckc`/`octo-bpm` command, use the tools' built-in help: `octo-ckc -c <Command> --help` / `octo-bpm -c <command> --help`.
