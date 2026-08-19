# CK YAML Authoring

A Construction Kit (CK) model is a folder of YAML files that describes types, their attributes, enums, records, and the associations between types. The Construction Kit Compiler (`octo-ckc`) validates the folder against JSON schemas and compiles it into a single file that `octo-cli` imports into a tenant.

## Folder layout

```
ConstructionKit/
├── ckModel.yaml           # Model id, description, dependencies
├── types/                 # Entity type definitions (one or more per file)
├── attributes/            # Reusable attribute definitions
├── enums/                 # Enumeration definitions
├── associations/          # Association role definitions
├── records/                # Embedded-document (struct) definitions
└── migrations/             # Optional — see versioning-and-migrations.md
```

Each folder can hold any number of `.yaml` files; the compiler reads every file in the folder regardless of name, and how you split definitions across files is a matter of taste. Every element file (everything except `ckModel.yaml`) starts with:

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
```

Create a scaffold folder with the compiler:

```bash
octo-ckc -c New -p './ConstructionKit'
```

## ckModel.yaml

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-meta.schema.json
modelId: TaskTracker-1.0.0
description: Minimal example model — a board that contains tracked items.
dependencies:
- System-[2.0,3.0)
```

| Field | Required | Description |
|---|---|---|
| `modelId` | Yes | `<ModelName>-<version>`, e.g. `TaskTracker-1.0.0`. `ModelName` is your model's namespace — every type/attribute/enum/record/association id in this model lives under it. |
| `description` | No | Free text. |
| `dependencies` | No | List of version **ranges** on other CK models, e.g. `System-[2.0,3.0)`. Every model needs at least a dependency on `System`, which provides the base `Entity` type and other foundational elements. |

Dependency ranges use interval notation: `[2.0,3.0)` means "≥ 2.0.0 and < 3.0.0". At compile time `octo-ckc` resolves each range against the configured catalogs and freezes the **highest matching published version** into the compiled model as an exact pin — see `catalogs-and-publishing.md` for what catalogs are searched.

## Attributes

An attribute is a named, typed value slot with a system-wide unique id. Define it once, reference it from any number of types or records.

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
attributes:
- id: ItemTitle
  description: Short title of the tracked item
  valueType: String

- id: ItemStatus
  description: Current status of the item
  valueType: Enum
  valueCkEnumId: ${this}/ItemStatus

- id: ItemSortOrder
  description: Manual sort position within the board
  valueType: Int
  metaData:
    - key: Unit
      value: "position"
```

| Field | Required | Description |
|---|---|---|
| `id` | Yes | Pascal-case identifier, unique within the model. |
| `valueType` | Yes | One of: `String`, `Boolean`, `DateTime`, `DateTimeOffset`, `Int`, `Int64`, `Double`, `TimeSpan`, `StringArray`, `IntArray`, `Enum`, `Record`, `RecordArray`, `Binary`, `BinaryLinked`, `GeospatialPoint`. |
| `valueCkEnumId` | For `Enum` | Reference to an enum definition, e.g. `${this}/ItemStatus`. |
| `valueCkRecordId` | For `Record`/`RecordArray` | Reference to a record definition. |
| `defaultValues` | No | Array of literal default values. |
| `isRuntimeState` | No, default `false` | Marks the attribute as owned by the running system rather than by seed/blueprint data — on a blueprint re-apply the tenant's current value is preserved instead of being overwritten. Use it for status/health/sequence-number style attributes. |
| `metaData` | No | Array of `{key, value, description}` triples for arbitrary semantic annotations (units, external semantic ids, etc.). |
| `description` | No | Free text. |

An attribute is defined once and then **referenced by id** wherever it's used (on a type, a record, or an association role) — see the pattern in Types below. Attributes cannot be assigned twice to the same type with different names; define `GrossPrice` and `NetPrice` as two separate attributes rather than reusing one `Price` attribute.

## Enums

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
enums:
- enumId: ItemStatus
  useFlags: false
  values:
  - key: 0
    name: Open
    description: Not started yet
  - key: 1
    name: InProgress
    description: Currently being worked on
  - key: 2
    name: Done
    description: Completed
```

| Field | Required | Description |
|---|---|---|
| `enumId` | Yes | Pascal-case identifier. |
| `useFlags` | No, default `false` | Treat the enum as a bit-flag set. |
| `isExtensible` | No, default `false` | Allow adding values at runtime via the `constructionKit.enums(...).updateValueExtensions` GraphQL mutation, without a model version bump. Intended for slowly-changing master data; a full tenant reload is needed to apply extension changes. |
| `values[].key` | Yes | Unique integer within the enum. |
| `values[].name` | Yes | Pascal-case, no whitespace/special characters. |
| `values[].description` | No | Free text. |

An attribute of `valueType: Enum` points at an enum via `valueCkEnumId`; a pipeline or the GraphQL API can read/write the value as either the integer key or the string name.

## Associations

An association role describes a named, bidirectional relationship type: how many entities you reach walking it one way (`outboundMultiplicity`) versus the other way (`inboundMultiplicity`), and what each direction is called for navigation.

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
associationRoles:
- id: Contains
  description: Links an item to the board that contains it
  inboundName: Board
  outboundName: Items
  inboundMultiplicity: One
  outboundMultiplicity: N
```

| Field | Required | Description |
|---|---|---|
| `id` | Yes | Pascal-case identifier for the role. |
| `inboundName` / `outboundName` | Yes | Pascal-case navigation property names used from each side. |
| `inboundMultiplicity` / `outboundMultiplicity` | Yes | One of `ZeroOrOne`, `One`, `N`. |
| `attributes` | No | Attributes carried on the association itself (not on either endpoint). |

`outboundMultiplicity: N` on the role above means: navigating outbound from a `Board` reaches `N` items (`outboundName: Items`). `inboundMultiplicity: One` means: navigating inbound from an `Item` reaches exactly `One` board (`inboundName: Board`) — i.e. one board has many items, and each item belongs to exactly one board.

Use the platform's built-in `System/ParentChild` role for plain hierarchical (parent/child) relationships instead of defining a custom role, unless the relationship needs its own semantic name.

## Records

A record is an embedded structured value (a "struct") — attributes bundled together without their own runtime identity, reusable as a `Record`/`RecordArray`-typed attribute value.

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
records:
- recordId: ProbeAddress
  attributes:
    - id: ${thisModel}/ProbeStreet
      name: Street
    - id: ${thisModel}/ProbeCity
      name: City
      isOptional: true
```

`recordId`, optionally `derivedFromCkRecordId` for record inheritance, and an `attributes` list with the same `id`/`name`/`isOptional` shape used on types (see below).

## Types

A type combines attributes and associations into an entity with runtime identity (instances of a type are called Runtime Entities, `RtEntity`, and identified by an `rtId`).

```yaml
$schema: https://schemas.meshmakers.cloud/construction-kit-elements.schema.json
types:
- typeId: Item
  description: A tracked item that belongs to exactly one board.
  derivedFromCkTypeId: ${System}/Entity-1
  displayNameRule: "${ItemTitle}"
  attributes:
  - id: ${this}/ItemTitle-1
    name: ItemTitle
  - id: ${this}/ItemStatus-1
    name: ItemStatus
  - id: ${this}/ItemSortOrder-1
    name: ItemSortOrder
    isOptional: true
  associations:
  - id: ${this}/Contains-1
    targetCkTypeId: ${this}/Board-1
```

| Field | Required | Description |
|---|---|---|
| `typeId` | Yes | Pascal-case identifier, may use `.` for namespacing (e.g. `Photovoltaic.Module`). |
| `derivedFromCkTypeId` | No (in practice, always for domain entities) | Base type to inherit from. Every domain type ultimately derives from `${System}/Entity` (directly or via an intermediate base type). |
| `isAbstract` / `isFinal` | No | Standard inheritance controls. |
| `displayNameRule` / `displayDescriptionRule` | No | Rule computing the read-only `rtDisplayName` / `rtDisplayDescription` fields on every save, e.g. `"${Name ?? SerialNumber}"`. Supports `${attributePath}` interpolation and `??` fallback; inherited by derived types (nearest non-empty rule wins). See the dedicated doc linked below for full syntax. |
| `attributes` | No | List of attribute *usages*: `id` (reference to an attribute definition), `name` (Pascal-case field name on this type — can differ from the attribute's own `id`), `isOptional`, `autoIncrementReference`. |
| `associations` | No | List of association *usages*: `id` (reference to an association role), `targetCkTypeId` (the type on the other end), `targetAttributes`. |
| `indexes` | No | Explicit MongoDB index declarations (`indexType`: `None`/`Ascending`/`Text`/`Unique`/`UniqueNotDeleted`). |

### Reference tokens

IDs that point at another element (`derivedFromCkTypeId`, an attribute `id` inside a type's `attributes` list, `valueCkEnumId`, an association `id`, `targetCkTypeId`, …) are written as `<model>/<elementName>`. Two placeholders avoid hard-coding the model name:

- `${this}` and `${thisModel}` are interchangeable — both resolve to the current model's own namespace. Public documentation examples tend to use `${thisModel}`; real-world models commonly use the shorter `${this}`.
- `${System}` resolves to the `System` base library that every model depends on.

A trailing `-<N>` (e.g. `-1`) is an explicit **element version** suffix. It is optional in source YAML — the compiler infers version `1` when omitted — but real models commonly write it explicitly (`${System}/Entity-1`, `${this}/ItemStatus-1`) to be unambiguous, and it is always present in the *compiled* output regardless of whether the source YAML included it.

## Ck vs. Rt naming — don't prefix with "Ck"

The framework itself draws the `Ck`/`Rt` distinction: **Ck** = schema/type definitions (what you author here — `CkType`, `CkAttribute`, `CkEnum`, …), **Rt** = runtime data instances of those types (`RtEntity`, with `rtId`, `rtDisplayName`, …). Do not prefix your own `typeId`s with `Ck` (e.g. write `Item`, not `CkItem`) — the framework's tooling and generated GraphQL fields already carry that distinction, and a `Ck`-prefixed type name reads as redundant everywhere it appears.

## Naming pitfalls

- Follow PascalCase for model, type, attribute, and enum names; singular nouns for type names (`Item`, not `Items`); a prefix for closely related types/enums in the same family (`ItemStatus`, not a bare `Status`) so the name carries its own context once imported alongside other models.
- Avoid type names that collide with platform-level concepts the API layer already gives meaning to. `Pipeline`, for instance, is reserved by the platform's own ETL pipeline concept (`System.Communication/Pipeline`) — naming a domain type `Pipeline` produces confusing, colliding vocabulary once both are visible through the same GraphQL schema and tooling. When in doubt, check the type search across already-loaded models before finalizing a name.
- Attributes are unique-in-system by design: reuse an existing attribute (e.g. the base library's `Name`) instead of redefining an equivalent one under a new id, so the same real-world concept isn't represented by two different attribute ids across models.

## GraphQL surfaces automatically

Nothing about publishing a GraphQL API is a separate step. The moment a CK model is imported into a tenant, every type, record, enum, and association in it becomes typed GraphQL fields under `runtime` — a type `TaskTracker/Item` becomes the camelCase query field `runtime.taskTrackerItem` (model namespace folded into the type name) plus create/update mutations, in addition to the generic, type-agnostic `runtime.runtimeEntities` API. Re-importing a model after a version bump updates the schema with it; no manual GraphQL schema work is needed.

See the mesh-data-layer skill for the full GraphQL query/mutation reference.

## See also

- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/intro>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/design>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/createConstructionKitLibrary>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/reference/ckEnum>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/reference/displayNameRules>
- <https://docs.meshmakers.cloud/docs/technologyGuide/constructionKits/reference/ckAutoIncrement>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/retreive/retreive>
- <https://docs.meshmakers.cloud/docs/technologyGuide/dataAccess/runtime/create/create>
