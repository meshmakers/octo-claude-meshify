# GraphQL API

OctoMesh exposes one GraphQL endpoint per tenant. It is served by the asset repository service and its schema is generated automatically from the tenant's installed Construction Kit (CK) models — every CK type, record, enum, and association becomes typed GraphQL fields the moment the model is imported. There is no separate step to "publish" an API: import or update the CK model, and the schema updates with it.

Use this reference to wire an existing backend against that endpoint as a data layer, independent of how the backend is otherwise built.

## Endpoint

```
https://{asset-repo-host}/tenants/{tenantId}/graphql
```

An interactive GraphQL Playground is served at the same path with `/playground` appended:

```
https://{asset-repo-host}/tenants/{tenantId}/graphql/playground
```

`{tenantId}` is the tenant whose data is being accessed; `{asset-repo-host}` is the asset repository service's public hostname for the environment. Every request needs a bearer token scoped to that tenant — see `auth-and-clients.md` for how to obtain one.

```http
POST /tenants/{tenantId}/graphql HTTP/1.1
Host: {asset-repo-host}
Authorization: Bearer {token}
Content-Type: application/json

{ "query": "{ runtime { ... } }" }
```

## Schema shape

The schema root has two query trees and one mutation tree:

```graphql
type Query {
  runtime: RuntimeQuery        # Master/config data, stored in MongoDB
  streamData: StreamQuery      # Time-series data, stored in CrateDB
  constructionKit: ConstructionKitQuery  # CK model introspection (types, attributes, enums, records)
}

type Mutation {
  runtime: RuntimeMutation
}
```

This reference covers `runtime` (the CRUD surface for master data). Time-series ingestion/query and CK model introspection are separate topics — the `constructionKit` tree allows enumerating a tenant's currently installed types/attributes/enums/records at runtime, useful when building a fully dynamic client.

## Typed vs. generic API

Every runtime entity type is reachable two ways:

| Approach | Shape | Use when |
|---|---|---|
| **Typed** | `runtime.<typeName>` (query), `runtime.<typeNamePlural> { create / update }` (mutation) | The CK type is known at build time — best IDE support, compile-time validation, and the shape client codegen tools produce. |
| **Generic** | `runtime.runtimeEntities` (query and mutation) | The CK type is only known at runtime, or the integration layer needs to stay type-agnostic. |

The typed field name is derived from the CK type's full name — camelCase, model namespace folded in, e.g. a type `EnergyCommunity/Customer` becomes query field `energyCommunityCustomer` and mutation field `energyCommunityCustomers { create / update }`. The exact plural/singular form is generated per type; when unsure, use the generic API or introspect the schema (see below).

## Querying

### List with pagination

```graphql
query {
  runtime {
    energyCommunityCustomer(first: 20, after: "cursor") {
      items {
        rtId
        ckTypeId
        contact { firstName lastName }
      }
      pageInfo {
        hasNextPage
        endCursor
        totalCount
      }
    }
  }
}
```

Always pass `first`; use `pageInfo.endCursor` as `after` for the next page.

### Generic query by CK type id

```graphql
query {
  runtime {
    runtimeEntities(ckId: "OctoSdkDemo/Customer", first: 10) {
      items {
        rtId
        ckTypeId
        rtWellKnownName
        rtCreationDateTime
        rtChangedDateTime
        attributes(first: 20) {
          items { attributeName value }
        }
      }
    }
  }
}
```

Every runtime entity — typed or generic — exposes these system fields: `rtId`, `ckTypeId`, `rtWellKnownName` (optional human-readable key), `rtDisplayName` / `rtDisplayDescription` (computed, read-only, never settable via API), `rtCreationDateTime`, `rtChangedDateTime`, `rtVersion`.

### Filtering

- **`rtId` / `rtIds`** — look up by identifier(s) directly.
- **`fieldFilter: [{attributePath, operator, comparisonValue}]`** — precise conditions on any attribute, combined with AND. Operators: `EQUALS`, `NOT_EQUALS`, `LESS_THAN`, `LESS_EQUAL_THAN`, `GREATER_THAN`, `GREATER_EQUAL_THAN`, `IN`, `NOT_IN`, `LIKE` (wildcard `*`), `MATCH_REG_EX`, `ANY_EQ` (array fields).
- **`searchFilter: {searchTerm, type, attributePaths | language}`** — free-text search. `type: ATTRIBUTE_FILTER` wildcard-matches specific attribute paths; `type: TEXT_SEARCH` uses a database full-text index (requires `indexType=Text` on the attribute in the CK model, plus a `language` code).
- **`sortOrder: [{attributePath, sortOrder}]`** — `ASCENDING`, `DESCENDING`, or `DEFAULT`. Default order is by `rtId`.
- **`options: { includeArchivedEntities: true }`** — include soft-deleted (archived) entities; excluded by default.

```graphql
query {
  runtime {
    energyCommunityCustomer(
      fieldFilter: [{ attributePath: "contact.lastName", operator: EQUALS, comparisonValue: "Doe" }]
      sortOrder: [{ attributePath: "contact.firstName", sortOrder: DESCENDING }]
    ) {
      items { rtId contact { firstName lastName } }
    }
  }
}
```

### Association navigation

Two ways to traverse relationships, on both typed and generic entities:

1. **Typed navigation properties** — generated per association role, nested directly in the selection set:

```graphql
query {
  runtime {
    energyCommunityCustomer(rtId: "693c4cd3464d7d9e1396cf0d") {
      items {
        rtId
        facilities {
          energyCommunityOperatingFacility { items { rtId name } }
        }
      }
    }
  }
}
```

2. **Generic `associations` field** — dynamic filtering by role, direction, and target type:

```graphql
query {
  runtime {
    energyCommunityCustomer(rtId: "693c4cd3464d7d9e1396cf0d") {
      items {
        rtId
        associations(
          roleId: "EnergyCommunity/AssociatedCustomer"
          direction: INBOUND
          ckId: "EnergyCommunity/OperatingFacility"
          includeIndirect: true
        ) {
          items { rtId ckTypeId }
        }
      }
    }
  }
}
```

`direction` is `INBOUND`, `OUTBOUND`, or `ANY`. `includeIndirect: true` follows transitive (multi-hop) associations of the same role. The generic `associations` field also exposes a `definitions` sub-connection returning the raw association records (`targetRtId`, `targetCkTypeId`, `originRtId`, `originCkTypeId`, `ckAssociationRoleId`) instead of the related entities themselves.

## Mutations

### Create

```graphql
mutation {
  runtime {
    energyCommunityCustomers {
      create(
        entities: [{
          contact: { firstName: "John", lastName: "Doe" }
        }]
      ) { rtId }
    }
  }
}
```

`entities` accepts an array — batch-create multiple entities in one round trip. The generic equivalent sets `ckTypeId` and an `attributes: [{attributeName, value}]` array instead of typed fields:

```graphql
mutation {
  runtime {
    runtimeEntities {
      create(entities: [{
        ckTypeId: "Basic/TreeNode"
        attributes: [{ attributeName: "name", value: "My Node" }]
      }]) { rtId }
    }
  }
}
```

**Creating with associations** — typed API passes the target directly on the association field; generic API nests it under `associations`:

```graphql
# Typed
customer: [{ modOption: CREATE, target: { rtId: "693c...", ckTypeId: "EnergyCommunity/Customer" } }]

# Generic
associations: { roleName: "parent", targets: [{ modOption: CREATE, target: { rtId: "65dc...", ckTypeId: "Basic/TreeNode" } }] }
```

Note the field name difference: query-side association filters use `roleId` (e.g. `"System/ParentChild"`), while the create/update `AssociationInput` uses `roleName`. Both identify the same CK association role.

### Update

```graphql
mutation {
  runtime {
    industryEnergyEnergyMeters {
      update(entities: [{ rtId: "662532d5241639b42933057e", item: { voltage: 235, state: ON } }]) {
        rtId voltage state
      }
    }
  }
}
```

Every update input requires `rtId`; only the fields present in `item` are changed — **except Record and RecordArray attributes, which are replaced wholesale**. To change one field inside a Record, query the current value, merge in the client, and send the full record back. Set a Record, RecordArray, or Binary attribute to `null` to clear it.

### Delete

```graphql
mutation {
  runtime {
    runtimeEntities {
      delete(
        options: ARCHIVE
        entities: [{ rtId: "662532d5241639b42933057e", ckTypeId: "Industry.Energy/EnergyMeter" }]
      )
    }
  }
}
```

Delete always goes through the generic `runtimeEntities.delete` mutation (there is no typed delete). Each entity is identified by an `RtEntityId`: `{ rtId, ckTypeId }` — and `ckTypeId` here is the **unversioned** form (`Model/TypeName`, e.g. `Industry.Energy/EnergyMeter`), never the version-suffixed internal id (`Industry.Energy-1.0.0/EnergyMeter-1`) that shows up in CK introspection responses. `options` is `ARCHIVE` (default — soft delete, excluded from normal queries, recoverable via `includeArchivedEntities: true`) or `ERASE` (hard delete, irreversible). A single call can batch entities of different types.

## Exporting the schema for client codegen

The endpoint supports the standard GraphQL introspection query (`__schema` / `__type`), so any standard GraphQL tool works against it unmodified: point `graphql-codegen`, Apollo's `apollo client:codegen` / `rover`, `get-graphql-schema`, or an equivalent framework codegen plugin at the tenant endpoint with the bearer token attached, and generate SDL or typed clients from there. Because the schema is tenant- and CK-model-specific, regenerate whenever the tenant's CK model changes (new type, new attribute, version bump).

```bash
# Example: pull raw SDL via a generic introspection-to-SDL tool
get-graphql-schema https://{asset-repo-host}/tenants/{tenantId}/graphql \
  -h "Authorization: Bearer {token}" > schema.graphql
```

## Wire conventions

| CK attribute type | GraphQL input | GraphQL output | Notes |
|---|---|---|---|
| String / Int32 / Int64 / Float / Double / Boolean | native scalar | native scalar | `Double` maps to GraphQL `Decimal` |
| DateTime / DateTimeOffset | ISO-8601 string | ISO-8601 string | e.g. `"2024-01-15T00:00:00Z"` |
| TimeSpan | decimal number | decimal number | Always **seconds** — 15 minutes = `900`, 1 hour = `3600` |
| Binary | `[Int]` (bytes 0–255) | `[Int]` (bytes 0–255) | Inline, for small payloads (< 16 MB) |
| BinaryLinked | GraphQL multipart upload, or Base64 string / byte array | `{ binaryId, filename, contentType, size, downloadUri }` | Download via REST: `GET /{tenantId}/v1/largeBinaries?largeBinaryId={binaryId}` (or just follow `downloadUri`) |
| Enum | see below | see below | |
| Record / RecordArray | nested input object / array | nested type / array | Update replaces the whole value, no partial merge |
| GeospatialPoint | `PointInput` | `RtGeospatialValue` (`{ point, distance }` when used with `geoNearFilter`) | |

**Enums** behave differently depending on which API is used:

- **Typed API** (`runtime.<typeName>`): enum fields are real GraphQL enums — write and read the member **name** (e.g. `status: ACTIVE`).
- **Generic API** (`runtimeEntities`): enum attribute values are `SimpleScalar`. Writing accepts either the member **name** (`"ACTIVE"`) or its **integer key**. Reading returns the **integer key** by default; pass `resolveEnumValuesToNames: true` on the `attributes` connection to get the name back instead.

```graphql
# Generic read, default: { "attributeName": "customerStatus", "value": 1 }
# Generic read, resolveEnumValuesToNames: true: { "attributeName": "customerStatus", "value": "ACTIVE" }
```

**Null semantics**: attributes not marked required may be `null`; a query field for an unset attribute simply returns `null` rather than being absent. `rtDisplayName` is non-null in the schema (it falls back to `<ckTypeId>@<rtId>` when the type has no display-name rule or all referenced attributes are empty); `rtDisplayDescription` stays nullable. Filtering/sorting on `rtDisplayName`/`rtDisplayDescription` operates on the stored value, not the fallback.

**Errors** come back GraphQL-style in the top-level `errors` array (`data` is `null` for a failed operation), each with a `message`, a `path`, and an `extensions.code` (e.g. `ENTITY_NOT_FOUND`, `VALIDATION_ERROR`, `CK_TYPE_NOT_FOUND`) worth branching on client-side rather than pattern-matching the message text.

## Further reading

- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/intro>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/runtime>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/retreive/retreive>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/retreive/associations>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/retreive/searchFilter>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/create/create>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/update/update>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/delete/delete>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/constructionKit/retreive/attributes>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/constructionKit/retreive/enums>
- <https://docs.meshmakers.cloud/docs/developerGuide/solutionArchitecture/apiIntegration>
