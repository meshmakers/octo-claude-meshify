---
name: ck-modeling
description: Author and evolve OctoMesh Construction Kit (CK) data models - the YAML for types, attributes, enums, associations and records, the ckModel.yaml manifest with dependencies, compiling and publishing with octo-ckc, importing into a tenant with ImportCk, the versioning rules (published versions are immutable - every change needs a bump), and CK migrations between versions. Use when designing or changing the domain model of an OctoMesh application. Trigger on - Construction Kit, CK model, CK YAML, ckModel.yaml, OctoMesh data model, model the domain on OctoMesh, CK entity type, CK type definition, CK attribute, CK enum, CK association, CK record, octo-ckc, octo-ckc Compile, ImportCk, CK catalog, publish model, model version bump, versioning rules, CK migration, migration script, ValidateVersion, LibraryStatus.
---

# CK Modeling — the OctoMesh Domain Model

## What a CK model is

A Construction Kit (CK) model is the versioned, typed schema of the domain: entity types with attributes, enums, and associations, authored as YAML in a `ConstructionKit/` folder. The platform compiles it, stores it per tenant, and generates the runtime data plane from it — including the per-tenant GraphQL API (no separate API work per type). `Ck` prefixes denote schema, `Rt` prefixes runtime instances; the framework derives both from the model — never prefix own type names with `Ck`.

## Authoring loop

```bash
octo-ckc -c New -p ./my-model                          # scaffold ConstructionKit/ + ckModel.yaml
# edit YAML: types/, attributes/, enums/, associations/, records/
octo-ckc -c Compile -p ./my-model -o ./out             # compile + validate locally
octo-cli -c ImportCk -f ./out/<modelfile>.yaml -w      # import into a dev tenant, iterate
octo-ckc -c ValidateVersion -p ./my-model              # gate: is the version bump sufficient?
octo-ckc -c Publish -f ./out/<modelfile>.yaml --catalog <catalog>   # default: LocalFileSystemCatalog
```

Exact flags, folder layout, and the full YAML syntax for every artifact kind: `references/ck-yaml-authoring.md`. The example model in `examples/task-tracker-ck/` compiles cleanly as shipped — copy it as a starting skeleton.

## Rules that prevent expensive mistakes

- **Published versions are immutable.** Re-importing the same version with changed content is a no-op — every change requires a version bump. Which changes force major vs minor: `references/versioning-and-migrations.md`.
- **Name types for the domain**, and avoid names that collide with platform concepts (e.g. `Pipeline` collides with DataFlow pipelines in tooling and conversation).
- **Model dependencies explicitly** in `ckModel.yaml` (e.g. `System-[2.0,3.0)`); use `${this}`/`${thisModel}` placeholders for intra-model references.
- **Migrations are code.** When runtime data must survive a breaking version change, ship migration scripts (`migrations/migration-meta.yaml` + `{from}-to-{to}.yaml`) with the model.

## Additional resources

### Reference files

- **`references/ck-yaml-authoring.md`** — folder layout, ckModel.yaml, full YAML syntax per artifact kind, placeholder tokens, naming rules, GraphQL surfacing
- **`references/versioning-and-migrations.md`** — SemVer immutability, the ValidateVersion change table, migration-script schema (preconditions, actions, transforms, post-validations), tenant upgrade order
- **`references/catalogs-and-publishing.md`** — catalog sources, the compile→import→validate→publish loop, tenant-side lifecycle commands (LibraryStatus, ImportFromCatalog, FixAll)

### Example

- **`examples/task-tracker-ck/`** — a minimal complete model (Board/Item types, ItemStatus enum, one-to-many Contains association) verified with a real octo-ckc compile
