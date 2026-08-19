---
name: mesh-data-layer
description: Use OctoMesh as the data layer for an application that keeps its own backend - the per-tenant GraphQL API generated from the tenant's CK models, typed vs generic runtime queries and mutations, filtering/paging/association navigation, machine-to-machine auth with client-credentials clients against the identity service, and the wire conventions a client must handle (enums written as names and read as integer keys, date coercion, null semantics). Trigger on - OctoMesh GraphQL, tenant GraphQL endpoint, query mesh entities from my backend, runtime entities API, OctoMesh as data layer, backend integration with OctoMesh, service-to-service auth, client credentials client, API client for OctoMesh, bearer token for the mesh API, identity service, scopes, acr_values, tenant API URL, GraphQL playground, schema introspection for codegen, SDL.
---

# Mesh Data Layer — GraphQL from an Existing Backend

## The endpoint

Each tenant exposes a GraphQL API generated from its CK models on the asset-repository service:

```
https://{asset-repo-host}/tenants/{tenantId}/graphql              # API
https://{asset-repo-host}/tenants/{tenantId}/graphql/playground   # interactive explorer
```

Importing a CK model is what creates the API — there is no separate API layer to build. Two surfaces exist: the **typed** API (fields generated per CK type) and the **generic** runtime API (works for any type via `ckTypeId` strings). Query/mutation shapes, filters, paging, association navigation, and the full CK-type→GraphQL wire table: `references/graphql-api.md`.

## Auth in three steps (machine-to-machine)

1. Register a client-credentials client and grant scopes: `AddClientCredentialsClient`, `AddScopeToClient` (details and roles: `references/auth-and-clients.md`).
2. Request a token from the identity service `/connect/token` — **include `acr_values=tenant:{tenantId}`**; omitting it is the classic `invalid_client` failure.
3. Send `Authorization: Bearer <token>` on every GraphQL request.

For developer tooling, use octo-cli's interactive Device Code login instead (`LogIn`, `AuthStatus`).

## Wire conventions the client must own

Centralize these in one API module instead of scattering them:

- Enums: **write member names, read integer keys** on the typed API; the generic API returns ints by default (`resolveEnumValuesToNames` opts into names).
- Deletes take the **unversioned** CK type id (`Vendor.Domain/Type`, not `…/Type-1`).
- GraphQL errors arrive as HTTP 200 + `errors[]` — distinguish them from a genuine 401.
- Date-looking strings are coerced to DateTime server-side; normalize on read.

## Additional resources

### Reference files

- **`references/graphql-api.md`** — endpoint, typed vs generic API, queries, mutations, filters, associations, wire-format table, introspection/codegen
- **`references/auth-and-clients.md`** — client registration, token flow with acr_values, device-code login, CLI contexts, multi-tenant URLs

### Examples

- **`examples/queries.graphql`** — commented typed + generic operations (query, association navigation, create/update/delete) on a placeholder CK type; regenerate field names from the reader's own schema
- **`examples/backend-integration.md`** — curl-only walkthrough: token → query → mutation → error handling
