# Web App Workloads — the Application Entity, Chart Contract, and Helm Rollout

Ship a web UI for a mesh app the platform-native way: an
`System.Communication/Application` entity in the blueprint seed, helm-installed by
the Communication Operator on `DeployWorkload`.

Why a workload and not the Mesh Adapter: the adapter serves `application/json` only —
it cannot serve HTML. The Application workload is a Helm release with an ingress and
a per-tenant hostname, lifecycle-managed via octo-cli or Studio.

Type and attribute reference:
<https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/libraries/System/Communication-3/Types>
(and the sibling `Attributes` / `Associations` / `Enums` pages).

## The Application Seed Entity

```yaml
- rtId: '0abc00000000000000000020'
  ckTypeId: System.Communication/Application-1
  associations:
    - roleId: System.Communication/Manages-1            # deploy target pool
      targetRtId: '670000000000000000000001'
      targetCkTypeId: System.Communication/Pool-1
    - roleId: System.Communication/HelmRepository-1     # chart source
      targetRtId: '670000000000000000000003'            # or your own seeded repo config
      targetCkTypeId: System.Communication/HelmRepositoryConfiguration-1
  attributes:
    - id: System/Name-1
      value: My App UI
    - id: System.Communication/ChartName-1
      value: my-app                     # chart name within the Helm repository
    - id: System.Communication/ChartVersion-1
      value: ""                         # empty = always pull the newest chart
    - id: System.Communication/DeploymentState-1
      value: 0                          # Undeployed — DeployWorkload triggers the rollout
    - id: System.Communication/ReceivesClusterSecrets-1
      value: false                      # app only talks HTTP to the adapter
    - id: System.Communication/IngressEnabled-1
      value: true
    - id: System.Communication/Hostname-1
      value: 'my-app-${octo.tenantId}.{{domain.default}}'
    - id: System.Communication/ValuesYaml-1
      value: |
        upstreamUrl: http://${octo.tenantId}-670000000000000000000002.octo.svc.cluster.local:80/${octo.tenantId}
```

Attribute semantics (all verified against the `System.Communication-3` library docs):

| Attribute | Semantics |
|---|---|
| `ChartName` | Helm chart name within the associated repository. |
| `ChartVersion` | SemVer, or empty = track the repository's newest chart version. Runtime-state: preserved across blueprint re-applies. |
| `IngressEnabled` | When true, the operator projects `ingress.enabled=true` plus the resolved `Hostname` into the Helm values. Ingress class, cluster issuer, and TLS come from cluster-wide operator defaults — not per workload. |
| `Hostname` | Public host when `IngressEnabled=true`. Applications are served at the root path. Runtime-state. |
| `ValuesYaml` | Full `values.yaml` content as a string — the base layer of Helm values. Runtime-state. |
| `Values` | Structured value overrides, deep-merged **on top of** `ValuesYaml`; entries flagged secret are stored encrypted. Operator-managed entries (e.g. `publicUri`) land here. |
| `ReceivesClusterSecrets` | When true, the operator injects cluster-internal DB credentials as secret value overrides. Keep `false` for a UI that only proxies HTTP to the adapter. |
| `LastDeploymentError` / `LastDeploymentErrorTimestamp` | Set when `DeploymentState` transitions to Error; cleared on the next successful deploy. First stop for triage. |

## Hostname Templating — One Seed, Every Cluster

`Hostname: 'my-app-${octo.tenantId}.{{domain.default}}'` combines both variable
systems:

- `${octo.tenantId}` resolves at **blueprint apply** — the entity lands with the
  tenant baked in.
- `{{domain.default}}` stays verbatim and resolves at **deploy time** against the
  communication controller's per-cluster domain configuration. The operator then
  emits the resolved `publicUri` into the chart values.

The result: the same seed yields the right public host on every cluster. On a local
development cluster the default domain is typically `127.0.0.1.nip.io`, so the app
answers at `https://my-app-<tenant>.127.0.0.1.nip.io`; on a managed cluster it is
that cluster's ingress domain. If the target cluster has no domain configured for
the placeholder, the deploy fails with `WorkloadHostnameUnknownDomain` — fall back
to an explicit `publicUri` in `ValuesYaml` there.

## Values Layering

Effective Helm values at deploy, lowest to highest priority:

1. chart `values.yaml` defaults
2. the workload's `ValuesYaml` (base layer, from the seed)
3. structured `Values` overrides (deep-merged; operator-managed entries live here)
4. operator-injected values: private registry, `ingress.enabled` + resolved
   `publicUri` (from `IngressEnabled`/`Hostname`), cluster secrets when
   `ReceivesClusterSecrets` is true

Keep the seed's `ValuesYaml` minimal — typically only the tenant-scoped
`upstreamUrl`. Never pin `image.tag` in a blueprint: the chart's `appVersion` is the
tag default, and CI should set `appVersion` to the matching image build at chart
publish time so chart and image always travel together.

## The Chart Contract

Full details: [examples/chart-contract.md](../examples/chart-contract.md). Summary of
what a compatible web-app chart provides (distilled from working OctoMesh app
charts):

- container env `PORT` (from `service.port`, default 5055) and `UPSTREAM_URL`
  (from the mandatory `upstreamUrl` value)
- one HTTP container port named `http`, equal to `service.port`
- liveness + readiness probes: `httpGet` on `GET /` against port `http`
- image reference built as `[privateRegistry/]repository:tag`, with `tag`
  defaulting to `Chart.AppVersion`
- `publicUri` value driving the ingress host

## The Thin Node Proxy Pattern

The proven app shape behind that contract: a zero-dependency Node HTTP server that

- serves the built SPA at `GET /` (also satisfies the probes) and static assets
  from the client build directory, falling back to `index.html` for SPA routes;
- proxies `/api/<rest>` (any method) to `UPSTREAM_URL/<rest>`, preserving query
  string and JSON body, with TLS verification off for https upstreams (local
  adapters use self-signed certificates);
- answers `502 {"error":"upstream_unreachable"}` when the upstream is down.

The browser only ever talks same-origin to the app — no CORS, no certificate
issues — while the proxy reaches the Mesh Adapter in-cluster over plain HTTP. The
tenant-scoped upstream is the adapter's in-cluster service:

```
http://<tenantId>-670000000000000000000002.<namespace>.svc.cluster.local:80/<tenantId>
```

(`<namespace>` is the namespace the operator deploys workloads into, `octo` by
default; the adapter's routes are prefixed with the tenant id.)

Build the image with a multi-stage Dockerfile — SPA build stage, then a slim runtime
stage with only the server and the built client:
[examples/Dockerfile](../examples/Dockerfile).

## Images and Registries

The communication controller can inject `image.privateRegistry` at deploy time when
the cluster is configured with a private registry — the final image reference then
becomes `<registry>/<repository>:<tag>`. Push your image to a registry the target
cluster can pull from, under exactly the repository name the chart's
`image.repository` declares.

**Local development cluster only** (e.g. kind — no registry needed, images are
node-local with `pullPolicy: IfNotPresent`): load the image into the cluster under
**both** the plain name and the registry-prefixed name the controller would inject,
so the reference resolves either way. `ImagePullBackOff` right after a deploy almost
always means the pod's image reference (check `kubectl get pod … -o jsonpath=`) does
not match any name that was pushed/loaded.

## Deploy and Operate

```bash
octo-cli -c DeployWorkload -id <applicationRtId>          # operator helm-installs
octo-cli -c UndeployWorkload -id <applicationRtId> -y     # helm-uninstall (destructive)
octo-cli -c UpdateWorkloadChartVersion -id <rtId> -cv 1.2.3   # sets ChartVersion only —
                                                              # run DeployWorkload after
octo-cli -c GetWorkloadsByChart -cn my-app                # find workloads by chart name
```

Command pages:
`https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/<Command>`.

### Changing an Application's Chart Requires Undeploy + Fresh Deploy

A plain redeploy runs `helm upgrade` on the same release. Kubernetes resource names
embed the chart name, so switching `ChartName` makes the new chart's ingress collide
with the old one on the same host (the ingress admission webhook rejects it) and the
atomic rollback can strand orphaned resources. Always:

```bash
octo-cli -c UndeployWorkload -id <rtId> -y
# change ChartName / HelmRepository association (new blueprint version or Studio)
octo-cli -c DeployWorkload -id <rtId>
```

If a failed upgrade already stranded resources on your own cluster, clean up with
`kubectl delete deploy,svc,ingress,sa -l app.kubernetes.io/instance=<release>`.

### Status and Error Inspection

- The workload entity itself is the primary status surface: `DeploymentState`
  (Undeployed/Pending/Deployed/Error/Disabled), `LastDeploymentError`,
  `LastDeploymentErrorTimestamp`, `StatusMessage` — readable in Refinery Studio or
  via GraphQL. This works on managed environments **without any kubectl access**.
- On clusters you operate yourself, add: pod state (`kubectl get pods`), probe
  failures (pod restarts / not Ready almost always mean `GET /` is not returning
  200 on `PORT`), and the ingress host (`kubectl get ingress`).
- Pending forever usually means the pool/operator is not running or the pool was
  never deployed (`DeployPool`).
