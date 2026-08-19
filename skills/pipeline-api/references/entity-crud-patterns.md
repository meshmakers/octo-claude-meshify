# Entity CRUD Patterns

Reading and writing runtime entities from pipelines: the query nodes, the update-info/apply-changes write path, ForEach accumulation, and the wire conventions (enums, dates, arrays) that cost real debugging time when unknown.

## Reading entities

| Node | Use for |
|---|---|
| `GetRtEntitiesByType@1` | Query by CK type with filters, paging (`skip`/`take`), `sortOrders` — <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/extract/getRtEntitiesByType_1> |
| `GetRtEntitiesById@1` | Fetch specific entities by `rtIds` / `rtIdsPath` — <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/extract/getRtEntitiesById_1> |
| `GetRtEntitiesByWellKnownName@1` | Look up by well-known name |
| `GetOrCreateRtEntitiesByType@1` | Resolve an existing entity's rtId by filters, or generate a fresh rtId — the upsert building block — <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/extract/getOrCreateRtEntitiesByType_1> |

```yaml
- type: GetRtEntitiesByType@1
  description: Look up one task by id
  ckTypeId: TaskBoard/Task          # unversioned in pipeline YAML
  take: 500
  targetPath: $.lookup
  fieldFilters:
    - attributePath: RtId
      operator: Equals
      comparisonValuePath: $.body.id
```

### Result shape

The result is an object, **not** a bare array:

```json
{ "TotalCount": 2,
  "Items": [ { "RtId": "…", "CkTypeId": "…", "RtWellKnownName": null,
               "Attributes": { "Title": "…", "Done": false } } ] }
```

- Iterate `$.lookup.Items`, never `$.lookup`.
- Guard with `$.lookup.TotalCount` (`If@1` + `operator: Equal/GreaterThan`, `valueType: Int`).
- Attributes read back under `Attributes.<Name>` using the attribute **name declared in the CK model**; system properties `RtId`, `CkTypeId`, `RtWellKnownName` are PascalCase.

### Field filters

```yaml
fieldFilters:
  - attributePath: Done             # attribute name as declared in the CK model,
    operator: Equals                # or a system property: RtId, RtWellKnownName, CkTypeId
    comparisonValue: true           # literal…
  - attributePath: DateKey
    operator: Equals
    comparisonValuePath: $.body.dateKey   # …or dynamic from the context
```

Operators include `Equals`, `NotEquals`, `LessThan`, `LessEqualThan`, `GreaterThan`, `GreaterEqualThan`, `In`, `NotIn`, `Like`, `MatchRegEx`, `AnyEq` — multiple filters combine with AND. `attributePath` is case-sensitive; use the exact declared attribute name.

### GetOrCreateRtEntitiesByType@1 — resolve-or-generate an rtId

```yaml
- type: GetOrCreateRtEntitiesByType@1
  ckTypeId: TaskBoard/Board
  fieldFilters:
    - attributePath: RtWellKnownName
      operator: Equals
      comparisonValue: "Main Board"
  rtIdTargetPath: $.boardRtId       # existing or freshly generated id
  modOperationPath: $.boardModOp    # Insert (new) or Update (found)
```

Nothing is persisted yet — the node only resolves ids. Feed `rtIdTargetPath`/`modOperationPath` into the write nodes below (`rtIdPath` / `updateKindPath`). This is the intended way to obtain ids that associations can reference.

## Writing entities: CreateUpdateInfo → ApplyChanges

Writes are two-phase: build **update-info objects** into an array, then persist the array in one batch.

### CreateUpdateInfo@1

Node page: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/transformation/createUpdateInfo_1>

```yaml
- type: CreateUpdateInfo@1
  description: Insert the new task
  updateKind: Insert                # Insert | Update | Delete
  ckTypeId: TaskBoard/Task
  rtIdPath: $.newRtId               # or rtId: <literal>; Update/Delete need an existing id
  generateRtId: true                # Insert only: generate an id and write it to rtIdPath
  targetPath: $.updates             # accumulate…
  targetValueWriteMode: Append      # …as array items
  targetValueKind: Array
  attributeUpdates:
    - attributeName: Title          # EXACT attribute name from the CK model definition
      attributeValueType: String
      valuePath: $.body.title       # dynamic — or `value:` for a literal
    - attributeName: Done
      attributeValueType: Boolean
      value: false
```

- `attributeValueType` values include `String`, `Int`, `Int64`, `Double`, `Boolean`, `DateTime`, `StringArray`, `Enum`, `TimeSpan`.
- Delete needs only `updateKind: Delete`, the id, and `attributeUpdates: []`.
- `generateRtId: true` is fine when nothing else references the new entity. When associations must point at it, resolve the id first with `GetOrCreateRtEntitiesByType@1` instead, so the same id is available to the association node.
- After `ApplyChanges`, the generated id is readable at `$.updates[0].RtId` — the standard way to return the new id in an HTTP response.

### CreateAssociationUpdate@1

Builds association create/delete operations. **Property names are `origin*`/`target*`** (verify with `GetPipelineSchema` — the adapter schema is authoritative):

```yaml
- type: CreateAssociationUpdate@1
  targetPath: $.assocUpdates
  targetValueWriteMode: Append
  targetValueKind: Array
  updateKind: CREATE                        # CREATE | DELETE
  originRtIdPath: $.childRtId               # the "from" entity
  originCkTypeId: TaskBoard/Task
  targetRtIdPath: $.boardRtId               # the "to" entity
  targetCkTypeId: TaskBoard/Board
  associationRoleId: System/ParentChild
```

Some CK types declare **mandatory** associations (multiplicity ONE — commonly `System/ParentChild`). Inserting such a type without the association fails at apply time with `Inbound association 'X' has minimum multiplicity of 'One'` — batch the association update together with the entity insert, guarded to run only when the mod-operation is Insert.

### ApplyChanges@1 vs ApplyChanges@2

| Node | Input | Notes |
|---|---|---|
| `ApplyChanges@1` | `path:` → array of entity update-infos | Entities only |
| `ApplyChanges@2` | `entityUpdatesPath:` + optional `associationUpdatesPath:` | Entities **and** associations in one batch — prefer this |

```yaml
- type: ApplyChanges@2
  entityUpdatesPath: $.updates
  associationUpdatesPath: $.assocUpdates    # omit when there are none
```

Docs: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/load/applyChanges_2>. Warning: `ApplyChanges@2` **silently no-ops** when the path holds nothing — a wrong `entityUpdatesPath` produces "success" with no writes. If nothing persists, check this first.

## ForEach@1 — iteration and accumulation

Node page: <https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/control/foreach_1>. `ForEach@1` runs its children in a **child context** per array element: the current item at `keyPath` (default `$.key`), a copy of the parent context at `$.full`.

**Absolute target paths inside ForEach do not reach the root context** — writing to `$.updates` (or even `$.full.updates`) inside the loop is lost when the iteration ends. Collect per-item results through the merge mechanism instead: whatever sits at `mergePath` in the child context after each iteration is gathered into an array at the parent's `targetPath`.

```yaml
# Build one Delete per done task, collected into $.dels
- type: GetRtEntitiesByType@1
  ckTypeId: TaskBoard/Task
  take: 500
  targetPath: $.lookup
  fieldFilters:
    - attributePath: Done
      operator: Equals
      comparisonValue: true

- type: ForEach@1
  iterationPath: $.lookup.Items
  keyPath: $.key                # current item
  mergePath: $.del              # child subtree to collect…
  targetPath: $.dels            # …into this parent array
  transformations:
    - type: CreateUpdateInfo@1
      updateKind: Delete
      ckTypeId: TaskBoard/Task
      rtIdPath: $.key.RtId
      targetPath: $.del         # child-context path — matches mergePath
      attributeUpdates: []

- type: If@1                    # ApplyChanges on an empty batch would no-op anyway; guard for clarity
  path: $.lookup.TotalCount
  operator: GreaterThan
  value: 0
  valueType: Int
  transformations:
    - type: ApplyChanges@2
      entityUpdatesPath: $.dels
```

The same mechanism projects entities into API responses: `mergePath: $.key.out`, children write `$.key.out.<field>`, and `targetPath: $.tasks` receives the array of projected objects. `maxDegreeOfParallelism: 1` forces sequential iteration when order matters.

## Wire conventions and pitfalls

### Enums: write names, read integer keys

- **Writing** (`attributeValueType: Enum`): supply the enum **member name** (`value: Monthly`, or a name via `valuePath`). Numeric keys are not recognized on the write path.
- **Reading**: enum attributes come back as **integer keys** (`0`, `1`, `2` …), not names. API clients must map both directions.

### Dates are normalized on the way in

The adapter's JSON body parse converts date-looking strings (e.g. `2026-08-19`) to DateTime values — a date sent as a plain string can come back re-formatted (full ISO timestamp). Either normalize on read in the client, or store dates as `String` attributes with values that do not round-trip through the date parser.

### Writing an empty array nulls the attribute

A `StringArray`/`IntArray` attribute updated with `[]` becomes **null**, not an empty array. If "empty set" must be representable, use a sentinel value (e.g. `["~"]`) written when the set is empty, and filter it on read — or model the count separately.

### Unset attributes

Prefer writing all attributes with explicit defaults (`""`, `0`, `false`) on insert. Entities created elsewhere (e.g. by hand in Refinery Studio) with absent optional attributes can break read pipelines that assume the paths exist.

## Math@1 — server-side arithmetic

There is no expression language in the write path — numeric logic is composed from `Math@1` steps (<https://docs.meshmakers.cloud/docs/technologyGuide/communication/dataFlows/nodes/transformation/math_1>). `path` selects the object(s) holding the numbers; `itemPath`/`itemTargetPath` are relative to each object:

```yaml
# Keep working values in one container object, e.g. $.calc
- type: Math@1
  description: newScore = score + delta
  path: $.calc
  itemPath: $.score
  operation: Add                # Add | Subtract | Multiply | Divide | Modulo | Round
  valuePath: $.calc.delta       # or value: <constant>
  itemTargetPath: $.score
```

Chain steps for compound formulas (e.g. halve an even share: Modulo 2 → Subtract remainder → Divide 2). Combine with `If@1` comparisons (`LessThan`, `GreaterThan` …) for clamping and branching.
