# Tenant Prerequisites

What must be true before an app blueprint can be installed on a target
tenant, and the one-time setup a developer's machine needs to get there.
Run through this once per new tenant (and once per developer machine); the
[migration playbook](migration-playbook.md) assumes it is already done.

## Developer machine setup (once per machine)

### Point `octo-cli` at the environment

Configure a named context with the target environment's service URLs and
default tenant. `AddContext` creates or updates a context by name without
touching whichever context is currently active:

```powershell
octo-cli -c AddContext -n <name> `
    -isu "https://<identityHost>/" `
    -asu "https://<assetRepoHost>/" `
    -bsu "https://<botHost>/" `
    -csu "https://<communicationHost>/" `
    -tid "<tenantId>"
```

| Short | Long | Required | Description |
|---|---|---|---|
| `-n` | `--name` | yes | Name of the context (e.g. `dev`, `prod`) |
| `-isu` | `--identityServicesUri` | no | Identity services URI |
| `-asu` | `--assetServicesUri` | no | Asset repository services URI |
| `-bsu` | `--botServicesUri` | no | Bot services URI |
| `-csu` | `--communicationServicesUri` | no | Communication services URI |
| `-rsu` | `--reportingServicesUri` | no | Reporting services URI |
| `-aisu` | `--aiServicesUri` | no | AI services URI |
| `-tid` | `--tenantId` | no | Default tenant id |

Switch between configured contexts with `octo-cli -c UseContext -n <name>`,
and list them with `octo-cli -c ListContexts`. A single active context can
also be configured directly with `octo-cli -c Config -isu ... -asu ... -tid ...`
(same URI flags, `-isu` required) if only one environment is ever in use.

Reference: [Config](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/Config), [AddContext](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/context-management/AddContext), [UseContext](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/context-management/UseContext), [ListContexts](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/context-management/ListContexts).

### Log in

```powershell
octo-cli -c LogIn -i
```

`-i` (`--interactive`) opens a browser for the OAuth2 device-code flow. For
setup scripts that re-run often, combine with `-in` (`--if-needed`): it
first tries a silent token refresh and only falls back to opening the
browser when that fails.

Verify the active token with `octo-cli -c AuthStatus` — it reports the JWT
claims of the current access token and which auth method produced it.

Reference: [LogIn](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/LogIn), [AuthStatus](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/AuthStatus).

## Tenant communication readiness (once per tenant)

An app that ships HTTP pipelines (architecture A) or an operator-deployed
web app needs communication enabled on the tenant, with its pool and
adapter actually running before anything can be installed against them.

### 1. Enable communication

```powershell
octo-cli -c EnableCommunication
```

Enables the communication controller for the current tenant. This seeds a
fixed set of runtime entities every communication-enabled tenant gets, at
the same runtime ids on every tenant:

| Entity | rtId |
|---|---|
| Pool | `670000000000000000000001` |
| Mesh Adapter | `670000000000000000000002` |
| Helm chart repository | `670000000000000000000003` |

A blueprint's seed data associates its pipelines to the Mesh Adapter and its
Application entity to the Pool and Helm chart repository using these fixed
ids — no lookup is needed at install time.

Reference: [EnableCommunication](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/EnableCommunication).

### 2. Deploy the pool

```powershell
octo-cli -c DeployPool -id 670000000000000000000001
```

Triggers the Communication Operator to create the pool's Kubernetes
resources. This does **not** deploy the pool's workloads — the Mesh Adapter
still needs its own deploy.

Reference: [DeployPool](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/DeployPool).

### 3. Deploy the Mesh Adapter workload

```powershell
octo-cli -c DeployWorkload -id 670000000000000000000002
```

Triggers a deploy of the Mesh Adapter through its parent pool. Wait for it
to come online before installing an app blueprint against it — see the
verification checks below.

Reference: [DeployWorkload](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/DeployWorkload).

### Verify tenant readiness

```powershell
octo-cli -c GetPools -j
octo-cli -c GetAdapters -j
```

Both a Pool and an Adapter carry a `CommunicationState` field
(`Unregistered` / `Online` / `Offline`); the Adapter additionally carries a
`DeploymentState` (`Undeployed` / `Pending` / `Deployed` / `Error`) and a
`ConfigurationState`. Confirm the Mesh Adapter shows `DeploymentState:
Deployed` and `CommunicationState: Online` before installing an app
blueprint that depends on it.

Reference: [GetPools](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/GetPools), [GetAdapters](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/GetAdapters), [Pools overview](https://docs.meshmakers.cloud/docs/technologyGuide/communication/pools/intro), [Adapters overview](https://docs.meshmakers.cloud/docs/technologyGuide/communication/adapters/intro).

After a blueprint installs a DataFlow (architecture A), check its rollout
the same way:

```powershell
octo-cli -c GetDataFlowStatus -id <dataFlowRtId> -j
```

Returns the aggregated execution status of the data flow and its pipelines.

Reference: [GetDataFlowStatus](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/communication-services/GetDataFlowStatus).

### Self-hosted clusters: the operator itself is a precondition

`EnableCommunication` and `DeployPool` assume a Communication Operator is
already installed on the target Kubernetes cluster and configured (via its
own environment variables / Helm values) with the Communication Controller
URI and the message-broker connection it should use. On a managed OctoMesh
environment this is already in place. On a self-hosted cluster, install the
operator first — see the platform's
[Pool installation guide](https://docs.meshmakers.cloud/docs/technologyGuide/communication/pools/installation)
for what the operator needs before a Pool can register.

## Glossary of platform moving parts

- **Asset Repository Service** — organizes, describes and serves the
  tenant's data products, including the generated GraphQL API.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/techTerms#services))
- **Identity Service** — centralized authentication and authorization;
  integrates with multiple identity providers.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/techTerms#services))
- **Communication Controller Service** — central hub for managing and
  securing communications across adapters and applications; regulates
  message flow and enables scalable, secure connectivity.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/techTerms#services))
- **Mesh Adapter** — an adapter installed centrally (as opposed to an Edge
  Adapter installed on-premises); executes pipelines and, for HTTP-triggered
  pipelines, hosts the HTTP server those routes are registered on.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/apiTerms#adapters))
- **Communication Operator** — installed on a Kubernetes cluster; manages
  the lifecycle of a Pool's workloads (Adapters and Applications) by running
  `helm upgrade --install` for each.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/techTerms#communication))
- **Pool** — a collection of workloads (Adapters and Applications) managed
  together by one Communication Operator, usually in its own Kubernetes
  namespace.
  ([reference](https://docs.meshmakers.cloud/docs/technologyGuide/communication/pools/intro))
- **DataFlow** — a logical grouping of related pipeline definitions;
  organizes the ETL processes and establishes the shared exchange pipelines
  in the same DataFlow use to call each other.
  ([reference](https://docs.meshmakers.cloud/docs/glossary/apiTerms#pipelines))
