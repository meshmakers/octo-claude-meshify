---
name: pipeline-api
description: Build declarative HTTP APIs and data pipelines on the OctoMesh Mesh Adapter - pipeline YAML with node@version steps, the DataContext and JSONPath data flow, FromHttpRequest endpoints and response shaping, entity CRUD via read nodes plus CreateUpdateInfo/ApplyChanges, cross-pipeline calls, and the deploy/debug iteration loop with a scratch DataFlow. Node configuration is looked up live on docs.meshmakers.cloud via a fixed URL pattern instead of memorized. Trigger on - OctoMesh pipeline, pipeline YAML, pipeline node, DataFlow, ETL on OctoMesh, HTTP endpoint on OctoMesh, REST API without a backend, Mesh Adapter, FromHttpRequest, DataContext, JSONPath in pipelines, ForEach, CreateUpdateInfo, ApplyChanges, GetRtEntitiesByType, ToPipelineDataEvent, awaitResult, ImportRt, DeployDataFlow, DeployPipeline, OctoMesh pipeline fails or debugging, pipeline node configuration, pipeline node docs.
---

# Pipeline API — Declarative Backends on the Mesh Adapter

## The model

A **DataFlow** groups **Pipelines**; each pipeline is YAML — a trigger node plus a list of `NodeName@version` steps operating on a single JSON **DataContext** addressed via JSONPath. The tenant's Mesh Adapter executes them and hosts HTTP routes itself at `/{tenantId}{route}`; the final DataContext (or an explicit projection) is the HTTP response. A whole app backend can be nothing but pipelines.

Fundamentals (YAML shape, DataContext write modifiers, trigger context placement): `references/pipeline-fundamentals.md`.

## Golden rules

1. **The YAML deserializer is strict.** Unknown keys are deploy-time errors; node config shapes vary per node and version. Therefore: **fetch the node's doc page before configuring an unfamiliar node** —
   `https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/<category>/<camelCaseNodeName>_<version>` (categories: `trigger`, `extract`, `transformation`, `load`, `control`). When a doc page and reality disagree, the adapter-generated schema is authoritative: `octo-cli -c GetPipelineSchema`.
2. **Deploy via `DeployDataFlow`, not per-pipeline `DeployPipeline`** — the latter enables debug mode, which breaks cross-pipeline `awaitResult` calls.
3. **Never `ImportRt -r` over a live DataFlow** — it poisons pipeline registration. Recovery and the safe scratch-DataFlow iteration loop: `references/deploy-and-debug.md`.
4. **Inside `ForEach`, absolute targetPaths do not reach the root context.** Accumulate per-item results via the `mergePath`/`targetPath` child-subtree mechanism (`references/entity-crud-patterns.md`).

## Building an HTTP API

Route → `FromHttpRequest@1` trigger; request lands in the DataContext (`$.body`, `$.query`, `$.path`, `$.method`). Shape responses explicitly; return errors as HTTP-200 payloads with an `error` field, or the client cannot distinguish them from adapter failures. Cross-pipeline composition uses `ToPipelineDataEvent@1` with `awaitResult`/`resultTargetPath`. Details, route-collision recovery, and the in-cluster service address: `references/http-api-patterns.md`.

## Entity CRUD and the wire contract

Read with `GetRtEntitiesByType@1`/`GetRtEntitiesById@1` (field filters), write with `CreateUpdateInfo@1` → `ApplyChanges` (+ `CreateAssociationUpdate@1` for edges). Clients must handle: enums are **written as member names, read back as integer keys**; date-looking strings are normalized to DateTime on body parse; **writing an empty array nulls the attribute**. Full patterns: `references/entity-crud-patterns.md`.

## Iterating

Work on a scratch DataFlow imported with `ImportRt`, deploy with `DeployDataFlow`, verify with `curl`, inspect with `GetDataFlowStatus`/pipeline debug data. Loop, failure-mode table, and all verified CLI flags: `references/deploy-and-debug.md`.

## Additional resources

### Reference files

- **`references/pipeline-fundamentals.md`** — entities, YAML shape, DataContext, JSONPath, node-doc URL pattern
- **`references/http-api-patterns.md`** — HTTP hosting, request/response shaping, cross-pipeline calls, route pitfalls
- **`references/entity-crud-patterns.md`** — read/write/association nodes, ForEach accumulation, wire-contract gotchas, Math@1
- **`references/deploy-and-debug.md`** — scratch-DataFlow loop, deploy commands, debugging, failure modes

### Examples (neutral TaskBoard domain, valid YAML)

- **`examples/taskboard-scratch-dataflow.yaml`** — complete ImportRt scratch file with a GET state-aggregation pipeline
- **`examples/post-create-task.pipeline.yaml`** — POST body → CreateUpdateInfo → ApplyChanges → response
- **`examples/foreach-batch-delete.pipeline.yaml`** — ForEach accumulation done right
