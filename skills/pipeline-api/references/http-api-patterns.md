# HTTP API Patterns

Build an HTTP API entirely from pipelines: the Mesh Adapter hosts the HTTP endpoints, one pipeline per route. No service code is required — a complete CRUD backend can be declarative pipeline YAML.

## The Mesh Adapter is the HTTP host

Pipelines triggered by `FromHttpRequest@1` register their route on the **Mesh Adapter's own HTTP server** — not on the Communication Controller and not on the Asset Repository. The URL shape is:

```
<mesh-adapter-endpoint>/{tenantId}{route}
```

- In a Kubernetes install, the adapter's HTTP server is exposed by the service `<tenantId>-670000000000000000000002` (the platform-seeded Mesh Adapter workload) on port 80, plain HTTP, in the OctoMesh namespace.
- Other clients inside the cluster reach it at `http://<tenantId>-670000000000000000000002.<namespace>.svc.cluster.local:80/<tenantId>`.
- For testing from a workstation against a **local development cluster**, port-forward and curl:

```bash
kubectl port-forward -n octo svc/<tenantId>-670000000000000000000002 5020:80
curl http://localhost:5020/<tenantId>/state
```

(Adjust `-n octo` to the namespace OctoMesh is installed in. A host-process adapter install may instead serve HTTPS on port 5020 with a self-signed certificate — use `curl -sk` there.)

## FromHttpRequest@1 — the route trigger

Node page: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/trigger/fromHttpRequest_1>

```yaml
triggers:
  - type: FromHttpRequest@1
    description: HTTP POST /tasks
    method: Post          # Get | Post | Put | Delete
    path: /tasks          # route below /{tenantId}
```

The trigger seeds the DataContext with the request:

| Path | Content |
|---|---|
| `$.body` | Parsed JSON request body (POST/PUT) |
| `$.query` | Query-string parameters (`GET /tasks?id=…` → `$.query.id`) |
| `$.path`, `$.method` | Route and HTTP method |
| `$.files`, `$.formData` | Uploads / form fields, when present |

Convention from working APIs: GET/DELETE endpoints read parameters from `$.query`; POST/PUT endpoints read `$.body`.

## The response is the final DataContext

When the pipeline finishes, the **entire remaining DataContext is serialized as the HTTP response body**. Left alone, that includes the request echo (`$.body`, `$.query`, `$.method`, …) and every intermediate value. Shape the response explicitly as the last step with `Project@1`:

```yaml
- type: Project@1
  description: Response = only these fields
  clear: true
  fields:
    - path: $.household
      inclusion: true
    - path: $.tasks
      inclusion: true
```

For scalar responses, set the value first, then project it:

```yaml
- type: SetPrimitiveValue@1
  valuePath: $.updates[0].RtId      # e.g. return the new entity id
  valueType: String
  targetPath: $.id
- type: Project@1
  clear: true
  fields:
    - path: $.id
      inclusion: true
```

## Return errors as payloads

The adapter returns **HTTP 200 even when the pipeline logic takes an error branch** — status-code-based error handling is not available from YAML. The proven pattern is an error field in the payload:

```yaml
- type: If@1
  description: Unknown id
  path: $.lookup.TotalCount
  operator: Equal
  value: 0
  valueType: Int
  transformations:
    - type: SetPrimitiveValue@1
      value: not_found
      valueType: String
      targetPath: $.error
# ... happy path guarded by the inverse If ...
- type: Project@1
  clear: true
  fields:
    - path: $.result
      inclusion: true
    - path: $.error
      inclusion: true
```

Clients check for the `error` key. `Project@1` omits absent paths, so the field only appears when set. Two caveats:

- An **unhandled node exception** (bad path, adapter-side failure) returns HTTP 500 with a raw exception page — guard inputs if the API is exposed beyond development.
- Because failures still return 200/500 without structured status, verify behavior by checking the data written, not just the HTTP status (see `deploy-and-debug.md`).

## Route registration pitfalls

- **Routes are unique per tenant-adapter, not per DataFlow.** Two deployed pipelines registering the same `path` collide: the second deploy fails with `Route '/x' already exists`. Never install two copies of the same HTTP API (e.g. a scratch DataFlow and a packaged install) in one tenant at the same time.
- **Failed deploys can leak routes.** If a route registers and a later part of the deployment fails, the route stays registered in adapter memory; subsequent deploys keep failing with `Route '/x' already exists` even after undeploying. Recovery: restart the Mesh Adapter (delete its pod so it is recreated), then deploy once, cleanly.

## Cross-pipeline calls: ToPipelineDataEvent@1 with awaitResult

Node page: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/load/toPipelineDataEvent_1>

Within one DataFlow, a pipeline can call another pipeline — even one running on a **different adapter** — and wait for its result. This is how an HTTP endpoint on the Mesh Adapter can delegate work to an edge adapter:

```yaml
# Caller (HTTP pipeline on the Mesh Adapter)
transformations:
  - type: ToPipelineDataEvent@1
    description: Delegate to the worker pipeline and wait
    path: $.query                     # what to send
    targetPath: $.request             # where it lands in the receiver's context
    targetValueWriteMode: Overwrite
    targetPipelineRtId: aaa000000000000000000007
    awaitResult: true
    timeoutSeconds: 30
    resultTargetPath: $               # replace own context with the worker's result

# Callee (worker pipeline, any adapter in the same DataFlow)
triggers:
  - type: FromPipelineDataEvent@1
transformations:
  - ...                               # its final DataContext becomes the caller's result
```

Semantics:

- `awaitResult: true` blocks the caller until the target pipeline finishes; the target's final DataContext is written at `resultTargetPath` (default `$.pipelineResult`). `resultTargetPath: $` replaces the caller's context entirely — combined with an HTTP trigger, the worker's output becomes the HTTP response.
- `targetPipelineRtId` is required; both pipelines must belong to the **same DataFlow**, and the target must trigger on `FromPipelineDataEvent@1`.
- If the target pipeline fails, the error propagates: the caller's execution fails and its remaining nodes do not run.
- If the target does not respond within `timeoutSeconds`, the caller fails with a timeout (an offline target adapter surfaces as a distribution timeout).
- Without `awaitResult`, the node is fire-and-forget pub/sub on the DataFlow's topic exchange.

## FromExecutePipelineCommand@1 — one per DataFlow

`FromExecutePipelineCommand@1` (manual execution via `octo-cli -c ExecutePipeline`) is registered **per DataFlow, not per pipeline** — only one pipeline in a DataFlow can carry this trigger. Do not add it to every HTTP pipeline "for testability"; the registrations collide. Test HTTP pipelines with curl instead, and reserve `FromExecutePipelineCommand@1` for at most one utility pipeline per DataFlow.
