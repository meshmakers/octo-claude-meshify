# Pipeline Fundamentals

Core concepts for OctoMesh ETL pipelines: the runtime entities that carry them, the YAML definition format, and the DataContext that flows through the nodes. Concept reference: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/intro>.

## Runtime entities: DataFlow, Pipeline, PipelineTrigger

Pipelines are not standalone files — they live as runtime entities in the tenant's Asset Repository:

| Entity (CK type) | Role |
|---|---|
| `System.Communication/DataFlow` | Logical grouping of pipelines that work together. Establishes a shared topic exchange in the event hub, enabling inter-pipeline communication (`ToPipelineDataEvent@1` → `FromPipelineDataEvent@1`). |
| `System.Communication/Pipeline` | One pipeline. Carries the YAML definition in its `System.Communication/PipelineDefinition-1` attribute and is linked to a DataFlow (`System/ParentChild`) and to the Adapter that executes it (`System.Communication/Executes`). |
| `System.Communication/PipelineTrigger` | Child of a DataFlow; starts pipelines on a cron schedule via the Bot Service. Target pipelines must use the `FromPipelineTriggerEvent@1` trigger node. See <https://docs.meshmakers.cloud/docs/technologyGuide/communication/pipelineTriggers/intro>. |
| `System.Communication/Adapter` | The runtime that executes pipelines. On every communication-enabled tenant, the platform seeds a Mesh Adapter with the fixed rtId `670000000000000000000002`. |

```
DataFlow
  ├── Pipeline         (ParentChild)  ── Executes ──►  Adapter
  └── PipelineTrigger  (ParentChild)  ── Triggers ──►  Pipeline(s)
```

A minimal Pipeline entity in an ImportRt/seed file:

```yaml
- rtId: 'aaa000000000000000000002'
  ckTypeId: System.Communication/Pipeline-1
  associations:
    - roleId: System/ParentChild-1
      targetRtId: 'aaa000000000000000000001'          # the DataFlow
      targetCkTypeId: System.Communication/DataFlow-1
    - roleId: System.Communication/Executes-1
      targetRtId: '670000000000000000000002'          # the tenant's Mesh Adapter
      targetCkTypeId: System.Communication/Adapter-1
  attributes:
    - id: System/Name-1
      value: "my-pipeline"
    - id: System/Enabled-1
      value: true
    - id: System.Communication/DeploymentState-1
      value: 0
    - id: System.Communication/PipelineDefinition-1
      value: |
        triggers: ...
        transformations: ...
```

## Pipeline definition YAML shape

A pipeline definition has exactly two root keys:

```yaml
triggers:
  - type: NodeType@Version        # how the pipeline starts
    # trigger-specific properties
transformations:
  - type: NodeType@Version        # ordered processing steps
    description: optional free text shown in status/debug views
    # node-specific properties
  - type: If@1                    # control-flow nodes nest child steps
    path: $.count
    operator: GreaterThan
    value: 0
    valueType: Int
    transformations:
      - type: ChildNode@Version
```

Rules:

- Every node is named `Name@MajorVersion` (e.g. `FromHttpRequest@1`, `ApplyChanges@2`). The version is part of the type name.
- Nodes execute in list order; control-flow nodes (`If@1`, `Switch@1`, `ForEach@1`, `For@1`, `Group@1`) carry a nested `transformations:` list.
- `description` is accepted on every node and is the only universally-free field — use it.
- CK type references inside pipeline YAML are **unversioned**: `ckTypeId: MyModel/Task`, not `MyModel/Task-1`. (Entity files for ImportRt use versioned ids — pipeline YAML does not.)

### Strict deserialization — unknown keys are errors

The pipeline YAML deserializer is **strict**. Any property a node does not declare fails the deployment with an error like `Property 'version' not found on NodeDefinitionRoot`. Consequences:

- No root-level keys other than `triggers:` and `transformations:` — no `name:`, no `version:`.
- A typo in a property name is a deployment error, not a silent ignore.
- Adding properties to deployed YAML is **not backward compatible** across adapter versions — a node property that does not exist on the target adapter's version breaks the deploy.

When unsure about a node's exact properties, fetch the adapter's own schema: `octo-cli -c GetPipelineSchema -aid <adapterRtId> -o schema.json` returns a JSON Schema of every node the adapter supports, with required keys and enum values.

## The DataContext

Every pipeline execution owns a single mutable JSON document — the DataContext. Triggers seed it; every transformation node reads from it and writes to it via JSONPath.

### Addressing with JSONPath

| Pattern | Meaning |
|---|---|
| `$` | Root of the current context |
| `$.body.title` | Nested property access |
| `$.items[0]`, `$.items[*]` | Array index / all elements |

Writing to a path that does not exist creates the intermediate structure automatically.

### Reading: `path` / `valuePath`

Nodes read input with `path` (the node's working subtree) or `valuePath` (a single value). When a node offers both `value` (literal) and `valuePath`, the path form reads dynamically from the context.

### Writing: `targetPath` plus three modifiers

Documented centrally at <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/intro>:

| Property | Values | Effect |
|---|---|---|
| `targetValueWriteMode` | `Overwrite` (default), `Append`, `Prepend`, `Merge` | Replace the target; add to end/start of an array; deep-merge two objects |
| `targetValueKind` | `Simple` (default), `Array` | Write the value as-is, or wrap it in an array first |
| `documentMode` | `Extend` (default), `Replace` | Merge into the existing document, or clear it before writing |

The single most important combination — collecting multiple results into one array:

```yaml
targetPath: $.updates
targetValueWriteMode: Append
targetValueKind: Array
```

### What each trigger puts into the context

| Trigger | Initial context |
|---|---|
| `FromHttpRequest@1` | `$.path`, `$.method`, `$.body` (parsed JSON body), `$.query` (query parameters), `$.files`, `$.formData` |
| `FromWatchRtEntity@1` | `$.Document` — the changed entity |
| `FromPipelineDataEvent@1` | Whatever the sending pipeline's `ToPipelineDataEvent@1` placed at its `targetPath` |
| `FromExecutePipelineCommand@1`, `FromPipelineTriggerEvent@1` | Empty context |

## Node documentation lookup

Before configuring a node not shown in these references, fetch its documentation page. Node pages follow one URL pattern:

```
https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/<category>/<nodeName>_<version>
```

where `<category>` is one of `trigger`, `extract`, `transformation`, `control`, `load`, and `<nodeName>` is the node name in lowerCamelCase — e.g. `FromHttpRequest@1` → `.../nodes/trigger/fromHttpRequest_1`, `CreateUpdateInfo@1` → `.../nodes/transformation/createUpdateInfo_1`. The full docs-site map (including how to discover which nodes currently exist per category) is owned by the meshify skill's `references/docs-site-map.md`.

For the authoritative property list of any node on a **specific** environment, prefer `octo-cli -c GetPipelineSchema` over the docs — the schema is generated from the adapter build that actually runs the pipeline and never lags.
