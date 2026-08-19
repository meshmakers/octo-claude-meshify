# TaskTracker — minimal CK model example

A minimal, domain-neutral Construction Kit model: one enum, two types, one association between them. Every file below compiles cleanly with `octo-ckc -c Compile` against a `System` dependency in the `[2.0,3.0)` range — copy the `ConstructionKit/` folder as the starting point for your own model and rename freely.

```
ConstructionKit/
├── ckModel.yaml
├── attributes/
│   ├── board.yaml
│   └── item.yaml
├── enums/
│   └── itemStatus.yaml
├── associations/
│   └── contains.yaml
└── types/
    ├── board.yaml
    └── item.yaml
```

Model shape: a `Board` (`Name`) contains many `Item`s (`Title`, `Status: ItemStatus`, optional `SortOrder`), linked by a `Contains` association role — one board, many items, each item belonging to exactly one board.

## Compile and import

```bash
octo-ckc -c Compile -p './ConstructionKit' -o './out'
octo-cli -c ImportCk -f './out/ck-tasktracker.yaml' -w
```

See `../../references/ck-yaml-authoring.md` for the syntax reference this example follows, and `../../references/catalogs-and-publishing.md` for the full compile → import → publish loop.
