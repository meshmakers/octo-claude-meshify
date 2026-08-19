# CLAUDE.md — octo-claude-meshify

## What This Is

A Claude Code plugin for **external** OctoMesh application developers: building apps on OctoMesh or migrating existing apps onto it ("meshifying"). Five skills: `meshify` (hub + migration playbook), `ck-modeling`, `pipeline-api`, `blueprint-packaging`, `mesh-data-layer`.

## Audience Contract (critical)

This plugin ships to people **outside Meshmakers**. Content must never reference:

- Internal environments or hosts (test-2, staging, prod clusters, `*.mm.cloud`, internal registries)
- Internal build tooling (octo-tools profile, `Invoke-BuildAll`, `DebugL`, CI agent pools, pipeline templates)
- Azure DevOps work items (`AB#…`) or internal repos as file references
- Absolute local paths

Allowed: platform-defined constants (well-known rtIds seeded by `System.Communication`, service ports, CK type ids) and clearly-marked *local development cluster* examples. The single documentation authority is <https://docs.meshmakers.cloud>.

## Design Principles

- **Look up, don't bake in.** Pipeline nodes evolve; skills document the *lookup protocol* (URL patterns on docs.meshmakers.cloud, see `skills/meshify/references/docs-site-map.md`) rather than cataloging node properties. Node details are fetched live when needed.
- **Assumed toolchain**: a running OctoMesh environment plus `octo-cli`, `octo-ckc`, `octo-bpm`, `docker`, `helm`. Local OctoMesh setup is out of scope.
- **Two target architectures** (both first-class): full pipeline backend, and mesh-as-data-layer via GraphQL. Custom adapter development is out of scope.
- **Self-contained examples.** `examples/` files are distilled and neutral; they do not require access to any sample repository.

## Skill Authoring Rules

- **Frontmatter description**: prose-first — one or two sentences stating what the skill does FIRST, then a `Trigger on:` tail with keywords. Keep under **1200 characters**.
- **SKILL.md body**: under **500 lines**; imperative form, no second person. Operational overview only — details live in `references/`.
- **Accuracy**: every command, flag, YAML key, and URL must be verified against docs.meshmakers.cloud (or a real, working example) before documenting. Never document from memory.
- **Progressive disclosure**: SKILL.md → `references/*.md` → live doc lookup via WebFetch.

## Validation

```bash
claude plugin validate . --strict
```

Also grep for leakage before releasing: `grep -rniE 'test-2|mm\.cloud|DebugL|Invoke-Build|AB#[0-9]|/Users/' skills/`

## Versioning

`version` in `.claude-plugin/plugin.json` is the release gate — bump semver per meaningful change set, keep `marketplace.json` in sync, record releases in `CHANGELOG.md`.
