# Auth and Clients

OctoMesh's Identity Service is a Duende IdentityServer instance that issues OAuth 2.0 / OpenID Connect tokens. Every API call — GraphQL included — needs a bearer token scoped to a tenant. This reference covers the machine-to-machine path an external backend uses to talk to the asset repository's GraphQL API, plus the interactive path for developer tooling.

## Client types

OctoMesh clients are registered **per tenant** and come in three types:

| Type | Use case | Flow |
|---|---|---|
| **Client Credentials** | A backend service calling the API with no user involved | `grant_type=client_credentials` |
| **Device Code** | CLI tools / headless devices where a user authenticates on a separate browser | Device authorization flow |
| **Authorization Code (+ PKCE)** | Browser-based web apps acting on behalf of a signed-in user | `grant_type=authorization_code` |

For a backend that uses OctoMesh purely as a data layer — no end-user login — **Client Credentials** is the one to register. Device Code matters for developer tooling (`octo-cli` itself is a built-in Device Code client). Authorization Code matters only when the application also needs user-level login against OctoMesh's Identity Service.

## Creating a Client Credentials client

```powershell
octo-cli -c AddClientCredentialsClient -id "my-backend" -n "My Backend Service" -s "MySecretKey123"
```

| Short | Long | Required | Description |
|---|---|---|---|
| `-id` | `--clientId` | yes | Client ID, must be unique within the tenant |
| `-n` | `--name` | yes | Display name |
| `-s` | `--secret` | yes | Client secret used for token requests |
| `-apic` | `--autoProvision` | no | Mirror this client into every new sub-tenant of the calling tenant automatically — useful for a service identity that needs to reach many tenants with one client ID/secret pair |

This must be run by an operator already authenticated against the target tenant (e.g. via `octo-cli -c LogIn -i`, see below).

### Granting scopes

A client can only request scopes it has been explicitly granted:

```powershell
octo-cli -c AddScopeToClient -id "my-backend" -n "octo_api"
```

| Scope | Access |
|---|---|
| `octo_api` | Full read/write access to all OctoMesh APIs |
| `octo_api.read_only` | Read-only access |

Grant `octo_api.read_only` instead of `octo_api` when the backend only ever queries data.

### Managing secrets

A client can hold multiple secrets (e.g. to rotate without downtime):

```powershell
# Add a new secret with an expiration date
octo-cli -c CreateApiSecretClient -cid "my-backend" -e "2027-12-31" -d "Production secret"

# List secrets
octo-cli -c GetApiSecretsClient -cid "my-backend"

# Delete a secret (by its SHA256 hash, from GetApiSecretsClient output)
octo-cli -c DeleteApiSecretClient -cid "my-backend" -s "<sha256-value>"
```

Secrets are stored as SHA256 hashes server-side, so the plaintext secret is only ever known at creation time — capture it immediately.

### Roles for role-protected endpoints

A `client_credentials` token carries `role` claims resolved from the client's directly-assigned roles plus any roles inherited from group membership, in the same claim shape as a user token. This matters if the backend calls role-protected surfaces (for example an HTTP-triggered pipeline that checks the caller's role):

```powershell
octo-cli -c AddClientToRole -id "my-backend" -r "DataAnalyst"
```

Scopes gate *which API surface* a token can reach; roles gate *what it's authorized to do within that surface*. Most pure data-layer integrations only need a scope.

## Acquiring a token (Client Credentials flow)

```http
POST /connect/token HTTP/1.1
Host: {identity-host}
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=my-backend
&client_secret=MySecretKey123
&scope=octo_api
&acr_values=tenant:{tenantId}
```

Response:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "octo_api"
}
```

**`acr_values=tenant:{tenantId}` is required.** Clients are registered per tenant, and `/connect/token` itself carries no tenant path segment — this parameter is how the Identity Service knows which tenant's client store to search. Omit it and the lookup falls back to the system tenant, and the request fails with `invalid_client` for any tenant-registered client.

A `client_credentials` token has no `sub` claim (no user identity) and no `tenant_id` / `allowed_tenants` claims — the tenant binding is established once, at token-issuance time, by which tenant's client store matched. Downstream services skip the route-tenant authorization check for these tokens as a result.

## Using the token

```http
POST /tenants/{tenantId}/graphql HTTP/1.1
Host: {asset-repo-host}
Authorization: Bearer {access_token}
Content-Type: application/json

{ "query": "{ runtime { ... } }" }
```

Tokens expire (`expires_in`, typically 3600s). `client_credentials` issues no refresh token (per the OAuth2 spec) — request a fresh token with the same client credentials call once it expires, or slightly before, rather than trying to refresh.

## Developer tooling: interactive Device Code login

`octo-cli` itself authenticates as a built-in Device Code client. For a human operator working against the platform (setting up clients, running the commands above, exploring data), log in interactively once per environment:

```powershell
octo-cli -c LogIn -i
```

This opens a browser for device-code login. For scripts/setup routines that re-run often, use `-in` (`--if-needed`) alongside `-i`: it first checks whether the stored token is still valid or silently refreshable, and only falls back to opening a browser when both fail.

```powershell
octo-cli -c LogIn -i -in
```

For headless/CI use of `octo-cli` itself (not the backend's own token acquisition — a separate non-interactive login for the CLI), use `LogInClientCredentials` against a Client Credentials client instead:

```powershell
octo-cli -c LogInClientCredentials -id "my-script-client" -s "<secret>"
```

Credentials can also come from `OCTO_CLI_CLIENT_ID` / `OCTO_CLI_CLIENT_SECRET` environment variables (args take precedence when both are present). While those env vars stay set, `octo-cli` re-acquires the token automatically on expiry, so a long-running script does not need to call `LogInClientCredentials` again between commands.

Check the current auth state at any time:

```powershell
octo-cli -c AuthStatus
```

## Configuring `octo-cli` itself

Point the CLI at the environment's service URLs and default tenant before logging in:

```powershell
octo-cli -c Config -isu "https://{identity-host}/" -asu "https://{asset-repo-host}/" -tid "{tenantId}"
```

| Short | Long | Required | Description |
|---|---|---|---|
| `-isu` | `--identityServicesUri` | yes | Identity Service base URL |
| `-asu` | `--assetServicesUri` | no | Asset repository service base URL |
| `-bsu` | `--botServicesUri` | no | Bot service base URL |
| `-csu` | `--communicationServicesUri` | no | Communication Controller base URL |
| `-rsu` | `--reportingServicesUri` | no | Reporting service base URL |
| `-aisu` | `--aiServicesUri` | no | AI service base URL |
| `-tid` | `--tenantId` | no | Default tenant for subsequent commands |

To manage more than one environment or tenant without reconfiguring each time, use named contexts instead:

```powershell
octo-cli -c AddContext -n "prod" -isu "https://{identity-host}/" -asu "https://{asset-repo-host}/" -tid "{tenantId}"
octo-cli -c UseContext -n "prod"
```

`UseContext` with no `-n` lists all configured contexts.

## Multi-tenant URL structure

Every OctoMesh installation can host multiple tenants. The three things that carry tenant context, and how they relate:

1. **The GraphQL/REST URL path** — `/tenants/{tenantId}/graphql` addresses one tenant's data explicitly.
2. **The token's tenant binding** — for `client_credentials`, set once via `acr_values=tenant:{tenantId}` on the token request; the same token cannot be reused across tenants. A backend serving multiple tenants needs one Client Credentials client (and one token) per tenant, or must re-request a token when switching tenants.
3. **`octo-cli`'s active context** — decides which tenant CLI commands (`AddClientCredentialsClient`, `AddScopeToClient`, etc.) operate against. Switch context before creating or managing a client for a different tenant.

## See also

- <https://docs.meshmakers.cloud/docs/technologyGuide/identityService/clients-and-scopes>
- <https://docs.meshmakers.cloud/docs/technologyGuide/identityService/users-and-roles>
- <https://docs.meshmakers.cloud/docs/developerGuide/solutionArchitecture/apiIntegration>
- <https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/intro>
- octo-cli command reference: [AddClientCredentialsClient](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/identity-services/AddClientCredentialsClient), [AddScopeToClient](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/identity-services/AddScopeToClient), [CreateApiSecretClient](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/identity-services/CreateApiSecretClient), [LogIn](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/LogIn), [LogInClientCredentials](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/LogInClientCredentials), [Config](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/Config), [AuthStatus](https://docs.meshmakers.cloud/docs/technologyGuide/tools/octo-cli/command-reference/general/AuthStatus)

## Host and port placeholders

The exact hostnames, ports, and URL scheme an OctoMesh installation exposes for the identity and asset repository services depend on how that installation was deployed (managed cloud vs. self-hosted Helm install). This reference therefore uses `{identity-host}` / `{asset-repo-host}` placeholders throughout — obtain the concrete values from the environment's operator or its configuration.
