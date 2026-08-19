# Backend Integration Walkthrough (curl)

A minimal, tool-agnostic walkthrough for wiring an external backend against OctoMesh: acquire a token with a Client Credentials client, run a query, run a mutation. Every step uses plain `curl` so it translates directly into whatever HTTP client the backend already uses.

Prerequisites: a Client Credentials client already registered and scoped for the target tenant — see `../references/auth-and-clients.md` for the `octo-cli -c AddClientCredentialsClient` / `AddScopeToClient` setup.

Set the shared variables once:

```bash
IDENTITY_HOST="https://{identity-host}"
ASSET_REPO_HOST="https://{asset-repo-host}"
TENANT_ID="{tenantId}"
CLIENT_ID="my-backend"
CLIENT_SECRET="MySecretKey123"
```

## Step 1 — Get a token

```bash
TOKEN_RESPONSE=$(curl -s -X POST "$IDENTITY_HOST/connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" \
  --data-urlencode "scope=octo_api" \
  --data-urlencode "acr_values=tenant:$TENANT_ID")

ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')
```

`acr_values=tenant:$TENANT_ID` is required — without it the Identity Service cannot find a tenant-registered client and the call fails with `invalid_client`. The response also carries `expires_in` (seconds); request a fresh token again once it lapses — `client_credentials` issues no refresh token.

## Step 2 — Run a query

```bash
curl -s -X POST "$ASSET_REPO_HOST/tenants/$TENANT_ID/graphql" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query GetActiveMachines($first: Int!) { runtime { acmeAssetsMachine(first: $first, fieldFilter: [{attributePath: \"status\", operator: EQUALS, comparisonValue: \"ACTIVE\"}]) { items { rtId name status } pageInfo { hasNextPage endCursor totalCount } } } }",
    "variables": { "first": 20 }
  }' | jq .
```

Expected shape on success:

```json
{
  "data": {
    "runtime": {
      "acmeAssetsMachine": {
        "items": [{ "rtId": "693c5b93464d7d9e1396cf1c", "name": "Press Line 3", "status": "ACTIVE" }],
        "pageInfo": { "hasNextPage": false, "endCursor": "...", "totalCount": 1 }
      }
    }
  }
}
```

## Step 3 — Run a mutation

```bash
curl -s -X POST "$ASSET_REPO_HOST/tenants/$TENANT_ID/graphql" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation UpdateStatus($entities: [AcmeAssetsMachineInputUpdate]!) { runtime { acmeAssetsMachines { update(entities: $entities) { rtId status } } } }",
    "variables": {
      "entities": [{ "rtId": "693c5b93464d7d9e1396cf1c", "item": { "status": "MAINTENANCE" } }]
    }
  }' | jq .
```

## Handling errors

A failed GraphQL operation returns HTTP 200 with `data: null` and a populated `errors` array — check `errors`, not just the HTTP status code:

```bash
ERRORS=$(echo "$RESPONSE" | jq -c '.errors // empty')
if [ -n "$ERRORS" ]; then
  echo "GraphQL call failed: $ERRORS" >&2
  exit 1
fi
```

Branch on `errors[].extensions.code` (`ENTITY_NOT_FOUND`, `VALIDATION_ERROR`, `CK_TYPE_NOT_FOUND`, ...) rather than the human-readable `message`, which is not a stable contract.

A 401 response (rather than a GraphQL error body) means the token itself was rejected or has expired — go back to Step 1 and re-acquire it.

## Next steps

- `../references/graphql-api.md` — full query/mutation/wire-format reference.
- `../references/auth-and-clients.md` — client registration, scopes, and the Device Code flow for interactive `octo-cli` use.
- `queries.graphql` — more query/mutation shapes, including association navigation and the generic (type-agnostic) API.
