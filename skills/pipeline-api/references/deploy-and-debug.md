# Deploy and Debug Pipelines

The iteration loop for pipeline development: import a scratch DataFlow, deploy, execute, inspect, fix, repeat. All commands are `octo-cli`; per-command reference pages live under <https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/> (deploy/status/debug) and `.../asset-repository-services/` (ImportRt).

## The iteration loop

1. Keep a **scratch DataFlow file** — one ImportRt YAML containing the DataFlow entity plus Pipeline entities with the definition embedded in `System.Communication/PipelineDefinition-1` (see `../examples/taskboard-scratch-dataflow.yaml` for the complete shape). Use scratch rtIds distinct from the ones a later packaged install will use, so both can coexist in different tenants.
2. Import the entities:

   ```bash
   octo-cli -c ImportRt -f test/scratch-dataflow.yaml -w
   ```

3. Deploy the whole DataFlow:

   ```bash
   octo-cli -c DeployDataFlow -id <dataFlowRtId>
   ```

4. Exercise it (curl for HTTP pipelines, `ExecutePipeline` otherwise) and inspect results.
5. Edit the YAML, re-import with `-r`, and redeploy — but read the `-r` caveat below first.

Find the adapter and its pipelines when ids are unknown: `octo-cli -c GetAdapters -j`. On communication-enabled tenants the Mesh Adapter has the fixed rtId `670000000000000000000002`.

## ImportRt and the `-r` re-registration caveat

`ImportRt -f <file> -w` schedules an import job and waits. Re-running the same file for **existing** entities requires `-r` (`--replace`) — without it, existing entities are not updated.

**Caveat:** `ImportRt -r` over a currently-deployed DataFlow can poison the pipeline registration state — subsequent deploys alternately remove changed and unchanged pipeline sets, and behavior becomes inconsistent. Recovery:

```bash
octo-cli -c UndeployDataFlow -id <dataFlowRtId>
octo-cli -c DeployDataFlow  -id <dataFlowRtId>
```

If that does not settle it, restart the Mesh Adapter (delete its pod) and deploy once. Making undeploy-before-reimport a habit avoids the problem entirely: `UndeployDataFlow` → `ImportRt -r` → `DeployDataFlow`. Fresh installs (new entities, first deploy) are unaffected.

## DeployDataFlow vs DeployPipeline — always deploy via DeployDataFlow

| Command | What it does | Debug flag |
|---|---|---|
| `DeployDataFlow -id <dataFlowRtId>` | Deploys every pipeline of the DataFlow from the stored `PipelineDefinition` attributes | deploys with debugging **disabled** |
| `DeployPipeline -aid <adapterRtId> -pid <pipelineRtId> -f <yaml>` | Pushes one YAML file to one pipeline | deploys with debugging **enabled** |

`DeployPipeline` is useful to push a single definition file quickly, but it deploys with per-node debug capture on — which changes runtime behavior (overhead, and it has broken cross-pipeline `awaitResult` calls in practice). The reliable routine: use `DeployDataFlow` for every normal deploy; if `DeployPipeline` was used to push YAML, follow up with `DeployDataFlow` to flip the debug flag back off. Toggle debugging deliberately via `SetPipelineDebug` instead (below).

## Status and execution

```bash
octo-cli -c GetDataFlowStatus -id <dataFlowRtId> -j     # aggregated state of all pipelines
octo-cli -c GetPipelineStatus -id <pipelineRtId> -j     # one pipeline's deployment state
octo-cli -c ExecutePipeline   -id <pipelineRtId>        # manual run (FromExecutePipelineCommand@1) → execution ID
octo-cli -c ExecutePipeline   -id <pipelineRtId> -f input.json   # with input data
octo-cli -c GetLatestPipelineExecution -id <pipelineRtId> -j     # status, duration, errors, OutputData
```

Interpreting execution status:

- **OutputData is only persisted when the pipeline contains `SetPipelineExecutionResult@1`** (`path:` selects what to store). Without that node, an empty OutputData is by design, not a failure.
- For HTTP-triggered runs, `Status: null` is ambiguous — execution tracking may not complete before the HTTP response returns, so null appears on successes too. **Verify by checking effects** (query the entities that should have changed, check the response payload), not by execution status alone.
- The adapter returns HTTP 200 for `FromHttpRequest` pipelines even when the run failed internally.

## Pipeline debugging

Enable per-node debug capture on a live pipeline — no redeploy needed:

```bash
octo-cli -c SetPipelineDebug -id <pipelineRtId> -e true    # enable (persisted if adapter offline)
octo-cli -c GetPipelineDebug -id <pipelineRtId> -j         # read current debug state
```

Then execute the pipeline and read the debug tree:

```bash
octo-cli -c GetPipelineDebugPoints -id <pipelineRtId> -eid <executionId> -j
```

The tree shows which nodes ran and the data at each point:

- Nodes **missing** from the tree never executed — the pipeline stopped before them.
- The **last node present** is usually where the error occurred.

Disable debug capture (`-e false`) when done — it adds overhead and interferes with cross-pipeline `awaitResult` calls.

## Verifying HTTP pipelines with curl

Port-forward the Mesh Adapter service (**local development cluster** example — adjust namespace/tenant):

```bash
kubectl port-forward -n octo svc/<tenantId>-670000000000000000000002 5020:80

curl http://localhost:5020/<tenantId>/state
curl -X POST http://localhost:5020/<tenantId>/tasks \
     -H "Content-Type: application/json" \
     -d '{"title":"Water the plants","points":2}'
```

Check both the response payload (including `error` fields — see `http-api-patterns.md`) and, for writes, the stored entities.

## Common failure modes and recovery

| Symptom | Cause | Fix |
|---|---|---|
| Deploy fails: `Property 'x' not found …` | Unknown YAML key — strict deserializer | Fix the property name; confirm against `GetPipelineSchema` |
| Deploy fails: `Route '/x' already exists` | Second pipeline registering the same route, or a leaked registration from an earlier failed deploy | Remove the duplicate route; for leaks, restart the Mesh Adapter pod, then deploy once |
| Deploys alternate between removing/adding pipelines | `ImportRt -r` over a live DataFlow | `UndeployDataFlow` + `DeployDataFlow`; adapter restart if needed |
| Apply succeeds but nothing is written | `ApplyChanges@2` path empty/wrong — it silently no-ops | Check `entityUpdatesPath` spelling and that updates were accumulated (ForEach merge!) |
| Write fails: `Inbound association 'X' has minimum multiplicity of 'One'` | Entity type requires a mandatory association | Add `CreateAssociationUpdate@1` to the same `ApplyChanges@2` batch |
| `Status: null` on an HTTP-triggered run | Tracking race — can be success or failure | Verify effects; enable debug and re-run |
| `awaitResult` calls time out | Target adapter offline, or debugging enabled on the target | Bring the adapter online; redeploy via `DeployDataFlow` (debug off) |

## Discovering what the adapter supports

```bash
octo-cli -c GetPipelineSchema -aid <adapterRtId> -o pipeline-schema.json
```

Returns the JSON Schema of every node the adapter supports — trigger and transformation nodes with required keys, property names, and enum values. Validate YAML against it before deploying, and prefer it over any doc page when they disagree.

## Cron-scheduled pipelines

For scheduled execution, add a `System.Communication/PipelineTrigger` entity (cron expression + `Triggers` association) to the DataFlow, give the target pipeline a `FromPipelineTriggerEvent@1` trigger, and activate with `octo-cli -c DeployTriggers`. Entity wiring details: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/pipelineTriggers/intro>.

Cron format (verified against the platform's scheduler stack, 2026-08-19): standard **5-field** cron (`minute hour day-of-month month day-of-week`); an expression with **6 fields is interpreted as having a leading seconds field**. There is no trailing year field.
