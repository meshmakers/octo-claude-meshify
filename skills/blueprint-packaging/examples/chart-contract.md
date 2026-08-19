# Web-App Chart Contract

The contract a Helm chart must fulfil to serve as an operator-deployed OctoMesh
Application workload for a "static SPA + thin API proxy" web app. Distilled from
working OctoMesh app charts; scaffold with `helm create` and adjust to match.

## Values the Chart Must Expose

| Value | Default | Purpose |
|---|---|---|
| `upstreamUrl` | `""` (mandatory) | Tenant-scoped Mesh Adapter base URL the app proxies to. Fail the render when unset — a proxy without an upstream is always a misconfiguration. |
| `publicUri` | `""` | Public host the app is exposed on. The Communication Operator injects the resolved value at deploy time (from the workload's `IngressEnabled` + `Hostname`). |
| `service.port` | `5055` | The port the app listens on; wired through as both `PORT` env and `containerPort`. |
| `image.repository` | your image name | Image repository (no registry prefix). |
| `image.privateRegistry` | `""` | Registry prefix; the operator overrides it at deploy time on clusters with a private registry. |
| `image.tag` | `""` | Empty = default to `Chart.AppVersion`. Set `appVersion` to the matching image build at chart publish time; never pin tags in blueprints. |
| `image.pullPolicy` | `IfNotPresent` | Also what makes node-local images work on a local development cluster. |
| `ingress.*` | disabled | Standard ingress block; the operator drives `ingress.enabled` + host from the workload entity. |

## What the Deployment Template Must Render

```yaml
containers:
  - name: {{ .Chart.Name }}
    # registry-aware image reference:
    #   privateRegistry set -> "<privateRegistry>/<repository>:<tag|appVersion>"
    #   otherwise          -> "<repository>:<tag|appVersion>"
    ports:
      - name: http
        containerPort: {{ .Values.service.port }}
        protocol: TCP
    env:
      - name: PORT
        value: {{ .Values.service.port | quote }}
      - name: UPSTREAM_URL
        value: {{ .Values.upstreamUrl | quote }}
    livenessProbe:
      httpGet: { path: /, port: http }
    readinessProbe:
      httpGet: { path: /, port: http }
```

## Contract Summary (what the image must do)

| Requirement | Detail |
|---|---|
| Listen on `PORT` | HTTP server binds the port from the env var (default 5055). |
| Proxy to `UPSTREAM_URL` | `/api/*` requests forward to the tenant-scoped Mesh Adapter base, preserving method, query, and JSON body. |
| `GET /` returns 200 | Serves the SPA; this is what both probes hit. |
| Single HTTP port | One container port, named `http`, equal to `service.port`. |

Any image fulfilling this contract can reuse an existing compatible chart via
image override (`ValuesYaml`: `image.repository` + `upstreamUrl`) — no chart
authoring needed for a first deploy.
